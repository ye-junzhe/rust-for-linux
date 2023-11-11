# rust-for-linux

## Env

- Debian GNU/Linux 12 (bookworm) aarch64 in Parallel Vitrual Machine
- QEMU qemu-system-aarch64 version 7.2.5
- rust-for-linux arm64 6.6.0-rc4

## 编译内核

使用fujita的linux分支

```bash
git clone https://github.com/fujita/linux -b rust-dev --depth=1
```

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise1.md

按照指导安装依赖，
- 但是注意cargo install bindgen，没有后面的cli，因为fujiya使用的是比较早的linux kernel版本
然后

```bash
make ARCH=arm64 LLVM=1 O=build defconfig

make ARCH=arm64 LLVM=1 O=build menuconfig
#set the following config to yes
General setup
        ---> [*] Rust support

make ARCH=arm64 LLVM=1 -j8
```

<img width="611" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/dc694004-6721-4866-baa9-88de45eeb71c">

## 编译Rust模块helloworld

~### QEMU启动Debian Quick Image Baker(DQIB)~(改用Busybox)
- ~https://people.debian.org/~gio/dqib/~

#### 安装QEMU

```bash
sudo apt install qemu-user qemu-system-arm # 安装qemu 7.x.x
```

> qemu 8.x.x以上运行DQIB kernel会报错 => "...net user not compiled into kernel..."

#### 利用busybox制作文件系统

```bash
make menuconfig ARCH=arm64


Settings  --->
        --- Build Options
        [*] Build static binary (no shared libs)

make -j20
```
```bash

cat qemu-init.sh

#!/bin/sh
busybox echo "init from a minimal initrd!"
busybox poweroff -f

cat qemu-initramfs.desc

dir     /bin                                                                            0755 0 0
file    /bin/busybox            busybox                                                 0755 0 0
slink   /bin/sh                 /bin/busybox                                            0755 0 0
file    /init                   qemu-init.sh                                            0755 0 0

../linux-fujita/build/usr/gen_init_cpio qemu-initramfs.desc > qemu-initramfs.img      
```

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

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise2.md

按照指导添加代码

```bash
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <M>Print Helloworld in Rust (NEW) # 其它也要勾选

修改
cat qemu-init.sh

#!/bin/sh
busybox echo "init from a minimal initrd!"
busybox insmod rust_helloworld.ko
busybox poweroff -f

cat qemu-initramfs.desc

dir     /bin                                                                            0755 0 0
file    /bin/busybox            busybox                                                 0755 0 0
slink   /bin/sh                 /bin/busybox                                            0755 0 0
file    /init                   qemu-init.sh                                            0755 0 0
file    /rust_helloworld.ko    ../linux-fujita/build/samples/rust/rust_helloworld.ko    0755 0 0
```

<img width="688" alt="图片" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/a249379e-9019-4f10-9f72-6eac9172c035">


## e1000网卡驱动

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise3.md

### Preferences

- https://github.com/fujita/rust-e1000 驱动编译成模块可以不用重新编译内核。把新的ko放到文件系统中即可。指定一个KDIR
- https://github.com/fujita/linux/tree/rust-e1000       too much

- https://github.com/orgs/rcore-os/discussions/30       not for now I guess
- https://github.com/rcore-os/virtio-drivers            cool

### fujita的e1000驱动在官方仓库linux内核编译失败，因此重新clone了fujita的linux rust-e1000分支

```bash
git clone https://github.com/fujita/linux.git -b rust-e1000 --depth=1
```
<img width="717" alt="图片" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/b1dff794-5735-40b4-a585-20baa53a0258">
