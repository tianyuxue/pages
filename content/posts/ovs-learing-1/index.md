---
title: "Open vSwitch学习 - 1 开发环境搭建"
date: "2022-02-17"
cover:
    image: "images/mountain.jpg"
    hidden: false
categories: 
  - "sdn"
  - "ovs"
tags: 
  - "sdn"
  - "ovs"
---
> 本文介绍了OVS的基本功能以及开发环境搭建的过程

<!--more-->
## 1 基本功能

ovs是一个分层的软件交换机，支持vlan、网卡bond、限速、vxlan隧道等功能、支持openflow1.0+协议，提供数据面高性能的转发功能。从部署视图看，进程结构如下：

- ovs-vswitchd 守护进程：
  - 实现了交换机的功能，包含了支持流表转发的内核模块
- ovsdb-server 进程：
  - 保存ovs配置的轻量级数据库
- ovs-dpctl：
  -  配置ovs内核模块的命令行工具
- ovs-vsctl：
  - 查询、修改 ovs-vswitchd配置的命令行工具

- ovs-appctl：
  - 控制 ovs-vswitchd 进程启动、停止的命令行工具
- ovs-ofctl：
  - 查询修改流表的命令行工具

- 除了以上进程，还有几个不常用的工具：

  - ovs-pki 管理系统证书的工具
  - ovs-testcontroller 用于开发，测试环境的使用sdn控制器

  - 支持流表解析的tcpdump工具

## 2 为什么要使用 ovs

虚拟化环境下，Hypervisors 需要二层Bridge功能将同一宿主机上的VM流量转发，目前Linux Bridge功能已经稳定完善，但是对于多宿主机之间VM迁移，网络状态变更支持不够完善，针对这些问题，ovs提供了下列功能：

- 快速响应网络环境变更
- 数据面可以集成专用的硬件芯片，做到线性转发

## 3 开发环境搭建

以Ubuntu20.04为例，从源码编译ovs的过程如下：

1. step1 下载源码
```shell
git clone https://github.com/openvswitch/ovs.git
git checkout v2.7.0
```
2. step2 安装必须的软件
```shell
apt install autoconf libtool 
```
3. step3 编译安装
```shell
./boot.sh
./configure
make
make install
```
4. step4 加载内核模块
```shell
/sbin/modprobe openvswitch
```
5. step5 启动ovs守护进程
```shell
export PATH=$PATH:/usr/local/share/openvswitch/scripts
ovs-ctl start
```
