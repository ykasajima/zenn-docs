---
title: "[yocto] v4l2loopbackビルド方法"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["yocto", "imx"]
published: true
---
# 背景
- imx yocto linux で GStreamer で [v4l2sink](https://gstreamer.freedesktop.org/documentation/video4linux2/v4l2sink.html) を使おうとするとエラーになる。
- 最新のカーネルにオーバーレイが入っているらしいが見つからなかった。
- ググるとカーネルモジュールの [umlaeute/v4l2loopback](https://github.com/umlaeute/v4l2loopback) でやるそうだ。
- imx レイヤーになかったので自前ビルドする。

# Yocto レシピ
Yoctoレシピは [v4l2loopback on Yocto](https://stackoverflow.com/questions/63075479/v4l2loopback-on-yocto) を参考に作成。

```v4l2loopback.bb
SUMMARY = "V4L2Loopback"
DESCRIPTION = "v4l2loopback module"
LICENSE = "GPLv2"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0;md5=801f80980d171dd6425610833a22dbe6"

SRC_URI = "git://github.com/umlaeute/v4l2loopback.git;protocol=https;branch=main"
S = "${WORKDIR}/git"

inherit module

export KERNEL_DIR="${KERNEL_SRC}"
MODULES_INSTALL_TARGET = "install"
RPROVIDES_${PN} += "kernel-module-v4l2loopback"
```

# ビルドエラー対処

## asm/tlbbatch.h: No such file or directory
asm/tlbbatch.h は x86 のものなので imx であるわけがない。

Makefile を見るとカーネルソースディレクトリが x86 になっているので、Yocto の [KERNEL_SRC](https://docs.yoctoproject.org/singleindex.html#term-KERNEL_SRC) に変える。

レシピに `export KERNEL_DIR="${KERNEL_SRC}"` を追加する。

## No rule to make target 'modules_install'
[v4l2loopback on Yocto](https://stackoverflow.com/questions/63075479/v4l2loopback-on-yocto) に書いてあるが、Makefile に `modules_install` ターゲットが存在しない。
> デフォルト値は [module.bbclass](http://cgit.openembedded.org/openembedded-core/tree/meta/classes/module.bbclass?h=rocko) で `MODULES_INSTALL_TARGET ?= "modules_install"` になっている。

レシピに `MODULES_INSTALL_TARGET = "install"` を追加する。

# 動作確認
v4l2loopback.ko を imx に持っていって以下を確認。

## モジュール読み込み
`insmod ./v4l2loopback.ko`

`lsmod` で存在確認

## v4l2sink を試す
insmod すると `/dev/video4` が追加されたので、これに videotestsrc を書き込む。で、video4 を RTSP 配信する。VLC で RTSP アクセスして表示できれば OK。

```
gst-launch-1.0 videotestsrc ! v4l2sink device=/dev/video4
gst-variable-rtsp-server -u "v4l2src device=/dev/video4 ! vpuenc_vp8 ! rtpvp8pay name=pay0"
```

# 参考
- [Incorporating Out-of-Tree Module](https://docs.yoctoproject.org/4.0.4/singleindex.html#incorporating-out-of-tree-modules)
- [Compile a Custom Kernel Module](https://developer.toradex.com/linux-bsp/how-to/build-yocto/custom-meta-layers-recipes-and-images-in-yocto-project-hello-world-examples/#compile-a-custom-kernel-module)
- [YoctoのSDKでOut-of-treeのカーネルモジュールをビルドする](https://mickey-happygolucky.hatenablog.com/entry/2020/12/15/015724)
- [Howto build a kernel module out of the kernel tree](https://wiki.koansoftware.com/index.php/Howto_build_a_kernel_module_out_of_the_kernel_tree)
- [Incorporating Out-of-Tree Modules in YOCTO](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/Incorporating-Out-of-Tree-Modules-in-YOCTO/ta-p/1373825)
