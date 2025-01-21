+++
title = "Kernel Exploit環境構築"
description = "Kernelのビルドとソースコードリーディングの環境構築"
date = 2024-10-17T15:00:00Z
draft = false

[taxonomies]
tags = ["Setup","Kernl Exploit"]
[extra]
toc = true
series = "Setup"
+++

## 想定環境

- OS: Ubuntu 22.04

## カーネルのビルドとソースコードリーディングの環境整備

### GCCを最新にする

```sh
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-13 g++-13
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100 --slave /usr/bin/g++ g++ /usr/bin/g++-13
```

### 提供ファイルのバージョン確認

以下を実行しカーネルのバージョンを特定する

```sh
cd /path/to/ctf
strings bzImage | grep -E '[0-9]+\.[0-9]+\.[0-9]+'
```

### Linuxソースのclone

```sh
git clone git@github.com:gregkh/linux.git
cd linux
git fetch --tags origin linux-6.6.y:linux6.6.y
git checkout v6.6.56
```

### デバッグシンボルをつけてビルド

```sh
# 依存パッケージのインストール
sudo apt install libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm bear dwarves
```

#### - デフォルトでビルドする場合

```sh
make alldefconfig && make menuconfig
```

#### - configファイルが提供されている場合

```sh
cp /path/to/ctf/config .config
make olddefconfig && make menuconfig
```

---

以下のコンフィグをオンにする

- `CONFIG_8250`,`CONFIG_8250_CONSOLE`: ログを出力させるために必要
- `CONFIG_BLK_DEV_INITRD`: rootfs(initramfs)のマウントに必要
- `CONFIG_DEVTMPFS`,`CONFIG_DEVTMPFS_MOUNT`: /devの自動マウント
- `CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT`: シンボル付きvmlinuxを吐く
- `CONFIG_BINFMT_SCRIPT`,`CONFIG_BINFMT_MISC`: エクスプロイトでよく使うmodprobeを有効化

ビルド(5分くらいかかる)

```sh
bear -- make -j$(nproc) bzImage vmlinux
```

`/vmlinux`がデバッグシンボル付きのカーネルイメージで`/arch/x86/boot/bzImage`がカーネルなのでこれを問題ファイルにコピー

```sh
mv vmlinux /path/to/ctf/vmlinux.own
mv bzImage /path/to/ctf/bzImage.own
```

### ソースコードリーディング

#### - CLionの場合

1. 初期画面からopenを選択
2. `/compile_commands.json`を選択
3. Open as Projectを選択

#### - VSCodeの場合

[https://p3land.smallkirby.com/kernel/prepare](https://p3land.smallkirby.com/kernel/prepare) を読むといいらしい(VSCodeを使ってないので詳しくはわからない)

## ツール類のインストール

### lysithea

お好きな場所で

```sh
git clone git@github.com:smallkirby/lysithea.git
```

### vermagic

```sh
git@github.com:yaxinsn/vermagic.git
cd vermagic
make -j$(nproc)
```

### gef

```sh
wget -q https://raw.githubusercontent.com/bata24/gef/dev/install.sh -O- | sudo sh
```

### QEMU

```sh
sudo apt install qemu-system qemu-system-common qemu-utils
```

### パスへの追加

bashrcに以下を追加

```sh
export PATH="$PATH:/path/to/lysithea"
export PATH="$PATH:/path/to/vermagic"
```

## 問題を解く

一般に以下のファイルが提供される

- `bzImage`: カーネル
- `rootfs.cpio` or `initramfs.cpio`: rootファイルシステム
- `[name].ko`: カーネルオブジェクト
- `run.sh`: 実行するためのシェルスクリプト

追加でこれらのファイルが提供されていると嬉しい

- `.config`: ビルドコンフィグ
- `[name].c`: カーネルオブジェクトのソースコード

### ディレクトリの初期化

```sh
cd /path/to/ctf
lysithea init
lysithea extract
```

`extract`ディレクトリにrootfsの内容が展開される。まずはデバッグようにルートでシェルを得られるようにする

```sh
grep -r 'setsid cttyhack setuidgid ' extracted
```

を実行し出てきたファイルがCTF用の初期化ファイルである。そのファイルの中身を以下のように書き換える

```diff
- setsid cttyhack setuidgid 9999 sh
+ setsid cttyhack setuidgid 0 sh
...
- echo 2 > /proc/sys/kernel/kptr_restrict
+ echo 0 > /proc/sys/kernel/kptr_restrict
...
- echo 1 > /proc/sys/kernel/dmesg_restrict
+ echo 0 >/proc/sys/kernel/dmesg_restrict
```

### KASLRの無効化

`run.sh.dev`を以下のように変更

```diff
- -kernel ./bzImage
+ -kernel ./bzImage.own
...
- -append "console=ttyS0 kaslr quiet
+ -append "console=ttyS0 nokaslr verbose
...
+ -s
```

### 実行

以下のコマンドが`exploit`のコンパイルと便利ファイルの追加をした上で、`extracted`の圧縮を行い、`run.sh.dev`を実行してくれる

```sh
lysithea local
```

ここで、別タブでgdbを起動してアタッチするとカーネルの動きが追える

```sh
sudo gdb vmlinux.own

gef> target remote:1234
gef> directory /path/to/linux # ソースコードの表示
```

![image](/image/build_result.png)

## トラブルシューティング

### `Invalid module format`というエラーが出た場合

`linux/include/linux/vermagic.h`を確認して必要なビルドオプションをつけて再度ビルドする

#### 例:

```sh
strings [chal].ko | grep -E '[0-9]+\.[0-9]+\.[0-9]+'
```

を実行して、`6.6.58 SMP preempt mod_unload`が出てきたとする

- `CONFIG_SMP=y`
- `CONFIG_PREEMPT_BUILD=y`
- `CONFIG_MODULE_UNLOAD=y`
- `CONFIG_MODVERSIONS=n`
- `RANDSTRUCT_NONE=y`

上記のコンフィグを設定してカーネルを再度ビルド

## 参考文献

- [p3land (セクキャンの講義資料)](https://p3land.smallkirby.com/kernel/)
- [qemu 各種アーキ環境構築](https://hackmd.io/@bata24/ryWzOHEMw)
- [Linux Kernel Debug Enviroment with buildroot](https://hackmd.io/@t3mp/H1jTrjTp2)
