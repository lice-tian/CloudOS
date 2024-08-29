# Lab3：探究虚拟化环境下SPDK存储设备的性能
<div style="text-align:right;">
    姓名：李天策<br>
    学号：1230379100163<br>
    邮箱：tiancelipipa@sjtu.edu.cn
</div>

## 实验设计
由于没有额外的SSD，本次实验采用了（双系统）物理机+虚拟磁盘文件的方案。

##### 1.使用环境：
- 宿主机（物理机）：
  - 系统/内核：Ubuntu-20.04.6 / Linux 5.15
  - CPU/内存：Intel(R) Core(TM) i7-9750H，x86_64（architecture），12（cores）/ 8G
- 虚拟磁盘文件大小：5G（1G-block、写入5次）
- 客户机（虚拟机）：
  - 系统/内核：Ubuntu-20.04.6 / Linux 5.15
  - CPU/内存（QEMU块设备测试方法）：4（cores）/ 4G
  - CPU/内存（vhost-user-blk）：6（cores）/ 3G
- QEMU：4.2.1，SPDK：23.09，fio：3.27

##### 2.测试方法指标参数：
在实验中，我们采用了4种方法测试了宿主机上的虚拟磁盘。具体方式如下
- 宿主机io_uring块设备（附录：bench.conf）
- 宿主机SPDK uring块设备（附录：bench-spdk-bdev.conf+bdev.json）
- 客户机QEMU块设备（附录：bench.conf）：`-drive file=/home/tianceli/CloudOS/Lab3/bdev.img,if=none,id=bdev`
- 客户机vhost-user-blk（附录：bench.conf）`-device vhost-user-blk-pci,chardev=spdk_vhost_blk0,num-queues=2`

*注：bench.conf仅修改了filename*

## 实验数据
实验中通过fio测试了QD=1和QD=32情况下randomread 4K的P50, P95, P99, P999, MAX, AVG latency,IOPS和bandwidth，实验结果如下：

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-pb0m{border-color:inherit;text-align:center;vertical-align:bottom}
.tg .tg-bobw{font-weight:bold;text-align:center;vertical-align:bottom}
.tg .tg-fll5{border-color:inherit;font-weight:bold;text-align:center;vertical-align:bottom}
.tg .tg-7btt{border-color:inherit;font-weight:bold;text-align:center;vertical-align:top}
.tg .tg-uzvj{border-color:inherit;font-weight:bold;text-align:center;vertical-align:middle}
.tg .tg-8d8j{text-align:center;vertical-align:bottom}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-pb0m" colspan="2" rowspan="2"></th>
    <th class="tg-fll5" colspan="2">宿主机<br>io_uring </th>
    <th class="tg-fll5" colspan="2">宿主机<br>spdk_uring</th>
    <th class="tg-7btt" colspan="2"> 客户机<br>qemu_blk</th>
    <th class="tg-fll5" colspan="2">客户机<br>vhost_user_blk</th>
  </tr>
  <tr>
    <th class="tg-fll5">QD=1</th>
    <th class="tg-fll5">QD=32</th>
    <th class="tg-fll5">QD=1</th>
    <th class="tg-fll5">QD=32</th>
    <th class="tg-fll5">QD=1</th>
    <th class="tg-fll5">QD=32</th>
    <th class="tg-fll5">QD=1</th>
    <th class="tg-fll5">QD=32</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-uzvj" rowspan="6">Latency(us)</td>
    <td class="tg-fll5">P50</td>
    <td class="tg-pb0m">97</td>
    <td class="tg-pb0m">149</td>
    <td class="tg-pb0m">99</td>
    <td class="tg-pb0m">151</td>
    <td class="tg-pb0m">192</td>
    <td class="tg-pb0m">251</td>
    <td class="tg-pb0m">113</td>
    <td class="tg-pb0m">180</td>
  </tr>
  <tr>
    <td class="tg-fll5">P95 </td>
    <td class="tg-pb0m">114</td>
    <td class="tg-pb0m">355</td>
    <td class="tg-pb0m">116</td>
    <td class="tg-pb0m">355</td>
    <td class="tg-pb0m">273</td>
    <td class="tg-pb0m">437</td>
    <td class="tg-pb0m">131</td>
    <td class="tg-pb0m">375</td>
  </tr>
  <tr>
    <td class="tg-fll5">P99</td>
    <td class="tg-pb0m">117</td>
    <td class="tg-pb0m">465</td>
    <td class="tg-pb0m">116</td>
    <td class="tg-pb0m">474</td>
    <td class="tg-pb0m">404</td>
    <td class="tg-pb0m">578</td>
    <td class="tg-pb0m">143</td>
    <td class="tg-pb0m">490</td>
  </tr>
  <tr>
    <td class="tg-fll5">P999</td>
    <td class="tg-pb0m">2114</td>
    <td class="tg-pb0m">2147</td>
    <td class="tg-pb0m">2073</td>
    <td class="tg-pb0m">4178</td>
    <td class="tg-pb0m">9372</td>
    <td class="tg-pb0m">5932</td>
    <td class="tg-pb0m">3326</td>
    <td class="tg-pb0m">2212</td>
  </tr>
  <tr>
    <td class="tg-fll5">MAX</td>
    <td class="tg-pb0m">9356</td>
    <td class="tg-pb0m">3551</td>
    <td class="tg-pb0m">2180</td>
    <td class="tg-pb0m">11862</td>
    <td class="tg-pb0m">61479</td>
    <td class="tg-pb0m">11599</td>
    <td class="tg-pb0m">6416</td>
    <td class="tg-pb0m">5060</td>
  </tr>
  <tr>
    <td class="tg-fll5">AVG</td>
    <td class="tg-pb0m">102.58</td>
    <td class="tg-pb0m">174.52</td>
    <td class="tg-pb0m">102.9</td>
    <td class="tg-pb0m">178.79</td>
    <td class="tg-pb0m">200.46</td>
    <td class="tg-pb0m">268.13</td>
    <td class="tg-pb0m">120.48</td>
    <td class="tg-pb0m">203.15</td>
  </tr>
  <tr>
    <td class="tg-uzvj" colspan="2">bandwidth(KiB/s)</td>
    <td class="tg-pb0m">38911.21</td>
    <td class="tg-pb0m">733617.9</td>
    <td class="tg-pb0m">38694.05</td>
    <td class="tg-pb0m">712824.8</td>
    <td class="tg-pb0m">19965.68</td>
    <td class="tg-pb0m">477125.68</td>
    <td class="tg-pb0m">33040.47</td>
    <td class="tg-pb0m">628724.42</td>
  </tr>
  <tr>
    <td class="tg-bobw" colspan="2">IOPS</td>
    <td class="tg-8d8j">9727.74</td>
    <td class="tg-8d8j">183404.4</td>
    <td class="tg-8d8j">9673.47</td>
    <td class="tg-8d8j">178206</td>
    <td class="tg-8d8j">4991.26</td>
    <td class="tg-8d8j">119281.32</td>
    <td class="tg-8d8j">8260</td>
    <td class="tg-8d8j">157181</td>
  </tr>
</tbody>
</table>

## 实验分析

- 对比**同一组实验QD=1和QD=32**:不难发现，队列深度（I/O深度）增大时，平均延迟会增大，相应地，带宽和IOPS会增大。随着I/O深度增加，I/O请求在队列中的等待时间会变长，即延迟增大；但也会使并发量增大，从而增大带宽和IOPS。
- 对比**宿主机上使用io_uring和spdk**：可以发现，在宿主机上使用spdk几乎没有性能提升，甚至有所降低（平均延迟增大，带宽和IOPS减小）。因为本次实验中，我使用了虚拟磁盘文件，因此spdk无法绕开内核层，没有办法避免系统调用带来的性能损耗，因此两个实验性能相近，spdk方法性能甚至略微下降。
- 对比**客户机上qemu_blk和vhost_user_blk**：可以发现，在客户机上使用spdk可以明显提高性能（平均延迟减小，带宽和IPOS增大）。SPDK vhost通过应用和其他的SPDK组件相同的用户空间和轮询技术，避免虚拟机在I/O提交时出发VMEXIT$^{[2]}$（有点类似绕过虚拟机的内核层），从而相对于QEMU块结构提高了I/O性能。

## 思考与练习

- **如果有不合常理的数据点，请叙述你希望的结果，并分析为什么结果不合常理。**

  - 不合常理的数据点：对比四组实验的Max Latency发现，“宿主机io_uring”、“客户机qemu_blk”和“客户机vhost_user_blk”三组实验在QD=32时的clat要小于QD=1时的clat，但“宿主机spdk_uring”中则相反；
  - 期望的结果：“宿主机spdk_uring”实验中，QD=32的max clat应小于QD=1时的max clat；
  - 原因：按照我的理解，队列深度增大会增加IO操作的延迟，但同时也会增加IO稳定性，即随着可并发数的增大，会使极端的高延迟情况减少，即只有极少数的请求会经历较长的延迟。在QD=32时，max clat应该较小；但“宿主机spdk_uring”却较大，这可能是因为出现了I/O竞争或过度排队的情况，因此出现了极端情况。

- **简述不同挂载方式的区别和优劣。**
  - 区别：QEMU块设备方式直接将块设备挂载到虚拟机中，没有涉及额外的中间件；vhost-user-blk使用SPDK的vhost技术，作为块设备和虚拟机的中间件，为QEMU虚拟机提供用户空间和轮询技术，从而加速了IO性能；
  - QEMU块设备：
    - 优点：简单直接，通用性更强，适合快速搭建虚拟机
    - 缺点：性能稍差，更依赖于宿主机的性能
  - vhost-user-blk：
    - 优点：更适合需要高性能存储的专业场景，可以为虚拟机负载提供更高的存储性能
    - 缺点：配置相对复杂，需要更多额外设置并启动多个组件

- **关于基准测试的性能理解。**
  latency和throughput近似成反比。一般情况下，最大吞吐量和带宽成正比，更小的延迟意味着更快的IO速率(IOPS)，进而提高带宽和最大吞吐量，因此成反比。这点从表中的数据也可以看出。

## 致谢
感谢闫星羽同学提供的表格结构以及关于SPDK vhost为QEMU提供IO加速功能的资料$^{[2]}$

## 参考资料/文献
[1][什么是SPDK,以及什么场景需要它](https://zhuanlan.zhihu.com/p/362978954)
[2][vhost Target](https://spdk.io/doc/vhost.html)
[3][io_uring](https://zhuanlan.zhihu.com/p/389978597)

## 附录
bench.conf
```bash{.line-numbers}
[global]
name=benchmark-bdev
filename=/home/tianceli/CloudOS/Lab3/bdev.img
ioengine=io_uring
sqthread_poll=1
direct=1
numjobs=1
runtime=10
thread
time_based
randrepeat=0
norandommap
refill_buffers
group_reporting

[4K-RR-QD1J1]
bs=4k
rw=randread
iodepth=1
stonewall

[4K-RW-QD1J1]
bs=4k
rw=randwrite
iodepth=1
stonewall

[4K-RR-QD32J1]
bs=4k
rw=randread
iodepth=32
stonewall

[4K-RW-QD32J1]
bs=4k
rw=randwrite
iodepth=32
stonewall

[4K-MIX-QD32J1]
bs=4k
rw=randrw
rwmixread=70
iodepth=32
stonewall
```

bench-spdk-bdev.conf
```bash{.line-numbers}
[global]
name=benchmark-spdk-bdev
spdk_json_conf=bdev.json
filename=bdev0
ioengine=/home/tianceli/CloudOS/Lab3/spdk/build/fio/spdk_bdev
direct=1
numjobs=1
runtime=10
thread
time_based
randrepeat=0
norandommap
refill_buffers
group_reporting

[4K-RR-QD1J1]
bs=4k
rw=randread
iodepth=1
stonewall

[4K-RW-QD1J1]
bs=4k
rw=randwrite
iodepth=1
stonewall

[4K-RR-QD32J1]
bs=4k
rw=randread
iodepth=32
stonewall

[4K-RW-QD32J1]
bs=4k
rw=randwrite
iodepth=32
stonewall

[4K-MIX-QD32J1]
bs=4k
rw=randrw
rwmixread=70
iodepth=32
stonewall
```

bdev.json
```javascript{.line-numbers}
{
    "subsystems": [
        {
            "subsystem": "bdev",
            "config": [
                {
                    "params": {
                        "filename": "/home/tianceli/CloudOS/Lab3/bdev.img",
                        "block_size": 512,
                        "name": "bdev0"
                    },
                    "method": "bdev_uring_create"
                }
            ]
        }
    ]
}
```