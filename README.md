# rust-for-linux

## Env

- Debian GNU/Linux 12 (bookworm) aarch64 in Parallel Vitrual Machine
- QEMU qemu-system-aarch64 version 7.2.5
- rust-for-linux arm64 6.6.0-rc4

## 编译内核

```bash
git clone https://github.com/Rust-for-Linux/linux -b rust-dev --depth=1
```

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise1.md

按照指导安装依赖，然后

```bash
make ARCH=arm64 LLVM=1 O=build defconfig

make ARCH=arm64 LLVM=1 O=build menuconfig
#set the following config to yes
General setup
        ---> [*] Rust support

make ARCH=arm64 LLVM=1 -j8
```

<img width="611" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/dc694004-6721-4866-baa9-88de45eeb71c">

## QEMU启动

### QEMU启动Debian Quick Image Baker(DQIB)

#### 安装QEMU

```bash
sudo apt install qemu-user qemu-system-arm # 安装qemu 7.x.x
```

> qemu 8.x.x以上运行DQIB kernel会报错 => "...net user not compiled into kernel..."

#### 下载DQIB

- https://people.debian.org/~gio/dqib/

按照readme.txt 先启动DQIB默认内核

```bash
$ qemu-system-aarch64
-machine 'virt'
-cpu 'cortex-a57'
-m 1G
-device virtio-blk-device,drive=hd
-drive file=image.qcow2,if=none,id=hd
-device virtio-net-device,netdev=net
-netdev user,id=net,hostfwd=tcp::2222-:22
-kernel kernel
-initrd initrd
-nographic
-append "root=LABEL=rootfs console=ttyAMA0"
```

<img width="1233" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/2c14c348-a2b1-463a-8fc3-4640102d11e0">

## 编译Rust模块helloworld

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise2.md

按照指导添加代码，但是会有下面这个报错
需要把第一个参数去掉
```bash
error[E0050]: method `init` has 2 parameters but the declaration in trait `kernel::Module::init` has 1
  --> ../samples/rust/rust_helloworld.rs:17:19
   |
17 | ...me:&'static Cstr, _module: &'static ThisModule) -...
   |       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected 1 parameter, found 2
   |
   = note: `init` from trait: `fn(&'static kernel::ThisModule) -> core::result::Result<Self, kernel::error::Error>`

error: aborting due to 2 previous errors
```

```bash
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <*>Print Helloworld in Rust (NEW) # 其它也要勾选
```

<img width="612" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/9644eae3-c735-4a58-ba71-8fc4ff65658e">

修改QEMU启动命令

```bash
$ qemu-system-aarch64
-machine 'virt'
-cpu 'cortex-a57'
-m 1G
-device virtio-blk-device,drive=hd
-drive file=image.qcow2,if=none,id=hd
-device virtio-net-device,netdev=net
-netdev user,id=net,hostfwd=tcp::2222-:22
-kernel /home/junzhe/Dev/Software/linux/build/arch/arm64/boot/Image # 添加编译好的内核
-initrd initrd
-nographic
-append "root=LABEL=rootfs console=ttyAMA0"
```

<img width="1233" alt="image" src="https://github.com/ye-junzhe/rust-for-linux/assets/53103747/70e5fff4-41cf-49e5-9fcc-959124876ef4">

## e1000网卡驱动

- https://github.com/rcore-os/rust-for-linux/blob/main/exercise3.md

### Preferences

- https://github.com/fujita/rust-e1000 驱动编译成模块可以不用重新编译内核。把新的ko放到文件系统中即可。指定一个KDIR
- https://github.com/fujita/linux/tree/rust-e1000       too much

- https://github.com/orgs/rcore-os/discussions/30       not for now I guess
- https://github.com/rcore-os/virtio-drivers            cool