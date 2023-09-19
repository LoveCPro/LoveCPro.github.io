---
layout: post
title:  netflow采集器源码解析
subtitle:    softflowd源码简解读以及适配
date:   2021-08-05T16:25:52-05:00
author: Dandan
catalog: true
header-img: img/post-bg-debug.png
tags:
    -  openwrt
---
# 前言
前段时间在忙流量采集的功能，用到了netflow。遇到了一系列的问题，今儿给做个总结。
# 源码解读
## 简介
先整理一下用到的工具吧，具体netflow计划放到下一篇。用到的采集工具是softflowd。  

softflowd 是一个网络流量监测工具，用于捕获和导出网络流量的流信息。它通常用于网络性能分析、安全审计和流量监视等用途。softflowd 主要针对流量分析，并将收集到的数据以 NetFlow 或 sFlow 格式导出，以便后续分析和可视化。
以下是 softflowd 的一些关键特点和详细信息：

- **流量捕获**： softflowd 能够捕获网络流量，并将其转化为流记录，以便进行进一步的分析。

- **支持协议**： softflowd 支持 IPv4 和 IPv6 流量的捕获和记录，这使得它非常灵活，并可以适应不同类型的网络。

- **导出格式**： softflowd 可以将捕获的流量数据导出为 NetFlow v5、NetFlow v9 或 sFlow 格式，这些格式通常用于网络流量分析工具。

- **多接口支持**： softflowd 允许您同时监视多个网络接口的流量，这对于大型网络环境非常有用。

- **数据存储**： softflowd 通常不负责数据的长期存储，而是将数据导出到其他流量分析工具或收集器中，例如流量分析软件、SIEM（安全信息与事件管理）系统等。

- **配置灵活性**： softflowd 的配置相对简单，您可以根据需要定制捕获和导出参数，以满足特定的监视需求。

- **轻量级**： softflowd 被设计为一款轻量级的工具，对系统资源的消耗相对较低，因此适用于多种环境。

总的来说，softflowd 是一个强大的工具，用于监视和分析网络流量，有助于网络管理员和安全专业人员更好地了解其网络的活动，以便做出决策和改进网络性能和安全性。
## 流程
源码摘要：
- **初始化**： softflowd 在启动时会进行初始化。它会读取配置文件，设置捕获和导出参数，创建必要的数据结构，然后开始监听指定的网络接口。

- **数据捕获**： softflowd 通过监听网络接口来捕获传入和传出的数据包。这一步通常使用底层的网络套接字操作来实现。

- **数据解析**： 捕获到的数据包会经过解析，softflowd 会识别数据包中的源 IP、目标 IP、源端口、目标端口等信息，并将其关联到相应的流记录中。这些记录通常存储在内存中的数据结构中。

- **流记录处理**： softflowd 会对流记录进行处理，包括更新流的状态、计算流的持续时间、累积流量统计信息等。此过程涉及数据结构的操作和计算。

- **导出数据**： 当流记录达到一定条件（例如时间间隔或流量阈值）或softflowd 收到导出数据的命令时，软件会将流记录导出为 NetFlow 或 sFlow 格式。这通常涉及到将记录编码为二进制格式，并将其发送到指定的目标，例如网络流量分析工具或流量收集器。

- **循环处理**： softflowd 会持续监听网络流量，捕获数据包，更新和处理流记录，并根据配置的导出策略定期导出数据，直到软件被停止或关闭。

- **清理和关闭**： 当softflowd 停止时，它会执行必要的清理工作，包括释放资源、关闭网络接口监听和保存必要的状态信息。  

其中主要的拼凑报文的代码：  
头部结构体：
```c
struct NF5_HEADER {
	u_int16_t version, flows;       //版本，个数
	u_int32_t uptime_ms, time_sec, time_nanosec, flow_sequence; //时间戳
	u_int8_t engine_type, engine_id;    
	u_int16_t sampling_interval;
};
```
pdu结构体：
```c
struct NF5_FLOW {
	u_int32_t src_ip, dest_ip, nexthop_ip;  //源、目的、下一跳ip
	u_int16_t if_index_in, if_index_out;    //出入接口index
	u_int32_t flow_packets, flow_octets;    //包个、大小
	u_int32_t flow_start, flow_finish;
	u_int16_t src_port, dest_port;  //源、目的端口号
	u_int8_t pad1;
	u_int8_t tcp_flags, protocol, tos;  //协议号
	u_int16_t src_as, dest_as;
	u_int8_t src_mask, dst_mask;    //  掩码
	u_int16_t pad2;
};
```
报文的拼接：
```c
	flw->src_ip = flows[i]->addr[1].v4.s_addr;
	flw->dest_ip = flows[i]->addr[0].v4.s_addr;
	flw->src_port = flows[i]->port[1];
	flw->dest_port = flows[i]->port[0];
	flw->flow_packets = htonl(flows[i]->packets[1]);
	flw->flow_octets = htonl(flows[i]->octets[1]);
	flw->flow_start =
	    htonl(timeval_sub_ms(&flows[i]->flow_start,
	    system_boot_time));
	flw->flow_finish =
	    htonl(timeval_sub_ms(&flows[i]->flow_last,
	    system_boot_time));
	flw->tcp_flags = flows[i]->tcp_flags[1];
	flw->protocol = flows[i]->protocol;
	offset += sizeof(*flw);
	j++;
	hdr->flows++;
```
使用时，由于业务环境所需，netflow报文中需要携带其他定制信息，全局信息，直接在结构体`NF5_HEADER`中添加字段，然后在报文拼接时候针对添加的字段赋值就ok。  
当然，如果头部字段添加之后，通过wiregshark解析netflow就不行了。需要自定义解析脚本，下一篇可能会搞一下wireshark解析自定义报文。