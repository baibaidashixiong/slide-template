- sshfs挂载文件:
```bash
sudo sshfs -o allow_other,IdentityFile=~/.ssh/id_rsa loongson@10.90.50.92:/home/loongson/develop/ /home/loongson/develop/
```
- 硬盘先同步再挂载:
```bash
sudo rsync -aXS /home/ /data
blkid
UUID=XXXX-XXXX-XXXX（磁盘UUID） /home ext4 nodev,nosuid 0 2
```
- 随机指令序列生成器:RISU(random instruction sequence generator for userspace testing),用于用户态测试架构模型,如qemu(https://git.linaro.org/people/peter.maydell/risu.git).
- add-symbol-file/symbol-file/file的异同:
```bash
add-symbol-file:该命令用于在调试会话中加载额外的符号文件，通常是为了调试动态加载的共享库。
`add-symbol-file /path/to/symbols/mylibrary.so 0x12345678` 0x12345678为info files的入口点(.text段)的偏移地址(textaddress)

symbol-file:Read symbol table information from file filename.
file filename: Use filename as the program to be debugged.
```
- 调试riscv linux:
```bash
# 编译
make ARCH=riscv defconfig
make ARCH=riscv menuconfig在kernel hacking中选上debug info
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j
# 启动
qemu -s -S -kernel bzImage(用于引导系统的压缩内核)
gdb vmlinux(包含调试信息,与bzImage是同一套代码)

# 编译riscv安卓
export SOONG_GEN_COMPDB=1
export SOONG_GEN_COMPDB_DEBUG=1
lunch aosp_riscv64-trunk-userdebug
```
- 编译内核：选取aosp-7-24版本
```bash
git clone https://android.googlesource.com/kernel/goldfish -b android-goldfish-3.18  --depth=1
export PATH=/home/zqz/android/aosp-7-24/aosp/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin:$PATH
make ARCH=arm CROSS_COMPILE=arm-eabi- ranchu_defconfig
make -j
```
- 调试内核：
```bash
# 端口1
./objs/emulator  -kernel /home/zqz/android/aosp-7-24/image/zImage -no-snapshot -verbose -avd Nexus_S_API_24 -qemu -machine virt -s -S
# 端口2
/home/zqz/android/aosp-7-24/aosp/prebuilts/gdb/linux-x86/bin/gdb
# gdbinit
target remote :1234
file /home/zqz/android/aosp-7-24/image/vmlinux
dir /home/zqz/android/kernel/goldfish
b start_kernel
add-symbol-file 
```
- 启动模拟器错误：`# Resetting for Cold Boot Emulator Engine failed`.
```bash
emulator -no-snapshot
```
`Cold boot: Required by the User`:

- 文件系统只读问题:`./objs/emulator -verbose  -writable-system  -avd Nexus_S_API_24 -qemu -machine virt`.
```bash
# 文件系统只读问题
./objs/emulator -verbose  -writable-system  -avd Nexus_S_API_24 -qemu -machine virt
adb root
adb remount
adb push ~/android/library/libart.so /system/lib/libart.so
```
- 调试android系统中的程序：
```bash
# android中
gdbserver 10.0.2.2:2345 /system/bin/pm

# host中（调试native）
adb forward tcp:2345 tcp:2345
~/android/aosp-7-24/aosp/prebuiltsgdb/linux-x86/bin/gdb
file /home/zqz/android/aosp-7-24/aosp/out/target/product/generic/symbols/system/bin/toybox
dir /home/zqz/android/aosp-7-24/aosp/external/toybox
b main
```
- 调试art：
```bash
./objs/emulator -verbose  -writable-system -avd Nexus_S_API_24 -qemu -machine virt
adb root
adb remount
adb push ~/android/library/libart.so /system/lib/libart.so

adb root
adb forward tcp:2345 tcp:2345
adb shell gdbserver 10.0.2.2:2345 am start -n com.android.calendar/com.android.calendar.LaunchActivity

# gdb
# 由于libart是在guest中的库，所以直接file不能调试
target remote :2345
dir /home/zqz/android/aosp-7-24/aosp/
b JNI_CreateJavaVM
b java_vm_ext.cc:939 # VM entry 
c

# 针对<optimized out>
x/d &your_variable
```
- libart中的printf输出到哪里？
```bash
adb shell pm list packages会调用runtime.cc，并调用Runtime::Create
```
- 调试art：在art目录下有一个`tools/art`的脚本用于调试：
```bash
# host
javac HelloWorld.java
dx --dex --output=HelloWorld.jar HelloWorld.class
adb connect IP.ADD.RE.SS
adb push HelloWorld.jar /data/user/
adb shell
# android
cd /data/user
dalvikvm64 -cp HelloWorld.jar HelloWorld # Hello world!
ls -l /data/dalvik-cache/arm64/*HelloWorld* ## Show the output oat file.
# host
adb pull /data/dalvik-cache/arm64/data@user@HelloWorld.jar@classes.dex
oatdump --oat-file=data@user@HelloWorld.jar@classes.dex
```
- 测试art中某个函数的返回值，以为例：首先在对应的Android.bp中加入打印输出文件，注意一定要加`defaults:`关键字。然后`m zqz_test`再运行`out/host/linux-x86/nativetest/zqz_test/zqz_test`。问：为什么测试程序加上`namespace art{}`后会自动跑gtest？
```c++
art_cc_test {
    name: "zqz_test",
    device_supported: false,
    defaults: [
        "art_gtest_defaults",
    ],
    codegen: {
        riscv64: {
            srcs: [
                "utils/riscv64/bits_test.cc",
            ],
        },
    },
}
```
- 调试汇编：
```bash
as a.s -o a.o
ld a.o -o a
readelf -h a
gdb a
(gdb)b *0x120078
(gdb)lay next
(gdb)r
```

#### 安卓使用手册
- CTS&VTS安卓测试套件工具
- 启动应用
```bash
# 列出所有安装包
pm list packages
# 启动📅应用
am start -n com.android.calendar/com.android.calendar.LaunchActivity
```
- 开启JIT日志：
```bash
adb root
adb shell stop
adb shell setprop dalvik.vm.extra-opts -verbose:jit
adb shell start
```
- 停用JIT：
```bash
adb root
adb shell stop
adb shell setprop dalvik.vm.usejit false
adb shell start
```
- 强制编译：
```bash
adb shell cmd package compile
```
- 设置系统启动JIT编译：
```bash
adb shell
su
setprop debug.generate-debug-info true 
setprop dalvik.vm.dex2oat-flags --compiler-filter=quicken
```
- 通过在art目录下搜索`dalvik.vm.`来寻找命令调试art，在`./cmdline/cmdline_parser_test.cc`中有test相关的参数。
- JVM中采用`-XX:`来改变JVM参数进行JVM调优等内容，art在`runtime/parsed_options.cc`有相关设置的参数。
- loongarch启动android：
```bash
su
stop hwservicemanager
start hwservicemanager

# qemu
su
setprop log.tag.dalvikvm64 V
echo 0 > /proc/sys/kernel/randomize_va_space
ifconfig eth0 up
ifconfig eth0 10.0.2.15
ip rule add from all lookup main pref 1

adb shell setprop log.tag.dalvikvm64 V
echo 0 > /proc/sys/kernel/printk

lunch aosp_loongarch64-eng  
export SOONG_GEN_COMPDB=1  
export SOONG_GEN_COMPDB_DEBUG=1  
export ART_TEST_CHROOT=/data/local/art-test-chroot  
export ANDROID_SERIAL=emulator-5554


# 真机
su
setprop log.tag.dalvikvm64 V
echo 0 > /proc/sys/kernel/randomize_va_space
ifconfig eth0 up
ifconfig eth0 10.90.50.51/24
ip route add default via 10.90.50.254

ip route del 10.0.0.0/8

adb devices
adb kill-server
adb shell


stop
setprop dalvik.vm.extra-opts -verbose:jit
setprop dalvik.vm.usejit true
start

```
- 运行/调试测试用例：
```bash
# command
ART_TEST_RUN_TEST_ALWAYS_CLEAN=false art/test.py -r --target --no-prebuild --ndebug --no-image --64 --jit-on-first-use --runtime-option="-verbose:jit" --run-test-option='-Xc  
ompiler-option --dump-cfg=aaa.txt' -j 1 -t 570-checker-osr-locals
/home/zqz/android/aosp.la/art/test/run-test --output-path /home/zqz/debug/out/001-Main-right  --always-clean --chroot /data/local/art-test-chroot  --no-prebuild --compact-dex-level fast --interpreter --no-relocate --runtime-option -Xcheck:jni --no-image --64 001-Main
/home/zqz/android/aosp-arm/art/test/run-test --output-path /home/zqz/debug/out/001-Main-right --never-clean --runtime-option -verbose:jit --chroot /data/local/art-test-chroot --no-prebuild --compact-dex-level fast --jit --runtime-option -Xjitthreshold:0 --no-relocate --runtime-option -Xcheck:jni --no-image --64 001-Main
art/test/run-test --output-path /home/zqz/debug/out/x86 --chroot /data/local/art-test-chroot  --no-prebuild --compact-dex-level fast --interpreter --no-relocate --runtime-option -Xcheck:jni --no-image --64 001-Main
# command
art/test.py -r --target --no-prebuild --debug --no-image --64 --interpreter -j 1 -t 001-Main --gdb
# command
ART_TEST_RUN_TEST_ALWAYS_CLEAN=false art/test.py -r --target --no-prebuild --debug --no-image --64 --jit-on-first-use -j 1 -t 001-Main --gdb
# command
ART_TEST_RUN_TEST_ALWAYS_CLEAN=false art/test.py -r --target --no-prebuild --debug --no-image --64 --jit-on-first-use --runtime-option="-verbose:compiler,jit" --run-test-option='-Xcompiler-option --dump-cfg=aaa.txt -Xcompiler-option --inline-max-code-units=0' -j 1 -t 500-instanceof
# gdbserver
adb forward tcp:5039 tcp:5039
target extended-remote :5039
dir ~/android/aosp.la
set sysroot /home/zqz/android/aosp.la/out/target/product/generic_loongarch64/symbols
set solib-search-path /home/zqz/android/aosp.la/out/target/product/generic_loongarch64/symbols/system/system_ext/apex/com.android.art.testing/lib64
dir ~/android/aosp.la
b art_quick_invoke_static_stub

# 只显示jit的log
adb shell setprop log.tag.dalvikvm64 V
art/test.py -r --target --no-prebuild --debug --runtime-option=-verbose:jit --no-image --64 --jit-on-first-use -j 1 -t 001-Main --gdb
# 关于dalvik.vm的详细相关信息，可以在framework/core/jni/AndroidRuntime.cpp中查看
```
- 在code_gen中生成的代码存放在何处？存放于`Assembler`类中的`protected: AssemblerBuffer`中的`uint8_t* contents_;`地址中。如在CodeGeneratorLOONGARCH64类中想要查看，`x/5i this->GetAssembler().buffer_.contents_`即可。地址是`0x7ffd4a80b8c8`。

#### 工具
- dexdump：用于解析字节码
- profman：用于性能分析：
```bash
profman --profile-file= --dump-only 
This command can dump the contents in the given prof file. 

profman --generate-test-profile= 
This command can generate a random profile file for testing.
```