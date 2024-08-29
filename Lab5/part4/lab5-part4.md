# StratoVirt 与 QEMU 方案对比​

### 问题

1. Linux 启动的协议以及 E820 表是如何设计的？（参考教材 6.4.5 节）

    - Linux启动协议：

        1. 当硬件电源打开后，CPU会自动进入实模式，加载BIOS到0xffff0到0xfffff的位置，此时仅能访问1MB的内存。CPU在0xffff0处运行BIOS，BIOS将会执行某些硬件检测，初始化某些硬件相关的重要信息，并从物理地址0处开始初始化中断向量，在BIOS执行完成后，该区域已经填充了内核头部所需要的所有参数。
        2. BIOS执行完成后就到了跳转至内核的入口点，此时内核将会执行两个函数：`go_to_protected_mode`和`protected_mode_jump`。`go_to_protected_mode`会将CPU设置为保护模式，在进入保护模式前需要调用`setup_idt`和`setup_gdt`函数来安装临时中断描述符表和全局描述符表，保护模式下通过CPU寄存器IDTR访问中断向量表，GDTR寄存器访问全局描述符；调用`protected_mode_jump`函数正式进入保护模式，该函数将设置CPU CR0寄存器中的PE位来启用保护模式，此时最多可以处理4GB的内存。之后将会跳转到32 位内核入口点`startup_32`，正式进行Linux内核的启动流程，引导完成。

    - E820表：

        在x86物理机上，E820是一个可以探测硬件内存分布的硬件，E820表则报告系统内存布局信息的数据结构。E820表中的每一项都是一个E820Entry数据结构，表示一段内存空间，包含了起始地址、结束地址和类型。常见的E820内存类型包括可用内存、已分配内存和保留内存。

2. 请简述 Epoll 实现原理？（参考教材 6.4.7 节）

   Epoll是Linux中提供的一种多路复用I/O模型。其实现原理基于事件驱动，允许一个线程同时监视多个文件描述符的I/O事件，而无需像传统的阻塞I/O模型那样为每个连接创建一个线程。课本中主要用来解决只有在人工输入字符才有串口信息输出的问题。原理如下:
   
   1. 创建Epoll管理结构体内容的对象，用于保存Epoll对象、监听事件和事发生时的闭包处理（事件回调函数处理）；
   2. 需要将监听的文件描述符加入Epoll中，传入的参数是上述代码块定义中EventNotifier结构体，将其设置为监听事件的数据指针；
   3. 创建一个线程进行事件监听循环处理，当监听事件发生时，获取监听事件数据指针，调用其存储的闭包函数，进行事件函数处理。


3. 请执行以下脚本并简述 StratoVirt 的技术重点

   `QA.sh`脚本中包含了StratoVirt的以下技术重点：
   
   1. 串口输出捕获：串口输出的PIO地址为0x3f8，在此处进行陷入
   2. 分配大的连续虚拟内存：最佳方式为mmap
   3. PIO指令处理：通过捕获 PIO 指令（in、out 指令）来实现对敏感指令的处理，在代码中使用 println 进行串行输出。
   4. 设置多个内存映射：通过构建具有不同槽位的多个内存区域来实现guest内存中不同区域的映射，并使用`set_user_memory_region`接口在KVM中更新内存映射
   5. 主机地址查找：获取guest的io地址，通过查找mmap映射来确定host中的相应地址
   6. mmio读写模拟：通过上一步获取host地址，之后对该地址处进行读写
   7. vCPU状态保存：vCPU切换需要保存rax-rdx, rsi, rsp, r8-r15, rip, rflags等通⽤寄存器和cs-ss, ldt, gdt, idt, cr0-cr4等特殊寄存器
   8. 构建E820表：根据虚拟机内存布局，向E820表中添加表项
   9. Epoll实现：问题2中已有描述

### 实验

#### 启动时间对比

| 测试次数 | StratoVirt（ms） | QEMU（ms） |
| :------: | :--------------: | :--------: |
|    1     |       201        |    897     |
|    2     |       216        |    931     |
|    3     |       235        |    959     |
|    4     |       237        |    976     |
|    5     |       222        |    929     |
|    6     |       221        |    942     |
|    7     |       213        |    961     |
|    8     |       199        |    945     |
|    9     |       209        |    952     |
|    10    |       229        |    964     |
|   mean   |      218.2       |   945.6    |

不难看出，StratoVirt的启动时间明显低于QEMU，几乎仅为QEMU的1/4。

#### 内存占用对比

| 测试次数 | StratoVirt（Kbytes） | QEMU（Kbytes） |
| :------: | :------------------: | :------------: |
|    1     |       1126276        |    2071256     |
|    2     |       1126276        |    2113264     |
|    3     |       1126276        |    2074340     |
|    4     |       1126276        |    2077576     |
|    5     |       1126276        |    2083564     |
|    6     |       1126276        |    2337544     |
|    7     |       1126276        |    2271976     |
|    8     |       1126276        |    2285312     |
|    9     |       1126276        |    2277116     |
|    10    |       1126276        |    2301704     |
|   mean   |       1126276        |   2189364.3    |

比较明显的是，StratoVirt进程占用的内存数量固定，而QEMU会在一定范围内波动；同时StratoVirt的内存占用较小，约仅为QEMU的1/2。

#### CPU性能对比

CPU测试中的属性较多，这里以CPU speed（events per second，EPS）对比StratoVirt和QEMU的性能

<table>
	<tr>
		<td>线程</td>
		<td>次数</td>
        <td>StratoVirt(EPS)</td>
        <td>QEMU(EPS)</td>
	</tr>
	<tr>
		<td rowspan="2">1</td>
		<td>1</td>
        <td>870.56</td>
        <td>864.66</td>
	</tr>
    <tr>
		<td>2</td>
        <td>870.47</td>
        <td>866.38</td>
	</tr>
    <tr>
		<td rowspan="2">4</td>
		<td>1</td>
        <td>869.92</td>
        <td>864.22</td>
	</tr>
    <tr>
		<td>2</td>
        <td>860.12</td>
        <td>865.60</td>
	</tr>
    <tr>
		<td rowspan="2">16</td>
		<td>1</td>
        <td>863.74</td>
        <td>863.90</td>
	</tr>
    <tr>
		<td>2</td>
        <td>870.35</td>
        <td>863.71</td>
	</tr>
    <tr>
		<td rowspan="2">32</td>
		<td>1</td>
        <td>863.45</td>
        <td>863.85</td>
	</tr>
    <tr>
		<td>2</td>
        <td>860.47</td>
        <td>858.43</td>
	</tr>
    <tr>
		<td rowspan="2">64</td>
		<td>1</td>
        <td>860.06</td>
        <td>865.87</td>
	</tr>
    <tr>
		<td>2</td>
        <td>862.54</td>
        <td>864.96</td>
	</tr>
    <tr>
		<td colspan="2">mean</td>
        <td>865.168</td>
        <td>864.158</td>
	</tr>
</table>

可以看到StratoVirt和QEMU的CPU性能基本相同

#### 内存性能对比

内存性能对比以吞吐量为例，分别对此顺序访问和随机访问两种情况

- 顺序访问：

  <table>
  	<tr>
  		<td>内存块大小</td>
  		<td>次数</td>
          <td>StratoVirt(MiB/sec)</td>
          <td>QEMU(MiB/sec)</td>
  	</tr>
  	<tr>
  		<td rowspan="2">1k</td>
  		<td>1</td>
          <td>4510.79</td>
          <td>4113.27</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>4525.40</td>
          <td>3983.74</td>
  	</tr>
      <tr>
  		<td rowspan="2">2k</td>
  		<td>1</td>
          <td>6988.34</td>
          <td>6480.83</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>6988.67</td>
          <td>6448.74</td>
  	</tr>
      <tr>
  		<td rowspan="2">4k</td>
  		<td>1</td>
          <td>9586.58</td>
          <td>9112.55</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>9579.90</td>
          <td>9058.08</td>
  	</tr>
      <tr>
  		<td rowspan="2">8k</td>
  		<td>1</td>
          <td>11608.34</td>
          <td>11331.07</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>11641.43</td>
          <td>11271.9</td>
  	</tr>
  </table>

- 随机访问：

  <table>
  	<tr>
  		<td>内存块大小</td>
  		<td>次数</td>
          <td>StratoVirt(MiB/sec)</td>
          <td>QEMU(MiB/sec)</td>
  	</tr>
  	<tr>
  		<td rowspan="2">1k</td>
  		<td>1</td>
          <td>1385.38</td>
          <td>1338.03</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>1387.68</td>
          <td>1336.24</td>
  	</tr>
      <tr>
  		<td rowspan="2">2k</td>
  		<td>1</td>
          <td>1517.37</td>
          <td>1492.08</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>1516.29</td>
          <td>1487.72</td>
  	</tr>
      <tr>
  		<td rowspan="2">4k</td>
  		<td>1</td>
          <td>1630.12</td>
          <td>1611.55</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>1556.81</td>
          <td>1611.46</td>
  	</tr>
      <tr>
  		<td rowspan="2">8k</td>
  		<td>1</td>
          <td>1651.10</td>
          <td>1638.73</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>1692.27</td>
          <td>1679.67</td>
  	</tr>
  </table>

对比之下发现，不论是顺序访问还是随机访问，StratoVirt的内存访问吞吐量都略高于QEMU。随着block size的增大，内存访问的吞吐量也提高，因为读写的局部性提高。

#### I/O 性能对比

用于测试的命令如下（以随机读，threads=1，bs=4K为例）：

```bash
sysbench fileio --time=60 --threads=1 --file-block-size=4K --file-test-mode=rndrd prepare
sysbench fileio --time=60 --threads=1 --file-block-size=4K --file-test-mode=rndrd run
sysbench fileio --time=60 --threads=1 --file-block-size=4K --file-test-mode=rndrd cleanup
```

测试的数据如下：

- threads=1，bs=4K，单队列读写延迟：

  <table>
  	<tr>
  		<td>模式</td>
  		<td>次数</td>
          <td>StratoVirt(ms)</td>
          <td>QEMU(ms)</td>
  	</tr>
  	<tr>
  		<td rowspan="2">rndrd</td>
  		<td>1</td>
          <td>0.103</td>
          <td>0.043</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>0.100</td>
          <td>0.042</td>
  	</tr>
      <tr>
  		<td rowspan="2">rndwr</td>
  		<td>1</td>
          <td>0.406</td>
          <td>0.501</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>0.410</td>
          <td>0.487</td>
  	</tr>
      <tr>
  		<td rowspan="2">seqrd</td>
  		<td>1</td>
          <td>0.017</td>
          <td>0.006</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>0.016</td>
          <td>0.007</td>
  	</tr>
      <tr>
  		<td rowspan="2">seqwr</td>
  		<td>1</td>
          <td>0.080</td>
          <td>0.042</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>0.080</td>
          <td>0.043</td>
  	</tr>
  </table>

  在rndwr，StratoVirt的延迟低于QEMU；其他情况下，均高于QEMU

- threads=32，bs=128K，吞吐量：

  <table>
  	<tr>
  		<td>模式</td>
  		<td>次数</td>
          <td>StratoVirt(MiB/s)</td>
          <td>QEMU(MiB/s)</td>
  	</tr>
  	<tr>
  		<td rowspan="2">rndrd</td>
  		<td>1</td>
          <td>1906.82</td>
          <td>2002.86</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>1938.16</td>
          <td>1979.19</td>
  	</tr>
      <tr>
  		<td rowspan="2">rndwr</td>
  		<td>1</td>
          <td>510.23</td>
          <td>350.59</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>513.95</td>
          <td>351.98</td>
  	</tr>
      <tr>
  		<td rowspan="2">seqrd</td>
  		<td>1</td>
          <td>1613.34</td>
          <td>1730.60</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>1666.86</td>
          <td>1674.12</td>
  	</tr>
      <tr>
  		<td rowspan="2">seqwr</td>
  		<td>1</td>
          <td>524.58</td>
          <td>443.02</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>529.48</td>
          <td>447.40</td>
  	</tr>
  </table>

  读时，StratoVirt的吞吐量略低于QEMU；写时，StratoVirt的吞吐量略高于QEMU

- threads=32，bs=4K，IOPS：

  <table>
  	<tr>
  		<td>模式</td>
  		<td>次数</td>
          <td>StratoVirt(/s)</td>
          <td>QEMU(/s)</td>
  	</tr>
  	<tr>
  		<td rowspan="2">rndrd</td>
  		<td>1</td>
          <td>73323.04</td>
          <td>75297.23</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>72923.20</td>
          <td>76058.04</td>
  	</tr>
      <tr>
  		<td rowspan="2">rndwr</td>
  		<td>1</td>
          <td>21842.47</td>
          <td>18511.15</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>21919.31</td>
          <td>18644.37</td>
  	</tr>
      <tr>
  		<td rowspan="2">seqrd</td>
  		<td>1</td>
          <td>126431.89</td>
          <td>135335.13</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>125315.46</td>
          <td>135428.63</td>
  	</tr>
      <tr>
  		<td rowspan="2">seqwr</td>
  		<td>1</td>
          <td>51119.93</td>
          <td>24406.12</td>
  	</tr>
      <tr>
  		<td>2</td>
          <td>51047.86</td>
          <td>24015.91</td>
  	</tr>
  </table>

  读时，StratoVirt的吞IOPS低于QEMU；写时，StratoVirt的吞吐量高于QEMU

#### 总结

总体来看，StratoVirt在启动时间、内存占用方面的表现都优于QEMU，这符合StratoVirt设计时的轻量化特点。在性能方面，StratoVirt也不弱于QEMU，在内存访问、CPU性能以及IO的写时吞吐量方面都有强于QEMU的表现。可能的原因如下：

- 在设计思想方面：StratoVirt专注于提供最⼩化的虚拟化功能，仅关注嵌套虚拟化场景，满足核心的虚拟化需求。相较之下，QEMU为了支持完整的虚拟化功能（包括用户态虚拟化和全系统虚拟化），引入了一些较为庞大的架构和模块
- 在编程语言方面：QEMU则使用C语言开发，而StratoVirt使用Rust编程语言进行开发，Rust具有内存安全和高并发性等优点，在高并发场景和异步编程时，更有优势
- 在实现方案方面：StratoVirt针对轻量级容器内虚拟机提供了最小化的虚拟化功能，采用了单一进程的架构，通过KVM提供轻量级虚拟化支持，并采用了异步编程模型、零拷贝等技术；而QEMU在实现时考虑了多种硬件和架构的模拟，这使得其代码相对庞大和复杂，对新技术的支持也不如StratoVirt
