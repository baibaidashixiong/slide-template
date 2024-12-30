- 下载`port_bionic`：`git clone https://gitee.com/aosp-riscv-bionic-porting/port_bionic.git`。
- 批量将target为riscv64-unknow-linux-gnu批量替换为riscv64-linux-gnu：
```bash
grep 'riscv64\-unknown\-linux\-gnu' -r ../ | grep target | cut -d':' -f 1 | awk '{print $1}' | xargs sed -i 's/riscv64-unknown-linux-gnu/riscv64-linux-gnu/g'
```
- 编译错误去除Werror：
```bash
override CFLAGS:=$(filter-out -Werror,$(CFLAGS))
```
- `llvm-ar: error: unknown option f`：
```bash
grep -r '\-format\=gnu' | cut -d':' -f 1 | awk '{print $1}' | xargs sed -i 's/-format=gnu/--format=gnu/g'
```
- `no such file /opt/riscv64/lib/gcc/riscv64-linux-gnu/10.1.0/libgcc.a`：
```bash
grep -r '/opt/riscv64/lib/gcc/riscv64-linux-gnu/10.1.0/libgcc.a' | cut -d':' -f 1 | awk '{print $1}' | xargs sed -i 's/\/opt\/riscv64\/lib\/gcc\/riscv64-linux-gnu\/10.1.0\/libgcc.a/\/  
usr\/lib\/gcc-cross\/riscv64-linux-gnu\/11\/libgcc.a/g'
grep -r '/opt/riscv64/lib/gcc/riscv64-linux-gnu/10.1.0/libgcc_eh.a' | cut -d':' -f 1 | awk '{print $1}' | xargs sed -i 's/\/opt\/riscv64\/lib\/gcc\/riscv64-linux-gnu\/10.1.0\/libgcc_e  
h.a/\/usr\/lib\/gcc-cross\/riscv64-linux-gnu\/11\/libgcc_eh.a/g'
```
- `clang: error: invalid linker name in argument '-fuse-ld=lld'`：
```bash
sed -i 's/-fuse-ld=lld/ /g' build/ld_android.mk
```
- `/usr/bin/riscv64-linux-gnu-ld：无法识别的选项‘--use-android-relr-tags’`,`/usr/bin/riscv64-linux-gnu-ld：无法识别的选项‘--pack-dyn-relocs=android+relr’`：
```bash
sed -i 's/-Wl,--use-android-relr-tags/ /g' build/ld_android.mk
sed -i 's/-Wl,--pack-dyn-relocs=android+relr/ /g' build/ld_android.mk
sed -i 's/-Wl,--pack-dyn-relocs=none/ /g' build/libdl.mk
```
- 在libc_shared.mk中去除fuse-ld：
```bash
override LDFLAGS:=$(filter-out -fuse-ld=lld,$(LDFLAGS))
```
- 在a文件夹下找b文件夹：
```bash
find ./ -name a | xargs -I 0 find 0 -name b
```