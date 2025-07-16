+++
title = "kmallocを読む"
description = "Internals of kmalloc"
date = 2025-05-24T19:00:00Z
draft = true

[taxonomies]
tags = ["Kernel Exploit", "Linux Internal"]
[extra]
toc = true
series = "Linux Internal"
+++

注: この記事は体系的にkmallocの動きをまとめた記事ではない。
あくまで筆者が疑問を解消することを目的に読み勧めた時のメモ。

# 目標

- `struct kmem_cache`がどのようにfreelistを管理しているのか
- bata24/gefのslab-dumpが使えない時に同等のことをする方法

- double free検知の手法
- kmem_cache_freeではalloc_pageしたpageがfreeできないのはなぜ？
- kfreeにbuddyで管理されているpageを渡した時の挙動

# バージョン

手元にあったのがcosなのでcosを読む

- repo: https://cos.googlesource.com/third_party/kernel
  - branch: release-R109-cos-6.1
  - commit: `5e17cd9189aa1`
  - config: https://storage.googleapis.com/kernelctf-build/releases/cos-109-17800.436.99/.config

# kmalloc

`CONFIG_SLUB`なので必要のないところは削ってある

```c
static __always_inline __alloc_size(1) void *kmalloc(size_t size, gfp_t flags)
{
	if (__builtin_constant_p(size)) {
		if (size > KMALLOC_MAX_CACHE_SIZE)
			return kmalloc_large(size, flags);
		index = kmalloc_index(size);

		if (!index)
			return ZERO_SIZE_PTR;

		return kmalloc_trace(
				kmalloc_caches[kmalloc_type(flags)][index],
				flags, size);
	}
	return __kmalloc(size, flags);
}
```

## slab_alloc_node

最終的にkmallocはここに到達する。
ここでは、`lru=NULL`かつ`node=NUMA_NO_NODE`とする

```c
static __always_inline void *slab_alloc_node(struct kmem_cache *s, struct list_lru *lru,
		gfp_t gfpflags, int node, unsigned long addr, size_t orig_size)
{
```

`s`は`struct kmem_cache`、`c`は`struct kmem_cache_cpu`。

`c->freelist`と`c->slab`が存在するならば、`c->freelist`を返す(fast
path)。
そうでなければ、`__slab_alloc`を呼ぶ(slow path)。

```c
	object = c->freelist;
	slab = c->slab;

	if (unlikely(!object || !slab)) {
		object = __slab_alloc(s, gfpflags, node, addr, c, orig_size);
	} else {
        ...
    }
```

`c`の例としてはこんな感じ。

```c
gef> p *(struct kmem_cache_cpu*)0xffff88813bc37160
$1 = {
  freelist = 0xffff8881009e3000,
  tid = 0x22800,
  slab = 0xffffea0004027800,
  partial = 0xffffea000400fa00,
  lock = {<No data fields>}
}
```

## \_\_\_slab_alloc

`slab->frozen`は`lock`みたいなもの。
`c->slab`が`active page`を指している。
`c->partial`は`partial list`。

`c->slab->freelist`が存在すればそれを返す。
そうでなければ、gotoで以下の2つのpathに振り分けられる。

### new_slab

`c->partial`を`c->slab`に入れる。
再度実行。

### new_objects

`get_partial`で、`s->node[node]->partial`から`c->slab`にロード。


# kfree
`((struct folio *)virt_to_page(object))->`

```c
void kfree(const void *object)
{
	struct folio *folio;
	struct slab *slab;
	struct kmem_cache *s;

	trace_kfree(_RET_IP_, object);

	if (unlikely(ZERO_OR_NULL_PTR(object)))
		return;

	folio = virt_to_folio(object);
	if (unlikely(!folio_test_slab(folio))) {
		free_large_kmalloc(folio, (void *)object);
		return;
	}

	slab = folio_slab(folio);
	s = slab->slab_cache;
	__kmem_cache_free(s, (void *)object, _RET_IP_);
}
```


## slab_free

