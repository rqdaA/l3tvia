+++
title = "XNUのpage fault handlerを読む"
description = "coins advent calendar 11日目の記事"
date = 2025-12-11T15:00:00Z
draft = false

[taxonomies]
tags = ["xnu"]
[extra]
toc = true
+++

この記事は[coins advent calendar](https://adventar.org/calendars/11747)の11日目の記事です。

10日目はベアさんの[会社に100万円のPCが落ちていたのでwindows消してLocalLLM構築してみた](https://qiita.com/bear_wash/items/c6fc08bc8714df9d1a44)でした。RTX 5000 Adaを使ってLLMを動かすのはロマンがありますね。

11日目はゃーさんの記事です。

---

以下では、xnuのversionは`xnu-11417.140.69`を前提とします。

このversionは`MacOS Sequoia 15.6`で使われています。ソースコードのダウンロードは以下から行ってください。

- [https://opensource.apple.com/releases/](https://opensource.apple.com/releases/)
- [https://github.com/apple-oss-distributions/xnu/tree/xnu-11417.140.69](https://github.com/apple-oss-distributions/xnu/tree/xnu-11417.140.69)

# Fault Handler

fault handlerは、CPUが例外を起こしたときに呼ばれるOS側の処理ルーチンのことです。
このうち、page fault handlerとは、仮想アドレスは割り当てられているが、物理ページが存在しないアドレスにアクセスしたときに投げられる例外(page fault)を処理するルーチンのことです。

例えば、以下のようなコードを実行したときにpage faultが発生し、page fault handlerが呼び出されます。

```c
#include <sys/mman.h>
#include <unistd.h>

void main() {
    char* ptr = mmap(0, getpagesize(), PROT_READ | PROT_WRITE, MAP_ANON | MAP_PRIVATE, -1, 0);
    ptr[0] = 'A';
}
```

この例では、`mmap`を利用して1page分の仮想アドレスを新たに割り当てて、そのpageに`'A'`という文字列を書き込んでいます。
mmapを呼び出した時点では、仮想アドレスを割り当てただけで、まだ実際の物理ページ(=メモリ)は割り当てられていません。その後、`'A'`を書き込んだとき、すなわちpage faultが発生したときに、初めて物理メモリをこのアドレスに割り当てます。

この記事では、xnuがどのようにpage faultを処理しているかを見ていきます。

## IDTのセットアップ

ユーザーランドでpage faultが発生すると、IDTを経由してkernel-landへ入ります。

IDTとは、(大雑把に言うと)ある例外が発生したときどの関数がその処理を行うかが書かれているものです。\
詳しくは[https://wiki.osdev.org/Interrupt_Descriptor_Table]([https://wiki.osdev.org/Interrupt_Descriptor_Table])を参照してください。

xnuのソースコードでは`osfmk/i386/i386_init.c`で初期化が行われています。

```c
__attribute__((noreturn))
void
vstart(vm_offset_t boot_args_start)
{
    // snip
        cpu_desc_init(cpu_datap(0));
        cpu_desc_load(cpu_datap(0));
    // snip
}

extern unsigned mldtsz;
void
cpu_desc_init(cpu_data_t *cdp)
{
    cpu_desc_index_t        *cdi = &cdp->cpu_desc_index;

    if (cdp == cpu_data_master) {
        // snip
        cdi->cdi_idtu.ptr  = (void *)DBLMAP((uintptr_t) &master_idt64);
        cdi->cdi_idtb.ptr  = (void *)((uintptr_t) &master_idt64);
        // snip
}


void
cpu_desc_load(cpu_data_t *cdp)
{
    cpu_desc_index_t        *cdi = &cdp->cpu_desc_index;

    // snip
    cdi->cdi_idtb.size = 0x1000 + cdp->cpu_number;
    cdi->cdi_idtu.size = cdi->cdi_idtb.size;
    // snip
    lidt((uintptr_t *) &cdi->cdi_idtu);
    // snip
}

static inline void
lidt(uintptr_t *desc)
{
    __asm__ volatile ("lidt %0" : : "m" (*desc));
}
```

`vstart`が`cpu_desc_init`を呼び出して、`master_idt64`を`cdi->cdi_idtu`に入れます。その後、`cpu_desc_load`が`cdi->cdi_idtu`を引数に`lidt`を呼び出してIDTの初期化を行っています。

`master_idt64`の定義は以下です。

```c
struct fake_descriptor64 master_idt64[IDTSZ]
__attribute__ ((section("__HIB,__desc")))
__attribute__ ((aligned(PAGE_SIZE))) = {
#include "../x86_64/idt_table.h"
};
```

`idt_table.h`の内容は以下です。
```c
TRAP(0x00, idt64_zero_div)
TRAP_IST1(0x01, idt64_debug)
TRAP_IST2(0x02, idt64_nmi)
USER_TRAP(0x03, idt64_int3)
USER_TRAP(0x04, idt64_into)
USER_TRAP(0x05, idt64_bounds)
TRAP(0x06, idt64_invop)
TRAP(0x07, idt64_nofpu)
TRAP_IST1(0x08, idt64_double_fault)
TRAP(0x09, idt64_fpu_over)
TRAP_ERR(0x0a, idt64_inv_tss)
TRAP_IST1(0x0b, idt64_segnp)
TRAP_IST1(0x0c, idt64_stack_fault)
TRAP_IST1(0x0d, idt64_gen_prot)
TRAP_SPC(0x0e, idt64_page_fault)
// snip
```

実際に、IDT Registerの内容を読んでみると最初のentryに`idt64_zero_div`が入っていることがわかります。
(IDTから参照されているアドレスが`idt64_zero_div`そのものでないのは、`double map`というセキュリティ機構によるもの)

```
IDT=     fffff69bc000c000

(lldb) x/8hx fffff69bc000c000
0xfffff69bc000c000: 0x2560 0x0008 0x8e00 0xc000 0xf69b 0xffff 0x0000 0x0000

(lldb) x/4i fffff69bc0002560
    0xfffff69bc0002560: push   0x0
    0xfffff69bc0002562: push   0x1
    0xfffff69bc0002564: push   0x0
    0xfffff69bc0002566: jmp    0xfffff69bc00037cf

(lldb) disas -n idt64_zero_div
kernel`idt64_zero_div:
    0xffffff800dd02560 <+0>:  push   0x0
    0xffffff800dd02562 <+2>:  push   0x1
    0xffffff800dd02564 <+4>:  push   0x0
    0xffffff800dd02566 <+6>:  jmp    0xffffff800dd037cf ; hi64_sysenter + 31
```


## user-landからkernel-landへ

ユーザーランドでpage fualtが発生するとIDT経由で`idt64_page_fault`が呼び出されます。

```asm
Entry(idt64_page_fault)
	pushq	$(HNDL_ALLTRAPS)
#if !(DEVELOPMENT || DEBUG)
	pushq	$(T_PAGE_FAULT)
	jmp	L_dispatch
```

`L_dispatch`はgsなどの設定をしたあとに、`ks_dispatch`を呼びます。

```asm
Entry(ks_dispatch)
	popq	%rax
	cmpw	$(KERNEL64_CS), ISF64_CS(%rsp)
	je	EXT(ks_dispatch_kernel)

	mov 	%rax, %gs:CPU_UBER_TMP
	mov 	%gs:CPU_UBER_ISF, %rax
	add 	$(ISF64_SIZE), %rax

	xchg	%rsp, %rax
/* Memory to memory moves (aint x86 wonderful):
 * Transfer the exception frame from the per-CPU exception stack to the
 * 'PCB' stack programmed at cswitch.
 */
	push	ISF64_SS(%rax)
	push	ISF64_RSP(%rax)
	push	ISF64_RFLAGS(%rax)
	push	ISF64_CS(%rax)
	push	ISF64_RIP(%rax)
	push	ISF64_ERR(%rax)
	push	ISF64_TRAPFN(%rax)
	push 	ISF64_TRAPNO(%rax)
	mov	%gs:CPU_UBER_TMP, %rax
	jmp	EXT(ks_dispatch_user)
```

`ks_dispatch`はtrapnoなどをstackに積み、`ks_dispatch_user`を呼びます。

```asm
Entry(ks_dispatch_user)
	cmpl	$(TASK_MAP_32BIT), %gs:CPU_TASK_MAP
	je	L_dispatch_U32		/* 32-bit user task */

L_dispatch_U64:
	subq	$(ISS64_OFFSET), %rsp
	mov	%r15, R64_R15(%rsp)
	mov	%rsp, %r15
	mov	%gs:CPU_KERNEL_STACK, %rsp
	jmp	L_dispatch_64bit
```

`L_dispatch_64bit`が呼ばれます。
```asm
/*
 * Here for 64-bit user task or kernel
 */
L_dispatch_64bit:
	movl	$(SS_64), SS_FLAVOR(%r15)

	/*
	 * Save segment regs if a 64-bit task has
	 * installed customized segments in the LDT
	 */
	cmpl	$0, %gs:CPU_CURTASK_HAS_LDT
	je	L_skip_save_extra_segregs

	mov	%ds, R64_DS(%r15)
	mov	%es, R64_ES(%r15)

L_skip_save_extra_segregs:
	mov	%fs, R64_FS(%r15)
	mov	%gs, R64_GS(%r15)


	/* Save general-purpose registers */
	mov	%rax, R64_RAX(%r15)
	mov	%rbx, R64_RBX(%r15)
	mov	%rcx, R64_RCX(%r15)
	mov	%rdx, R64_RDX(%r15)
	mov	%rbp, R64_RBP(%r15)
	mov	%rdi, R64_RDI(%r15)
	mov	%rsi, R64_RSI(%r15)
	mov	%r8,  R64_R8(%r15)
	mov	%r9,  R64_R9(%r15)
	mov	%r10, R64_R10(%r15)
	mov	%r11, R64_R11(%r15)
	mov	%r12, R64_R12(%r15)
	mov	%r13, R64_R13(%r15)
	mov	%r14, R64_R14(%r15)

	/* Zero unused GPRs. BX/DX/SI are clobbered elsewhere across the exception handler, and are skipped. */
	xor	%ecx, %ecx
	xor	%edi, %edi
	xor	%r8, %r8
	xor	%r9, %r9
	xor	%r10, %r10
	xor	%r11, %r11
	xor	%r12, %r12
	xor	%r13, %r13
	xor	%r14, %r14

	/* cr2 is significant only for page-faults */
	xor	%rax, %rax
	cmpl	$T_PAGE_FAULT, R64_TRAPNO(%r15)
	jne	1f
	mov	%cr2, %rax
1:
	mov	%rax, R64_CR2(%r15)

L_dispatch_U64_after_fault:
	mov	R64_TRAPNO(%r15), %ebx	/* %ebx := trapno for later */
	mov	R64_TRAPFN(%r15), %rdx	/* %rdx := trapfn for later */
	mov	R64_CS(%r15), %esi	/* %esi := cs for later */

	jmp	L_common_dispatch
```
ここで、user-landのレジスタを退避します。その後`rdx`にtrapnoを入れて`L_common_dispatch`を呼びます。

```asm
L_common_dispatch:
	/* snip */
66:
	leaq	EXT(idt64_hndl_table1)(%rip), %rax
	jmp	*(%rax, %rdx, 8)
```

TLB周りの処理をしたあと、`idt64_hndl_table1[trapno]`を呼び出します。なお、`idt64_hndl_table1`の定義は以下です。

```asm
EXT(idt64_hndl_table1):
	.quad	EXT(hndl_allintrs)
	.quad	EXT(hndl_alltraps)
	.quad	EXT(hndl_sysenter)
	.quad	EXT(hndl_syscall)
	.quad	EXT(hndl_unix_scall)
	.quad	EXT(hndl_mach_scall)
	.quad	EXT(hndl_mdep_scall)
	.quad	EXT(hndl_double_fault)
	.quad	EXT(hndl_machine_check)
.text
```

`idt64_page_fault`で`$(HNDL_ALLTRAPS)`をpushしていることからもわかる通り、page
faultの時は`hndl_all_traps`が呼ばれます。

```asm
Entry(hndl_alltraps)
	mov	%esi, %eax
	testb	$3, %al
	jz	trap_from_kernel

	TIME_TRAP_UENTRY

	/* Check for active vtimers in the current task */
	mov	%gs:CPU_ACTIVE_THREAD, %rcx
	movl	$-1, TH_IOTIER_OVERRIDE(%rcx)	/* Reset IO tier override to -1 before handling trap/exception */
	mov	TH_TASK(%rcx), %rbx
	TASK_VTIMER_CHECK(%rbx, %rcx)

	CCALL1(user_trap, %r15)			/* call user trap routine */
```

`hndl_alltraps`は`user_trap`を呼び出します。

## user_trap

```c
void
user_trap(
	x86_saved_state_t *saved_state)
{
	// snip
	if (is_saved_state64(saved_state)) {
		x86_saved_state64_t     *regs;

		regs = saved_state64(saved_state);

		/* Record cpu where state was captured */
		regs->isf.cpu = current_cpu;

		type = regs->isf.trapno;
		err  = (int)regs->isf.err & 0xffff;
		vaddr = (user_addr_t)regs->cr2;
		rip   = (user_addr_t)regs->isf.rip;
    // snip
    }
    // snip
	switch (type) {
    	case T_PAGE_FAULT:
	{
		prot = VM_PROT_READ;

		if (err & T_PF_WRITE) {
			prot |= VM_PROT_WRITE;
		}
		if (__improbable(err & T_PF_EXECUTE)) {
			prot |= VM_PROT_EXECUTE;
		}
		kret = vm_fault(thread->map,
		    vaddr,
		    prot, FALSE, VM_KERN_MEMORY_NONE,
		    THREAD_ABORTSAFE, NULL, 0);
		if (__probable((kret == KERN_SUCCESS) || (kret == KERN_ABORTED))) {
			break;
		} else if (__improbable(kret == KERN_FAILURE)) {
			/*
			 * For a user trap, vm_fault() should never return KERN_FAILURE.
			 * If it does, we're leaking preemption disables somewhere in the kernel.
			 */
			panic("vm_fault() KERN_FAILURE from user fault on thread %p", thread);
		}

		/* PAL debug hook (empty on x86) */
		pal_dbg_page_fault(thread, vaddr, kret);
		exc = EXC_BAD_ACCESS;
		code = kret;
		subcode = vaddr;
	}
	break;
```
`user_trap`はstackに積まれた`saved_state`をみて、例外の種類でswitchを行います。

page faultの場合は、`T_PAGE_FAULT`です。その後、引数の設定を行って`vm_fault`を呼びます。

```c
kern_return_t
vm_fault(
	vm_map_t        map,
	vm_map_offset_t vaddr,
	vm_prot_t       fault_type,
	boolean_t       change_wiring,
	vm_tag_t        wire_tag,               /* if wiring must pass tag != VM_KERN_MEMORY_NONE */
	int             interruptible,
	pmap_t          caller_pmap,
	vm_map_offset_t caller_pmap_addr)
{
	struct vm_object_fault_info fault_info = {
		.interruptible = interruptible,
		.fi_change_wiring = change_wiring,
	};

	return vm_fault_internal(map, vaddr, fault_type, wire_tag,
	           caller_pmap, caller_pmap_addr,
	           NULL, &fault_info);
}
```
`vm_fault_internal`が呼ばれます。ここでpage fault handlerのメインの処理が行われます(2400行あります)。

## vm_fault_internal

xnuのメモリ管理については、[@i41nbeer氏のこのスライド](https://papers.put.as/papers/macosx/2019/OBTS_v2_Beer.pdf)が詳しいです。

正確性を犠牲にして雑にいうならば、`vm_map_t`が仮想アドレス空間全体、`vm_map_entry_t`が連続した仮想アドレス空間、`pmap_t`がページテーブルを表します。

```c
kern_return_t
vm_fault_internal(
	vm_map_t           map,
	vm_map_offset_t    vaddr,
	vm_prot_t          caller_prot,
	vm_tag_t           wire_tag,               /* if wiring must pass tag != VM_KERN_MEMORY_NONE */
	pmap_t             caller_pmap,
	vm_map_offset_t    caller_pmap_addr,
	ppnum_t            *physpage_p,
	vm_object_fault_info_t fault_info)
{
```

関数定義はこのようになっています。

```c
	kr = vm_map_lookup_and_lock_object(&map, vaddr,
	    (fault_type | (need_copy ? VM_PROT_COPY : 0)),
	    object_lock_type, &version,
	    &object, &offset, &prot, &wired,
	    fault_info,
	    &real_map,
	    &object_is_contended);
```
様々なassertをしたあと、`vm_map_lookup_and_lock_object`が呼ばれます。
```c
kern_return_t
vm_map_lookup_and_lock_object(
	vm_map_t                *var_map,       /* IN/OUT */
	vm_map_offset_t         vaddr,
	vm_prot_t               fault_type,
	int                     object_lock_type,
	vm_map_version_t        *out_version,   /* OUT */
	vm_object_t             *object,        /* OUT */
	vm_object_offset_t      *offset,        /* OUT */
	vm_prot_t               *out_prot,      /* OUT */
	boolean_t               *wired,         /* OUT */
	vm_object_fault_info_t  fault_info,     /* OUT */
	vm_map_t                *real_map,      /* OUT */
	bool                    *contended)     /* OUT */
```

この関数は`var_map`の`vaddr`に一致するアドレス空間の様々なデータを返します。

長くなるので省略しますが、最終的にこの関数は`vm_map_store_lookup_entry_rb`を呼び出して、`var_map`が管理するRB
treeから`vm_map_entry_t`を取得してきて、そこから種々の情報を取得します。

```c
bool
vm_map_store_lookup_entry_rb(vm_map_t map, vm_map_offset_t address, vm_map_entry_t *vm_entry)
{
	struct vm_map_header *hdr = &map->hdr;
	struct vm_map_store  *rb_entry = RB_ROOT(&hdr->rb_head_store);
	vm_map_entry_t       cur = vm_map_to_entry(map);
	vm_map_entry_t       prev = VM_MAP_ENTRY_NULL;

	while (rb_entry != (struct vm_map_store*)NULL) {
		cur =  VME_FOR_STORE(rb_entry);
		if (address >= cur->vme_start) {
			if (address < cur->vme_end) {
				*vm_entry = cur;
				return TRUE;
			}
			rb_entry = RB_RIGHT(rb_entry, entry);
			prev = cur;
		} else {
			rb_entry = RB_LEFT(rb_entry, entry);
		}
	}
	if (prev == VM_MAP_ENTRY_NULL) {
		prev = vm_map_to_entry(map);
	}
	*vm_entry = prev;
	return FALSE;
}
```

その後、`vm_fault_page`を呼び出して、pageをinsertします。
```c
	vm_fault_return_t err = vm_fault_page(object, offset, fault_type,
	    (fault_info->fi_change_wiring && !wired),
	    FALSE,                /* page not looked up */
	    &prot, &result_page, &top_page,
	    &type_of_fault,
	    &error_code, map->no_zero_fill,
	    fault_info);
```

```c
			m = vm_page_grab_options(grab_options);
			if (m == VM_PAGE_NULL) {
				vm_fault_cleanup(object, first_m);
				thread_interrupt_level(interruptible_state);

				return VM_FAULT_MEMORY_SHORTAGE;
			}

			if (fault_info && fault_info->batch_pmap_op == TRUE) {
				vm_page_insert_internal(m, object,
				    vm_object_trunc_page(offset),
				    VM_KERN_MEMORY_NONE, FALSE, TRUE, TRUE, FALSE, NULL);
			} else {
				vm_page_insert(m, object, vm_object_trunc_page(offset));
			}

```

この後、様々な後処理をして`hndl_alltraps`に戻ります(長いので割愛)。

## kernel-landからuser-landへ

`hndl_alltraps`に戻ってきたあと、`return_from_traps`へ進みます。

```asm
Entry(return_from_trap)
	movq	%gs:CPU_ACTIVE_THREAD,%r15	/* Get current thread */
	movl	$-1, TH_IOTIER_OVERRIDE(%r15)	/* Reset IO tier override to -1 before returning to userspace */
	movq	TH_PCB_ISS(%r15), %r15		/* PCB stack */
	movl	%gs:CPU_PENDING_AST,%eax
	testl	%eax,%eax
	je	EXT(return_to_user)		/* branch if no AST */
```

```asm
Entry(ret_to_user)
	mov	%gs:CPU_ACTIVE_THREAD, %rdx
	cmpq	$0, TH_PCB_IDS(%rdx)	/* Is there a debug register context? */
	jnz	L_dr_restore_island
L_post_dr_restore:
	/*
	 * We now mark the task's address space as active for TLB coherency.
	 * Handle special cases such as pagezero-less tasks here.
	 */
	mov	%gs:CPU_TASK_CR3, %rcx
	mov	%rcx, %gs:CPU_ACTIVE_CR3
	cmpl	$0, %gs:CPU_PAGEZERO_MAPPED
	jnz	L_cr3_switch_island
	movl	EXT(no_shared_cr3)(%rip), %eax
	test	%eax, %eax		/* -no_shared_cr3 */
jnz	L_cr3_switch_island
    L_cr3_switch_return:
	mov	%gs:CPU_DR7, %rax	/* Is there a debug control register?*/
	cmp	$0, %rax
	je	4f
	mov	%rax, %dr7		/* Set DR7 */
	movq	$0, %gs:CPU_DR7
4:
	cmpl	$(SS_64), SS_FLAVOR(%r15)	/* 64-bit state? */
	jne	L_32bit_return

	/*
	 * Restore general 64-bit registers.
	 * Here on fault stack and PCB address in R15.
	 */
	leaq	EXT(idt64_hndl_table0)(%rip), %rax
	jmp	*8(%rax)

```

`ks_64bit_return`が呼ばれます。

```asm
Entry(ks_64bit_return)

	mov	R64_R14(%r15), %r14
	mov	R64_R13(%r15), %r13
	mov	R64_R12(%r15), %r12
	mov	R64_R11(%r15), %r11
	mov	R64_R10(%r15), %r10
	mov	R64_R9(%r15),  %r9
	mov	R64_R8(%r15),  %r8
	mov	R64_RSI(%r15), %rsi
	mov	R64_RDI(%r15), %rdi
	mov	R64_RBP(%r15), %rbp
	mov	R64_RDX(%r15), %rdx
	mov	R64_RCX(%r15), %rcx
	mov	R64_RBX(%r15), %rbx
	mov	R64_RAX(%r15), %rax
	/* Switch to per-CPU exception stack */
	mov	%gs:CPU_ESTACK, %rsp

	/* Synthesize interrupt stack frame from PCB savearea to exception stack */
	push	R64_SS(%r15)
	push	R64_RSP(%r15)
	push	R64_RFLAGS(%r15)
	push	R64_CS(%r15)
	push	R64_RIP(%r15)

	cmpw	$(KERNEL64_CS), 8(%rsp)
	jne	1f			/* Returning to user (%r15 will be restored after the segment checks) */
	mov	R64_R15(%r15), %r15
	jmp	L_64b_kernel_return	/* Returning to kernel */

L_64b_kernel_return:
.globl EXT(ret64_iret)
EXT(ret64_iret):
        iretq			/* return from interrupt */
```

`iret`が呼ばれて、無事user-landへ帰還できます。


# おわりに

あまりメモリ管理の本質的なところに全然触れられなかった...。
