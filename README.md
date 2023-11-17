# rust-for-linux

<!--toc:start-->
- [rust-for-linux](#rust-for-linux)
  - [Env](#env)
  - [编译内核(Exercise 1)](#编译内核exercise-1)
  - [编译Rust模块helloworld(Exercise 2)](#编译rust模块helloworldexercise-2)
    - [安装QEMU](#安装qemu)
    - [利用busybox制作文件系统](#利用busybox制作文件系统)
  - [e1000网卡驱动(Exercise 3)](#e1000网卡驱动exercise-3)
    - [Preferences](#preferences)
    - [先学习编译fujita e1000，了解busybox加载模块的流程](#先学习编译fujita-e1000了解busybox加载模块的流程)
      - [编译rust_e1000.ko](#编译ruste1000ko)
      - [Linux kernel网络参数设置](#linux-kernel网络参数设置)
      - [QEMU启动参数](#qemu启动参数)
    - [作业3 e1000-driver 添加代码](#作业3-e1000-driver-添加代码)
<!--toc:end-->

## Env

- Debian GNU/Linux 12 (bookworm) aarch64 in Parallel Desktop Virtual Machine
- QEMU qemu-system-aarch64 version 7.2.5
- rust-for-linux arm64 6.6.0-rc4
- BusyBox v1.35.0 (Debian 1:1.35.0-4+b3) multi-call binary.
<img width="663" alt="图片" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/19506571-5a38-4373-89fa-e64ded9aa1d8">

## 编译内核(Exercise 1)

使用fujita的linux分支

```bash
git clone https://github.com/fujita/linux -b rust-dev --depth=1
```

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise1.md

按照指导安装依赖，
- 但是注意cargo install bindgen，没有后面的cli，因为fujiya使用的是比较早的linux kernel版本

```bash
make ARCH=arm64 LLVM=1 O=build defconfig

make ARCH=arm64 LLVM=1 O=build menuconfig
#set the following config to yes
General setup
        ---> [*] Rust support

make ARCH=arm64 LLVM=1 -j8
```

<img width="611" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/dc694004-6721-4866-baa9-88de45eeb71c">

## 编译Rust模块helloworld(Exercise 2)

### 安装QEMU

```bash
sudo apt install qemu-user qemu-system-arm # 安装qemu 7.x.x
```

> qemu 8.x.x以上运行DQIB kernel会报错 => "...net user not compiled into kernel..."

### 利用busybox制作文件系统

- [Debian Quick Image Baker(DQIB)](https://people.debian.org/~gio/dqib/)
另一个选择，但是不是很会用

```bash
make menuconfig ARCH=arm64

Settings  --->
        --- Build Options
        [*] Build static binary (no shared libs)

make -j20
make install
```

`cat qemu-init.sh`

```bash
#!/bin/sh
busybox echo "[INFO] Init from a minimal initrd!"
busybox echo "============================"
busybox poweroff -f
```

`cat qemu-initramfs.desc`

```bash
dir     /bin                                                                            0755 0 0
file    /bin/busybox            busybox                                                 0755 0 0
slink   /bin/sh                 /bin/busybox                                            0755 0 0
file    /init                   qemu-init.sh                                            0755 0 0
```

```bash
# 制作img
# 每次修改完qemu-init.sh & qemu-initramfs.desc要记得重新制作
../linux-fujita/build/usr/gen_init_cpio qemu-initramfs.desc > qemu-initramfs.img      
```

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise2.md

按照指导添加代码

```bash
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <M>Print Helloworld in Rust (NEW) # 其它也要勾选
```
`cat qemu-init.sh`

```bash
#!/bin/sh
busybox echo "[INFO] Init from a minimal initrd!"
busybox echo "============================"
busybox insmod rust_helloworld.ko

busybox poweroff -f
```

`cat qemu-initramfs.desc`

```bash
dir     /bin                                                                            0755 0 0
file    /bin/busybox            busybox                                                 0755 0 0
slink   /bin/sh                 /bin/busybox                                            0755 0 0
file    /init                   qemu-init.sh                                            0755 0 0
file    /rust_helloworld.ko    ../linux-fujita/build/samples/rust/rust_helloworld.ko    0755 0 0
```

- QEMU启动参数

```bash
#!/bin/sh

qemu-system-aarch64                                                                \
        -machine 'virt'                                                            \
        -cpu 'cortex-a57'                                                          \
        -m 1G                                                                      \
        -netdev user,id=net,hostfwd=tcp::2222-:22                                  \
        -kernel /home/junzhe/Dev/Software/linux-fujita/build/arch/arm64/boot/Image \
        -initrd ../busybox/qemu-initramfs.img                                      \
        -nographic                                                                 \
        -append "root=LABEL=rootfs console=ttyAMA0"                                \
```

<img width="688" alt="图片" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/a249379e-9019-4f10-9f72-6eac9172c035">


## e1000网卡驱动(Exercise 3)

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise3.md

### Preferences

- https://github.com/fujita/rust-e1000
- https://github.com/fujita/linux/tree/rust-e1000

<img width="717" alt="图片" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/b1dff794-5735-40b4-a585-20baa53a0258">

### 先学习编译fujita e1000，了解busybox加载模块的流程

#### 编译rust_e1000.ko

```bash
git clone https://github.com/fujita/rust-e1000
make KDIR=/home/junzhe/Dev/Software/linux-fujita/build/ LLVM=1      
```

`cat qemu-initramfs.desc`

```bash
dir     /bin                                                                            0755 0 0
file    /bin/busybox            busybox                                                 0755 0 0
slink   /bin/sh                 /bin/busybox                                            0755 0 0
file    /init                   qemu-init.sh                                            0755 0 0
file    /rust_helloworld.ko    ../linux-fujita/build/samples/rust/rust_helloworld.ko    0755 0 0
file    /rust_e1000.ko         ../rust-e1000/rust_e1000.ko                              0755 0 0 # 新增
```

#### Linux kernel网络参数设置

`cat qemu-init.sh`

```bash
#!/bin/sh

busybox echo "[INFO] Init from a minimal initrd!"
busybox echo "============================"
busybox echo "[INFO] Modules Loaded"
busybox insmod rust_helloworld.ko
busybox insmod rust_e1000.ko
busybox echo "============================"
busybox echo "[INFO] Network configured"
busybox echo "IP set to 192.168.100.224"
busybox ifconfig lo 127.0.0.1 netmask 255.0.0.0 up
busybox ifconfig eth0 192.168.100.224 netmask 255.255.255.0 broadcast 192.168.100.255 up
busybox ip addr
busybox echo "============================"

export 'PS1=(kernel) >'
busybox sh

busybox poweroff -f
```

#### QEMU启动参数

```bash
qemu-system-aarch64 \
        -machine 'virt' \
        -cpu 'cortex-a57' \
        -m 1G \
        -kernel /home/junzhe/Dev/Software/linux-fujita/build/arch/arm64/boot/Image \
        -initrd ../busybox/qemu-initramfs.img \
        -netdev tap,ifname=tap0,id=tap0,script=no,downscript=no -device e1000,netdev=tap0 \
        -nographic                                                                 \
        -append "root=LABEL=rootfs console=ttyAMA0" \
```

- ping通
  
<img width="894" alt="图片" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/3fbbb2d5-7b63-4074-afe1-6128a6f8d448">


### 作业3 e1000-driver 添加代码

- https://github.com/yuoo655/e1000-driver/tree/main/src

- 修改后的代码 https://github.com/ye-junzhe/e1000-driver

- 增加自定义函数并编译成功

<img width="1363" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/a32539e3-d7eb-4f92-9a64-d5ac40cc6bca">

## e1000网卡驱动(Exercise 4)

### 停用内置e1000网卡驱动

在menuconfig中搜索e1000，并停用

### 作业4添加代码

- 修改后的代码 https://github.com/ye-junzhe/e1000-driver

编译后对e1000模块进行加载

<img width="1041" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/33e8d065-241f-418b-b523-933bd4244d7e">

<img width="916" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/b0783f4b-28db-40f3-be52-834de5f04114">

<img width="871" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/2034c6cb-b703-40e4-946e-24c15e340f31">

```bash
busybox insmod e1000_for_linux.ko
busybox ifconfig eth0 up
busybox ifconfig eth0 10.0.2.20
busybox ip route add default via 10.0.2.2 dev eth0
```

- ping通外网
<img width="874" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/22ba334c-8209-47fe-981d-02694df60561">


TODO:
- https://github.com/rcore-os/virtio-drivers
