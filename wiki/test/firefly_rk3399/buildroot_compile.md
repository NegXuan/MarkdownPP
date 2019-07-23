# 编译 Buildroot 固件

本章介绍 Buildroot 固件的编译使用。

## 准备工作

### 下载源码

下载repo工具：

```bash
mkdir linux
cd linux
git clone https://github.com/FireflyTeam/repo.git
```

下载 Linux-SDK：

* 方法一

```bash
mkdir linux-sdk
cd linux-sdk

# 初始化repo仓库
../repo/repo init --repo-url https://github.com/FireflyTeam/repo.git -u https://github.com/FireflyTeam/manifests.git -b linux-sdk -m rk3399/rk3399_linux_release.xml

# 同步源码
../repo/repo sync -c
```

* 方法二

下载[Linux_SDK.7z](http://www.t-firefly.com/doc/download/page/id/54.html#other_186)

```bash
# 把压缩包放在上一步下载repo工具的目录下
7z x Linux_SDK.7z

# 更新代码
repo sync -c
```

### 配置编译环境

安装编译所需工具，确保工具都正确安装：

```bash
sudo apt-get install repo git-core gitk git-gui gcc-arm-linux-gnueabihf u-boot-tools device-tree-compiler \
gcc-aarch64-linux-gnu mtools parted libudev-dev libusb-1.0-0-dev python-linaro-image-tools \
linaro-image-tools autoconf autotools-dev libsigsegv2 m4 intltool libdrm-dev curl sed make \
binutils build-essential gcc g++ bash patch gzip bzip2 perl tar cpio python unzip rsync file bc wget \
libncurses5 libqt4-dev libglib2.0-dev libgtk2.0-dev libglade2-dev cvs git mercurial rsync openssh-client \
subversion asciidoc w3m dblatex graphviz python-matplotlib libc6:i386 libssl-dev texinfo \
liblz4-tool genext2fs lib32stdc++6
```

## 编译SDK

### 配置编译文件

选择开发板对应的配置文件。配置文件会链接到 `device/rockchip/.BoardConfig.mk`，查看该文件可确认当前所使用的配置文件：

```bash
./build.sh firefly-rk3399.mk

# 文件路径在 `device/rockchip/rk3399/firefly-rk3399.mk`
```

`.mk` 文件默认配置为编译 Buildroot 固件，下面对 Buildroot 相关配置进行说明：

```bash
# Buildroot config
export RK_CFG_BUILDROOT=rockchip_rk3399     # Buildroot 根文件系统配置文件

# 文件路径在 `buildroot/configs/rockchip_rk3399_defconfig`
```

```bash
# Recovery config
export RK_CFG_RECOVERY=rockchip_rk3399_recovery     # recovery 模式下根文件系统配置文件（可省略）

# 文件路径在 `buildroot/configs/rockchip_rk3399_recovery_defconfig`
```

```bash
# rootfs image path
export RK_ROOTFS_IMG=buildroot/output/$RK_CFG_BUILDROOT/images/rootfs.$RK_ROOTFS_TYPE   # Buildroot 根文件系统镜像路径

# 本例中，文件路径在 `buildroot/output/rockchip_rk3399/images/rootfs.ext4`
# 注：该文件路径将在首次编译根文件系统后生成
```

执行编译命令时，将会根据 `.mk` 文件进行编译。

### 全自动编译

全自动编译会编译并打包固件 `update.img`，生成固件目录 `rockdev/`：

```bash
./build.sh
```

### 部分编译

* 编译 kernel

```bash
./build.sh kernel
```

* 编译 u-boot

```bash
./build.sh uboot
```

* 编译 rootfs

编译 Buildroot 根文件系统，将会在 `buildroot/output` 生成编译输出目录：

```bash
./build.sh buildroot

# 注：确保作为普通用户编译 Buildroot 根文件系统，避免不必要的错误。编译过程中会自动下载所需软件包，请保持联网状态
```

## 固件打包

### 更新链接

为确保 `rockdev/` 目录下文件链接正确，更新各部分镜像链接：

```bash
./mkfirmware.sh
```

### 打包固件

将 `rockdev` 目录的各部分镜像打包成固件 `update.img`：

```bash
./build.sh updateimg
```

## Buildroot 介绍

### output 目录

Buildroot 编译输出结果保存在 `output` 目录，具体目录由配置文件决定，本例保存在 `buildroot/output/rockchip_rk3399` 目录，后续可以在该目录执行 `make` 编译根文件系统。

采用全自动编译方式时，默认会生成 `buildroot/output/rockchip_rk3399_recovery` 目录，这是 `recovery` 的编译输出目录。

子目录说明：

* `build/` 包含所有的源文件，包括 Buildroot 所需主机工具和选择的包，这个目录包含所有
模块源码。
* `host/` 主机端编译需要的工具包括交叉编译工具。
* `images/` 包含压缩好的根文件系统镜像文件。
* `staging/` 这个目录类似根文件系统的目录结构，包含编译生成的所有头文件和库，以及其他开发文件，不过他们没有裁剪，比较庞大，不适用于目标文件系统。
* `target/` 包含完整的根文件系统，对比 `staging/`，它没有开发文件，不包含头文件，二进制文件也经过 `strip` 处理。

### 自定义 Buildroot

下文将介绍一些自定义 Buildroot 的方法。

#### 模块配置

默认编译好的根文件系统不一定满足我们的需求，我们可能需要增加一些第三方包，或者修改包的配置选项，Buildroot 支持图形化方式去做选择配置：

```bash
cd buildroot/output/rockchip_rk3399/

# 进入图形化配置界面，选择所需模块，保存退出
make menuconfig

# 保存到配置文件 'buildroot/configs/rockchip_rk3399_defconfig'
make savedefconfig

#编译 Buildroot 根文件系统
make
```

需要了解的是：

* 进行编译时，Buildroot 根据配置，会自动从网络获取相关的软件包，包括一些第三方库，插件，实用工具等，放在dl/目录。
* 软件包会解压在 `output/build/` 目录下，然后进行编译。
* 如果要修改软件包的源码，可以通过打补丁的方式进行修改，补丁集中放在 `package/` 目录，Buildroot 会在解压软件包时为其打上相应的补丁。

#### busybox 配置修改

busybox 用于管理系统的命令工具，可按如下方式修改：

```bash
cd buildroot/output/rockchip_rk3399/

# 进入图形化配置界面，选择所需工具，退出保存
make busybox-menuconfig

# 保存到配置文件 `board/rockchip/common/base/busybox.config`
make busybox-update-config

make
```

#### 文件系统覆盖

文件系统覆盖是指在目标文件系统编译完成后将文件覆盖到文件系统目录。通过这种方式，我们可以简单的添加或修改一些文件：

* 本例覆盖目录 `buildroot/board/rockchip/rk3399/fs-overlay-64`
* 公有覆盖目录 `buildroot/board/rockchip/common`

例：`buildroot/board/rockchip/rk3399/fs-overlay-64/etc/input-event-daemon.conf` 将覆盖文件系统的 `/etc/input-event-daemon.conf` 文件。

### Buildroot 官网

更加详细具体的开发技巧可到 Buildroot 官网学习。

[Buildroot 官网](https://buildroot.org/)

[Buildroot 开发手册](https://buildroot.org/downloads/manual/manual.html)