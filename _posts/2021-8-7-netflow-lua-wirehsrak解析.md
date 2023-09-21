---
layout: post
title:  wireshark解析器lua脚本
subtitle:    lua脚本解析报文
date:   2021-08-07T16:25:52-05:00
author: Dandan
catalog: true
header-img: img/post-bg-debug.png
tags:
    -  openwrt
---

# 前言
工作中经常会自定义协议，wireshark抓包用已有的协议去解析已不可用。为调试时或者方便测试同事测试时就需要看看报文中的协议数据是否正确，或者记录报文中的数据以验证功能可用性，此时就需要自解析抓到的报文---lua脚本。  
Wireshark 中的 Lua 脚本是一种强大的自定义工具，它允许你扩展 Wireshark 的功能，执行特定任务或分析网络数据包时编写自定义过滤器和协议解析器。以下是关于 Wireshark 中 Lua 脚本的一些重要信息：

- **Lua脚本引擎**：Wireshark 包括一个内置的 Lua 解释器，用于执行 Lua 脚本。这个引擎使用户能够编写脚本以处理捕获的数据包。

- **使用场景**：Lua 脚本可用于执行各种任务，包括创建自定义协议解析器、过滤数据包、提取特定信息、生成统计报告等。

- **协议解析器**：你可以编写 Lua 脚本来解析自定义协议。这对于分析特定应用程序或设备生成的网络流量非常有用。

- **过滤器**：Lua 脚本可以用于创建高级过滤器，以便筛选出感兴趣的数据包。这允许你根据自定义条件过滤捕获的数据。

- **插件管理**：Wireshark 提供了插件管理界面，可用于加载和管理 Lua 插件。这使得添加、删除或禁用脚本变得更加容易

# 使用
## 自定义lua脚本
```lua
-- 自定义解析lua 

-- 协议声明
local netflow_proto = Proto("Dream-Proto","UDP Protocol for Test","Dream UDP Protocol")
-- 协议字段定义
local f_version = ProtoField.uint16("dream.version","Version",base.DEC, {[1] = "DemoMessage"})
local f_count = ProtoField.uint16("dream.count", "count", base.DEC)

local f_sysUptime = ProtoField.uint32("dream.sysUptime","sysUptime", base.DEC)

local f_sequence = ProtoField.uint32("dream.sequence", "sequence", base.DEC)
local f_engine_type = ProtoField.uint8("dream.engine_type", "engine_type", base.DEC)

local f_sip = ProtoField.ipv4("dream.sip", "sip", base.DEC)
local f_dip = ProtoField.ipv4("dream.dip", "dip", base.DEC)
local f_nextip = ProtoField.ipv4("dream.nexthopip", "nexthopip", base.DEC)

local f_inindex = ProtoField.uint16("dream.inindex", "inindex", base.DEC)
local f_outindex = ProtoField.uint16("dream.outindex", "outindex", base.DEC)

local f_pkts = ProtoField.uint32("dream.pkts", "pkts", base.DEC)
local f_bytes = ProtoField.uint32("dream.bytes", "bytes", base.DEC)


local f_sport = ProtoField.uint16("dream.sport", "sport", base.DEC)
local f_dport = ProtoField.uint16("dream.dport", "dport", base.DEC)

local f_pad = ProtoField.uint8("dream.pad", "Padding", base.DEC)
local f_tcpflag = ProtoField.uint8("dream.tcpflag", "TCP Flags", base.DEC)
local f_proto = ProtoField.uint8("dream.proto", "Protocol", base.DEC)
local f_tos = ProtoField.uint8("dream.tos", "Tos", base.HEX)


local f_reserve = ProtoField.uint16("dream.reserve", "Reserve", base.DEC)

-- 协议字段注册
netflow_proto.fields = {f_version,
						f_count,
						f_sysUptime,
						f_sequence,	
						f_engine_type,
						f_sip, f_dip, f_nextip,
						f_inindex, f_outindex, 
						f_pkts, f_bytes, f_firsttime, f_lasttime,
						f_sport, f_dport,
						f_pad, f_tcpflag, f_proto, f_tos,         --8
						f_sas, f_das, 							--16
						f_smask, f_dmask,                       --8
						f_reserve                               --16
						}
 
local arr_version = {
    [1] = "DemoMessage"
}
 

-- inter to ip
function pdutree(pdu_tree, buffer)
	local offsetpdu = 0
	
	pdu_tree:add(f_sip, buffer(offsetpdu, 4))
	offsetpdu = offsetpdu + 4
	
	pdu_tree:add(f_dip, buffer(offsetpdu, 4))
	offsetpdu = offsetpdu + 4
	
	--add other
		
end
 
-- 协议解析逻辑
function netflow_proto.dissector(buffer,pinfo,tree)
    -- 设置Wireshark包列表中Protocol列所对应的该协议名称
    pinfo.cols.protocol:set("Netflow")
 
    local buffer_len = buffer:len()
    --先检查报文长度，太短的不是我的协议
    if buffer_len < 28 then return false end
    -- 在具体包信息创建下新建Dream UDP Message协议项根节点
    local proto_tree = tree:add(netflow_proto, buffer(0, buffer_len), "Netflow UDP Message")
 
    local offset = 0
 
 --   local version = buffer(offset, 2):uint()
    local version = buffer(offset, 2):uint()
    -- 在协议项根节点中增加解析后的MsgType字段
    proto_tree:add(f_version, buffer(offset, 2))
    offset = offset + 2
 
    -- 在协议项根节点中增加解析后的MsgLength字段
    proto_tree:add(f_count, buffer(offset,2))
    offset = offset + 2
 
    local sysUptime = buffer(offset, 4):uint()
    -- 在协议项根节点中增加解析后的sysUptime字段
    proto_tree:add(f_sysUptime, buffer(offset,4))
    offset = offset + 4
	
	local unix_secs = buffer(offset, 4):uint()
    -- 在协议项根节点中增加解析后的unix_secs字段
    proto_tree:add(f_unix_secs, buffer(offset,4))
    offset = offset + 4
	
	local unix_nsecs = buffer(offset, 4):uint()
    -- 在协议项根节点中增加解析后的unix_nsecs字段
    proto_tree:add(f_unix_nsecs, buffer(offset,4))
    offset = offset + 4
	
	local sequence = buffer(offset, 4):uint()
    -- 在协议项根节点中增加解析后的sequence字段
    proto_tree:add(f_sequence, buffer(offset,4))
    offset = offset + 4
	
    -- add other 

	i=1
	while( offset <= buffer_len )
	do
	  
	  local pdu_tree = proto_tree:add(netflow_proto, buffer(offset, 48), "pdu"..i)
	  pdutree(pdu_tree, buffer(offset))
	  i = i+1
	  offset = offset + 48
    end
   
end
 
-- 将协议分析脚本注册至UDP 9995端口

DissectorTable.get("udp.port"):add(9995,netflow_proto)
```
其中的一些属性解析：
- Dissector: 中文直译是解剖器，就是用来解析包的类，为了解析一个新协议，我们需要编写的也是一个Dissector。
  
- DissectorTable: 解析器表是Wireshark中解析器的组织形式，是某一种协议的子解析器的一个列表，其作用是把所有的解析器组织成一种树状结构，便于Wireshark在解析包的时候自动选择对应的解析器。例如TCP协议的子解析器 http, smtp, sip等都被加在了"tcp.port"这个解析器表中，可以根据抓到的包的不同的tcp端口号，自动选择对应的解析器。
  
- 表示协议字段，一般用于解析字段后往解析树上添加节点。根据字段类型不同，其接口可以分为两大类。

## 安装脚本：
- 脚本名：my.lua， 将 my.lua放在 wireshark 根目录下 打开wireshark 目录下的init.lua,在最后一行 替换成： dofile(DATA_DIR..“my.lua”)  
- 打开wireshark, ，分析 —→ 重新载入lua插件即可

就可以看到 将udp 9995的报文解析成自己定义的字段。