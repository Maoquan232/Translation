# 嵌入式调试

官网英文原文地址：http://dev.px4.io/advanced-gdb-debugging.html

运行PX4的自驾仪支持通过GDB或LLDB进行调试

## 查看内存消耗

下面的命令可以列出最大的静态内存分配

<div class="host-code"></div>

```bash
arm-none-eabi-nm --size-sort --print-size --radix=dec build_px4fmu-v2_default/src/firmware/nuttx/firmware_nuttx | grep " [bBdD] "
```

NSH中的命令可以查看剩余内存：

```bash
free
```

Top命令可以看到每个应用占用的堆栈使用：

```
top
```
应用的堆栈使用是通过stack coloring计算，因此得到的并不是当前堆栈使用情况，，而是从任务开始以来最大的堆栈占用值。

### 堆栈分配
动态堆栈分配可以在运行SITL仿真时通过POSIX中的[gperftools](https://github.com/gperftools/gperftools)进行追踪.
安装好后，按下面方法使用:
  * Run jmavsim: `cd Tools/jMAVSim/out/production && java -Djava.ext.dirs= -jar jmavsim_run.jar -udp 127.0.0.1:14560`
  * Then:

```bash
cd build_posix_sitl_default/tmp
export HEAPPROFILE=/tmp/heapprofile.hprof
env LD_PRELOAD=/lib64/libtcmalloc.so ../src/firmware/posix/px4 posix-configs/SITL/init/lpe/iris
 +pprof --pdf ../src/firmware/posix/px4 /tmp/heapprofile.hprof.0001.heap > heap.pdf
```　。
它会产生一个包含堆栈分配曲线图的PDF文件。图表中的数字都是0因这是在MB中。可以参考的是各个应用所占堆栈的百分比，他们表示应用占用的实时内存。

## Sending MAVLink debug key / value pairs

关于这部分的代码如下:

  * [Debug Tutorial Code](https://github.com/PX4/Firmware/blob/master/src/examples/px4_mavlink_debug/px4_mavlink_debug.c)
  * [Enable the tutorial app](https://github.com/PX4/Firmware/tree/master/cmake/configs) by uncommenting / enabling the mavlink debug app in the config of your board

设置一个调试发布topic只需要小部分代码，首先是头文件：

<div class="host-code"></div>

```C
#include <uORB/uORB.h>
#include <uORB/topics/debug_key_value.h>
```

然后广告调试值主题（不同发布名称只需要一次广告）。把下面部分放在主循环之前：

## 在NuttX中调试硬故障

硬故障是由于运行的系统检测到没有有效的可指令执行时出现的状态。经常是因为RAM中的关键部分出现崩溃。典型的情况是，不正确的内存地址进入堆栈让处理器认为内存中的地址无效了。
  * NuttX有两个堆栈: 中断处理的IRQ堆栈和用户堆栈
  * 堆栈向下堆积，所以下面例子中栈顶的地址是 0x20021060, 大小是0x11f4 (4596 bytes)，栈低地址是 0x2001fe6c.

```bash
Assertion failed at file:armv7-m/up_hardfault.c line: 184 task: ekf_att_pos_estimator
sp:     20003f90
IRQ stack:
  base: 20003fdc
  size: 000002e8
20003f80: 080d27c6 20003f90 20021060 0809b8d5 080d288c 000000b8 08097155 00000010
20003fa0: 20003ce0 00000003 00000000 0809bb61 0809bb4d 080a6857 e000ed24 080a3879
20003fc0: 00000000 2001f578 080ca038 000182b8 20017cc0 0809bad1 20020c14 00000000
sp:     20020ce8
User stack:
  base: 20021060
  size: 000011f4
20020ce0: 60000010 2001f578 2001f578 080ca038 000182b8 0808439f 2001fb88 20020d4c
20020d00: 20020d44 080a1073 666b655b 65686320 205d6b63 6f6c6576 79746963 76696420
20020d20: 65747265 63202c64 6b636568 63636120 63206c65 69666e6f 08020067 0805c4eb
20020d40: 080ca9d4 0805c21b 080ca1cc 080ca9d4 385833fb 38217db9 00000000 080ca964
20020d60: 080ca980 080ca9a0 080ca9bc 080ca9d4 080ca9fc 080caa14 20022824 00000002
20020d80: 2002218c 0806a30f 08069ab2 81000000 3f7fffec 00000000 3b4ae00c 3b12eaa6
20020da0: 00000000 00000000 080ca010 4281fb70 20020f78 20017cc0 20020f98 20017cdc
20020dc0: 2001ee0c 0808d7ff 080ca010 00000000 3f800000 00000000 080ca020 3aa35c4e
20020de0: 3834d331 00000000 01010101 00000000 01010001 000d4f89 000d4f89 000f9fda
20020e00: 3f7d8df4 3bac67ea 3ca594e6 be0b9299 40b643aa 41ebe4ed bcc04e1b 43e89c96
20020e20: 448f3bc9 c3c50317 b4c8d827 362d3366 b49d74cf ba966159 00000000 00000000
20020e40: 3eb4da7b 3b96b9b7 3eead66a 00000000 00000000 00000000 00000000 00000000
20020e60: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
20020e80: 00000016 00000000 00000000 00010000 00000000 3c23d70a 00000000 00000000
20020ea0: 00000000 20020f78 00000000 2001ed20 20020fa4 2001f498 2001f1a8 2001f500
20020ec0: 2001f520 00000003 2001f170 ffffffe9 3b831ad2 3c23d70a 00000000 00000000
20020ee0: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
20020f00: 00000000 00000000 00000000 00000000 2001f4f0 2001f4a0 3d093964 00000001
20020f20: 00000000 0808ae91 20012d10 2001da40 0000260b 2001f577 2001da40 0000260b
20020f40: 2001f1a8 08087fd7 08087f9d 080cf448 0000260b 080afab1 080afa9d 00000003
20020f60: 2001f577 0809c577 2001ed20 2001f4d8 2001f498 0805e077 2001f568 20024540
20020f80: 00000000 00000000 00000000 0000260b 3d093a57 00000000 2001f540 2001f4f0
20020fa0: 0000260b 3ea5b000 3ddbf5fa 00000000 3c23d70a 00000000 00000000 000f423f
20020fc0: 00000000 000182b8 20017cc0 2001ed20 2001f4e8 00000000 2001f120 0805ea0d
20020fe0: 2001f090 2001f120 2001eda8 ffffffff 000182b8 00000000 00000000 00000000
20021000: 00000000 00000000 00000009 00000000 08090001 2001f93c 0000000c 00000000
20021020: 00000101 2001f96c 00000000 00000000 00000000 00000000 00000000 00000000
20021040: 00000000 00000000 00000000 00000000 00000000 0809866d 00000000 00000000
R0: 20000f48 0a91ae0c 20020d00 20020d00 2001f578 080ca038 000182b8 20017cc0
R8: 2001ed20 2001f4e8 2001ed20 00000005 20020d20 20020ce8 0808439f 08087c4e
xPSR: 61000000 BASEPRI: 00000000 CONTROL: 00000000
EXC_RETURN: ffffffe9
```

为了解码硬故障，把二进制文件exact导入调试器:

<div class="host-code"></div>

```bash
arm-none-eabi-gdb build_px4fmu-v2_default/src/firmware/nuttx/firmware_nuttx
```

然后在GDB的提示中，从R8的最后一条指令也就是内存中的第一个地址开始，(因为地址从 `0x080`开始, 第一条指令是 `0x0808439f`)。 从左到右执行，硬故障前的最后一步是 ```mavlink_log.c``` 要发布东西,这样就找到了硬故障 

<div class="host-code"></div>

```gdb
(gdb) info line *0x0808439f
Line 77 of "../src/modules/systemlib/mavlink_log.c" starts at address 0x8084398 <mavlink_vasprintf+36>
   and ends at 0x80843a0 <mavlink_vasprintf+44>.
```

<div class="host-code"></div>

```gdb
(gdb) info line *0x08087c4e
Line 311 of "../src/modules/uORB/uORBDevices_nuttx.cpp"
   starts at address 0x8087c4e <uORB::DeviceNode::publish(orb_metadata const*, void*, void const*)+2>
   and ends at 0x8087c52 <uORB::DeviceNode::publish(orb_metadata const*, void*, void const*)+6>.
```
