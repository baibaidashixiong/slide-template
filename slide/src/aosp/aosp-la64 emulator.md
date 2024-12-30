- [android emulator开发手册](https://android.googlesource.com/platform/external/qemu/+/refs/heads/aosp-emu-30-release/android/docs/LINUX-DEV.md)。
- 所需安装包：
```bash
apt install libicu-dev linux-libc-dev libpulse-dev clang libaom-dev
apt install libsnappy-dev libaom-dev libcodec2-dev libgsm1-dev libmp3lame0 libopenjp2-7-dev libopus-dev libshine-dev librsvg2-dev libcairo2-dev libva-dev libspeex-dev libvorbis-dev libtheora-dev libsoxr-dev libtwolame-dev libwebp-dev libx265-dev libxvidcore-dev libwavpack-dev android-libcutils-dev
```
- 列出包中安装到系统的文件：`dpkg-query -L libopenjp2-7-dev`.
- 
- 下载源码：目前所用分支为emu-32-release
```bash
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b emu-32-release --depth=1
repo sync -c
```
- 切换分支：
```bash
repo init -b emu-32-release
repo sync -l
```
- repo工具：
```bash
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod +x repo
```
- android修改远程url：
```bash
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest
```
- 替换已有aosp的remote：`.repo/manifests.git/config`
```bash
url = https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest

git config --global url.https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/.insteadof https://android.googlesource.com
```
- repo创立分支：
```bash
repo start la --all
```
- 运行编译命令：
```bash
./android/rebuild.sh --ccache=none --no-clean
```
- 运行gen-sdk：
```bash
./android/scripts/unix/gen-android-sdk-toolchain.sh --host=linux-loongarch64 /root/android/emulator/external/qemu/objs/toolchain --aosp-dir=/root/android/emulator --aosp-clang_ver=clang-r450784d
/root/android/emulator/external/qemu/android/scripts/unix/gen-android-sdk-toolchain.sh --host=linux-loongarch64 /root/android/emulator/external/qemu/objs/toolchain --aosp-dir=/root/android/emulator --aosp-clang_ver=clang-r450784d --verbosity=2
```
- 非loongarch的common包：
- [ ] qt/qtwebengine
- [ ] 
- docker中设置latex自动运行：设置mmap最小设置栈大小不要太大也不要太小（原本为4096）；设置binformat；
- 想用latx来运行x86程序，但是debian的docker无法用priviledged模式启动
```bash
# 需要设置docker为priviliged
echo 65536 > /proc/sys/vm/mmap_min_addr
ulimit -s 8192

echo ':x86_64:M::\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff  
\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/latx-x86_64:' > /proc/sys/fs/binfmt_misc/register
```
- sccache问题：
```bash
sccache: error: failed to execute compile  
sccache: caused by: Failed to send data to or receive data from server  
sccache: caused by: Failed to read response header  
sccache: caused by: failed to fill whole buffer
```
- external/cares/ares_setup.h：
```bash
config_linux/ares_config.h
```
- `error: '_Atomic' does not name a type`：
```
#ifdef __cplusplus  
#include <atomic>  
using namespace std;  
#else  
#include <stdatomic.h>  
#endif
```
- `loongarch64-linux-gnu-g++: error: unrecognized command line option '-fno-emulated-tls'; did you mean '-Wno-templates'?`：在`device/generic/goldfish-opengl/system/OpenglSystemCommon/Android.mk`和`device/generic/goldfish-opengl/system/OpenglSystemCommon/CMakeLists.txt`中去除`-fno-emulated-tls`。
- `use of undeclared identifier 'fmax'; did you mean 'max'?`：添加`#include <math.h>`。
- `'annotate' attribute directive ignored [-Werror=attributes]`：在文件中加入
```bash
#pragma GCC diagnostic ignored "-Wattributes"
```
- `/usr/bin/ld: android/bluetooth/rootcanal/bluetooth_packetgen_ext/CMakeFiles/bluetooth_packetgen.dir/root/android/emulator/packages/modules/Bluetooth/system/gd/packet/parser/gen_cpp.cc.o: in function 'no symbol':main.cc:(.text+0x1018): undefined reference to std::filesystem::__cxx11::path::_M_split_cmpts()'`：在`android/bluetooth/packet_gen/CMakeLists.txt`编译选项中加入`target_link_libraries(bluetooth_packetgen PRIVATE stdc++fs)`。
- `error: cannot initialize a parameter of type 'unsigned long long *' with an rvalue of type 'CUdeviceptr *' (aka 'unsigned int *')`：修改指针类型。
- `/usr/bin/ld: lib64/libandroid-emu-metrics.so: undefined reference to ucnv_setToUCallBack_67'`：在`emu/metrics/CMakeLists.txt`中加入：
```bash
target_link_libraries(PRIVATE -licuuc)
```
- moc错误：使用系统moc
- `android-emu-adb-interface_unittests`错误：在`android/emu/CMakeLists.txt`中增加：
```bash
target_link_libraries(android-emu-adb-interface_unittests PRIVATE -lsnappy -laom -lcodec2 -lgsm -lmp3lame -lopenjp2 -lopus -lshine -lX11 -lva -lspeex -lvorbis -lsoxr -ltwolame -lvorbisenc -lwebpmux -lx265 -lxvidcore -lzvbi  -lcairo  -lvdpau -ltheoradec -ltheoraenc -ldrm  -lva-drm -lva-x11 -lwavpack  -lglib-2.0 -lrsvg-2 -lgobject-2.0 -lffi)  
target_link_libraries(android-emu-feature_unittests PRIVATE -lsnappy -laom -lcodec2 -lgsm -lmp3lame -lopenjp2 -lopus -lshine -lX11 -lva -lspeex -lvorbis -lsoxr -ltwolame -lvorbisenc -lwebpmux -lx265 -lxvidcore -lzvbi  -lcairo  -lvdpau -ltheoradec -ltheoraenc -ldrm  -lva-drm -lva-x11 -lwavpack  -lglib-2.0 -lrsvg-2 -lgobject-2.0 -lffi)
target_link_libraries(android-emu-studio-config_unittests PRIVATE -lz -licuuc)
```
- `CrashReporter.cpp:(.text+0xb8): undefined reference to android::crashreport::consentProvider()'`：在`android/android-emugl/host/libs/libOpenglRender/CMakeLists.txt`中AARCH64附近进行修改。
- `qemu-system-aarch64`编译错误：
```bash
-lchromaprint -lgme -lopenmpt -lbz2 -lgnutls -lbluray -lssh -ldrm -laom -lcodec2 -lgsm -lmp3lame -lopenjp2 -lopus -lshine -lX11 -lva -lspeex -lvorbis -lsoxr -ltwolame -lvorbisenc -lwebp -lwebpmux -lx265 -lxvidcore -lzvbi -lcairo -lvdpau -ltheoradec -ltheoraenc -ldrm -lva-drm -lva-x11 -lwavpack -lrsvg-2 -lgobject-2.0 -lglib-2.0 -lffi -lsnappy
```
- x86上运行arm aosp：
```bash
adb nodaemon server
./emulator -no-window -avd Nexus_S_API_26 -qemu -machine virt
```
- 磁盘修复：
```bash
lsof /home/loongson/develop/
sudo umount /dev/sdb1
sudo dmesg
sudo fsck -fyC /dev/sdb1
```
- 使用ag命令搜索当前目录下除去`.c/.h`的所有文件：
```bash
ag -G '^(?!.*/*[.](c|h)$)' "kvm" --ignore-dir objs.bak/
```
- 调试工具：
```bash
addresssantizer
systemtap

```