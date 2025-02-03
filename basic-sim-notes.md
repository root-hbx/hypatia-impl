# basic-sim 文档总结

这一部分对应于 `ns3-sat-sim/simulator/contrib/basic-sim/doc/getting_started.md`

我们会介绍模拟器的内核实现，这与后期自定义新功能的关联度很高 🚀

Step 2 和 Step 4 都会用到这些！

* `basic_simulation_and_run_folder.md` -- Basic concepts of the framework
* `ptop_topology.md` -- Point-to-point topology
* `arbiter_routing.md` -- A new type of routing with more flexibility
* `link_utilization_tracking.md` -- Link utilization tracking
* `link_queue_tracking.md` -- Link queue tracking
* `flows_application.md` -- Flow application ("send from A to B a flow of size X at time T")
* `udp_burst_application.md` -- UDP burst application ("send from A to B at a rate of X Mbit/s at time T for duration D")
* `pingmesh_application.md` -- Ping application ("send from A to B a ping at an interval I")
* `tcp_optimizer.md` -- Optimize certain TCP parameters
* `future_work.md` -- To find out what can be extended / improved

## Getting Started

这份文档描述的是如何使用 ns-3 进行一个简单的网络仿真。

### 1. 目标

仿真目标是模拟一个交换机（Switch），下连接有3个服务器。这个交换机通常被称为 ToR（Top of Rack，机架顶部交换机）。每条链路的延迟为 10 微秒，链路速率为 100 Mbit/s，每个链路接口有一个 100 包的 FIFO 队列。仿真持续时间是 4 秒。

仿真拓扑如下：

```
           Switch (id: 0)
         /     |          \
       /       |           \
     /         |            \
 Server (1)  Server (2)  Server (3)
```

在这个网络中，我们将安装以下三种不同的应用：

1. **三条 TCP 流：**
   - 从服务器 1 到服务器 3 发送 100 Mbit 的数据（开始时间：T=0）
   - 从服务器 2 到服务器 3 发送 100 Mbit 的数据（开始时间：T=500ms）
   - 从服务器 2 到服务器 3 发送 100 Mbit 的数据（开始时间：T=500ms）

2. **两条 UDP 爆发：**
   - 从服务器 1 到服务器 2 以 50 Mbit/s 的速率发送数据（开始时间：T=0，持续 5 秒）
   - 从服务器 2 到服务器 1 以 50 Mbit/s 的速率发送数据（开始时间：T=0，持续 5 秒）

3. **Ping 测试：** 在所有服务器之间发送 Ping 请求，间隔为 10 毫秒。

### 2. 期望结果

对每个应用和流量的预期行为是：

- **TCP 流：**
  - F0 从服务器 1 到 3 在前 500 毫秒将会得到 50 Mbit/s 的带宽，之后和 F1/F2 竞争带宽，最终完成。
  - F1 和 F2 将共享部分带宽，F1 和 F2 的速率大约为 20-30 Mbit/s。
- **UDP 流：**
  - UDP 流 B0 和 B1 的数据不会受到 TCP 流的影响，它们能稳定传输数据。
- **Ping 测试：** 由于队列的积压，Ping 延迟会略微增加。

### 3. 实验步骤

按照以下步骤进行实验：

#### 3.1 创建文件

首先，在任意地方创建一个目录 `example_run`，然后在该目录下创建四个文件：

- **config_ns3.properties：** 配置仿真参数的文件，包含仿真结束时间、随机种子、拓扑文件、调度配置等。
- **topology.properties：** 描述拓扑结构的文件，包含节点和链路的设置。
- **tcp_flow_schedule.csv：** 描述 TCP 流的调度文件，指定了每个 TCP 流的源、目标、大小、起始时间等信息。
- **udp_burst_schedule.csv：** 描述 UDP 爆发流的调度文件，指定了 UDP 流的源、目标、速率、持续时间等信息。

#### 3.2 创建文件结构

你的目录结构应如下所示：

```
example_run
|-- config_ns3.properties
|-- topology.properties
|-- tcp_flow_schedule.csv
|-- udp_burst_schedule.csv
```

#### 3.3 编写主程序

在 `$ns-3` 的 `scratch/` 文件夹中创建一个名为 `my_main.cc` 的文件，文件内容为仿真程序代码。此代码将：

- 加载仿真环境
- 读取拓扑文件并安装路由器和调度器
- 安装 TCP、UDP 和 Ping 流的调度器
- 执行仿真并输出结果

#### 3.4 运行仿真

在 `$ns-3/simulator/` 目录下运行以下命令来启动仿真：

```
./waf --run="my_main --run_dir='/your/path/to/example_run'"
```

如果你想保存控制台输出，可以使用：

```
mkdir -p /your/path/to/example_run/logs_ns3
./waf --run="my_main --run_dir='/your/path/to/example_run'" 2>&1 | tee /your/path/to/example_run/logs_ns3/console.txt
```

#### 3.5 查看仿真结果

仿真结束后，在 `/your/path/to/example_run/logs_ns3` 目录下会生成一些结果文件，内容包括：

- **finished.txt**：仿真完成标志。
- **timing_results.csv** 和 **timing_results.txt**：时间统计结果。
- **tcp_flows.csv** 和 **tcp_flows.txt**：TCP 流的详细信息。
- **udp_bursts_incoming.csv** 和 **udp_bursts_outgoing.csv**：UDP 流的收发统计。
- **pingmesh.csv** 和 **pingmesh.txt**：Ping 测试的统计数据。
- **link_utilization.csv**：链路利用率。
- **link_utilization_summary.txt**：链路利用率汇总。

例如，`tcp_flows.txt` 文件会包含每个 TCP 流的源、目标、大小、起始时间、结束时间、发送进度、平均速率等信息。

## Point-to-point Topology

**点对点拓扑**提供了一种快速创建由点对点链路连接的节点网络的方法。它通常用于连接多个节点，其中每一对节点通过一条链路直接相连。

这份文档中提到的文件包括:

- **`model/core/topology.h`**：基础拓扑类。
- **`model/core/ptop-topology.cc/h`**：点对点拓扑类。

### 1.1 配置文件

首先，你需要在仿真文件夹的 `config_ns3.properties` 文件中添加以下内容：

```
topology_ptop_filename="topology.properties"
```

这行配置告诉仿真程序使用 `topology.properties` 这个文件来定义拓扑。

### 1.2 创建拓扑文件

在仿真文件夹中创建一个名为 `topology.properties` 的文件。

这个文件包含了拓扑的布局和链路的属性（如延迟、速率、队列大小、qdisc）。它独立于仿真文件，可以根据需要修改。 

```
num_nodes=4
num_undirected_edges=3
switches=set(0)
switches_which_are_tors=set(0)
servers=set(1,2,3)
undirected_edges=set(0-1,0-2,0-3)

link_channel_delay_ns=10000
link_device_data_rate_megabit_per_s=100.0
link_device_queue=drop_tail(100p)
link_interface_traffic_control_qdisc=disabled
```

具体来说:

- **num_nodes=4**：表示网络中有 4 个节点。
- **num_undirected_edges=3**：表示有 3 条无向边（每条无向边都将转化为两条链路）。
- **switches=set(0)**：表示节点 0 是一个交换机（Switch）。
- **switches_which_are_tors=set(0)**：表示节点 0 也是一个 ToR（Top of Rack，机架顶部交换机）。
- **servers=set(1,2,3)**：表示节点 1、2 和 3 是服务器。
- **undirected_edges=set(0-1,0-2,0-3)**：表示节点 0 分别与节点 1、2 和 3 通过无向边连接。
- **link_channel_delay_ns=10000**：表示链路的传播延迟是 10000 纳秒（即 10 微秒）。
- **link_device_data_rate_megabit_per_s=100.0**：表示链路的数据传输速率为 100 Mbit/s。
- **link_device_queue=drop_tail(100p)**：表示链路的队列实现使用 DropTail（丢弃尾部的包），最大队列长度为 100 包。
- **link_interface_traffic_control_qdisc=disabled**：表示禁用流量控制队列（qdisc）。

### 1.3 在代码中使用点对点拓扑

接下来，你需要在你的仿真代码中导入点对点拓扑类，代码如下：

```c++
#include "ns3/ptop-topology.h"
// 可能还需要导入：
#include "ns3/ipv4-arbiter-routing-helper.h"
#include "ns3/arbiter-ecmp-helper.h"
```

### 1.4 创建拓扑对象

在仿真开始前，你需要在代码中创建一个点对点拓扑对象：

```c++
Ptr<TopologyPtop> topology = CreateObject<TopologyPtop>(basicSimulation, Ipv4ArbiterRoutingHelper());
   
// 安装路由器（例如使用 ECMP 路由器）
ArbiterEcmpHelper::InstallArbiters(basicSimulation, topology);
```

- **CreateObject<TopologyPtop>**：创建一个点对点拓扑对象，`basicSimulation` 是基本仿真对象，`Ipv4ArbiterRoutingHelper()` 是 IPv4 路由帮助类。
- **ArbiterEcmpHelper::InstallArbiters**：安装 ECMP（等成本多路径）路由器。

如果你不使用 ECMP 路由器，还可以使用传统的路由方法，但性能较差，因为传统方法使用线性查找，适合简单拓扑，而不适合点对点拓扑。

### 1.5 获取拓扑中的节点

你可以通过以下方式获取拓扑中的所有节点：

```c++
NodeContainer nodes = topology->GetNodes(); // 注意：传递的是引用
```

## Arbiter Routing

这份文档介绍了在 ns-3 中使用 **Arbiter 路由**，一种灵活的路由方法，允许你实现任意的路由策略。

### 1. Arbiter 路由

在传统的 ns-3 路由类中，路由策略被限制为特定的路由表结构，而 **Arbiter 路由** 提供了一种更加灵活的路由方式，允许你自定义和实现任何路由策略。**Arbiter** 类是所有自定义路由类的基类，它为你提供了实现路由决策的框架。

### 2. 主要文件介绍

以下是文档中提到的几个核心文件，它们实现了不同的 Arbiter 路由功能：

- **Arbiter**: `model/core/arbiter.c/h`
    - 这是一个抽象的超级类，它要求任何继承它的类必须实现一个名为 `decide` 的函数，来决定如何处理一个数据包。
    - 这个函数会根据源节点、目标节点、数据包、IP 头等信息，决定是否丢弃包或将包转发到哪个接口，目标网关的 IP 地址是多少。
    - 函数返回一个元组：`bool failed, uint32_t out_if_idx, uint32_t gateway_ip_address`。
- **ArbiterPtop**: `model/core/arbiter-ptop.c/h`
    - 这是一个扩展了 `Arbiter` 类的类，专门针对点对点拓扑（point-to-point topology）。
    - 它把 `decide` 函数改为 `topology_decide`，该函数不需要网关，只返回下一跳的节点 ID，或者返回 `-1` 表示丢弃该包。

------

- **ArbiterEcmp**: `model/core/arbiter-ecmp.c/h`
    - 这是继承自 `ArbiterPtop` 的一个类，它根据 5 元组哈希值来决定路由，并从一组候选的下一跳中选择一个作为最终的路由。
- **ArbiterEcmpHelper**: `model/arbiter-ecmp-helper.c/h`
    - 这是一个辅助类，用于计算 `ArbiterEcmp` 实例的路由状态，并将该路由状态安装到它们上。

------

- **Ipv4ArbiterRouting**: `model/core/ipv4-arbiter-routing.c/h`
    -  这是一个路由实例类，它会在每次路由决策时调用对应的 `Arbiter` 实例，来决定每个数据包的路由路径。
- **Ipv4ArbiterRoutingHelper**: `helper/core/ipv4-arbiter-routing-helper.c/h`
    - 这是一个帮助类，用于将 `Ipv4ArbiterRouting` 实例安装到节点上。

### 3. 如何使用这些助手类

#### 3.1 导入助手类

在你的仿真代码中，首先需要导入与 **Arbiter 路由** 相关的头文件：

```cpp
#include "ns3/ipv4-arbiter-routing-helper.h"
#include "ns3/arbiter-ecmp-helper.h"
```

#### 3.2 安装 ECMP 路由

然后，在仿真开始之前，你可以在代码中安装 ECMP 路由。具体方法是创建点对点拓扑并安装路由器 arbiters：

```cpp
// 创建点对点拓扑，并安装路由 arbiters
Ptr<TopologyPtop> topology = CreateObject<TopologyPtop>(basicSimulation);
ArbiterEcmpHelper::InstallArbiters(basicSimulation, topology);
```

- **CreateObject<TopologyPtop>**：创建一个点对点拓扑对象。
- **ArbiterEcmpHelper::InstallArbiters**：使用 `ArbiterEcmpHelper` 安装 ECMP 路由 arbiters。

### 4. **如何创建你自己的 Arbiter**

如果你希望实现一个自定义的 **Arbiter 路由**，你可以创建一个继承自 `Arbiter` 类（对于非点对点拓扑）或 `ArbiterPtop` 类（对于点对点拓扑）的类。核心的实现就是覆盖一个函数。

因此，在你自己实现的继承类中，需要覆盖的函数是：

- 对于点对点拓扑，覆盖 `topology_decide` 函数。
- 对于其他类型的拓扑，覆盖 `decide` 函数。

这两个函数的功能是：根据源节点、目标节点、数据包和其他信息，决定数据包是否丢弃，或者将数据包转发到哪个接口。

如果你需要计算路由状态，可以参考 **ArbiterEcmpHelper** 类的实现，它展示了如何计算和安装路由状态。

## Link Utilization Tracking

在 ns-3 中进行 **链路利用率跟踪（Link Utilization Tracking）**，可以用来跟踪点对点网络设备（即链路）的利用情况。

### 1. 链路利用率跟踪概述

链路利用率跟踪主要用于监控和记录网络链路的利用情况。通过该功能，你可以实时地看到每条链路在一定时间间隔内的忙碌时间以及利用率。

文档中涉及的文件有：

- **`ptop-link-utilization-tracker.cc/h`**：链路利用率跟踪器的实现。
- **`ptop-link-utilization-tracker-helper.cc/h`**：用于在拓扑中的网络设备上安装链路利用率跟踪器的辅助类。

事实上，笔者更推荐使用helper类，这跟其他安装设备的方式是统一的！

### 2. 使用helper进行安装

#### 2.1 配置文件

首先，在你仿真目录下的 `config_ns3.properties` 配置文件中添加以下内容：

```
enable_link_utilization_tracking=true
link_utilization_tracking_interval_ns=100000000
```

- **`enable_link_utilization_tracking=true`**：启用链路利用率跟踪功能。
- **`link_utilization_tracking_interval_ns=100000000`**：设置跟踪间隔时间（单位为纳秒）。
    - `100000000` 纳秒等于 100 毫秒，表示每 100 毫秒进行一次利用率统计。

要启用链路利用率跟踪，必须在 `config_ns3.properties` 中设置以下两个选项：

- **`enable_link_utilization_tracking`**：启用链路利用率跟踪，设置为 `true`。
- **`link_utilization_tracking_interval_ns`**：设置链路利用率跟踪的时间间隔（单位是纳秒）。
    - 这个值决定了你每隔多长时间统计一次链路的利用率。
    - 间隔过小会导致性能开销过大，间隔过大则会导致结果不够精细。

#### 2.2 导入助手类

在你的仿真代码中导入链路利用率跟踪的辅助类：

```cpp
#include "ns3/ptop-link-utilization-tracker-helper.h"
```

#### 2.3 安装链路利用率跟踪器

在仿真开始之前，你需要在代码中创建链路利用率跟踪器并将其安装到拓扑中的网络设备上：

```cpp
// 安装链路利用率跟踪器
PtopLinkUtilizationTrackerHelper linkUtilizationTrackerHelper = PtopLinkUtilizationTrackerHelper(basicSimulation, topology);
```

这行代码会在你的拓扑中的 __所有链路__ 上安装利用率跟踪器。

#### 2.4 链路利用率结果

仿真运行结束后，你可以使用以下代码将链路利用率的结果写入文件：

```cpp
// 写入链路利用率结果
linkUtilizationTrackerHelper.WriteResults();
```

运行结束后，仿真结果将保存在 `logs_ns3` 文件夹中。

你将在运行文件夹中的 `logs_ns3` 文件夹内看到链路利用率的日志文件。

### 3. 直接安装

如果你不使用助手类，而是希望直接使用链路利用率跟踪器（笔者不推荐），按照以下步骤进行：

#### 3.1 导入链路利用率跟踪器

在你的代码中导入链路利用率跟踪器：

```cpp
#include "ns3/ptop-link-utilization-tracker.h"
```

#### 3.2 创建链路利用率跟踪器

在仿真开始之前，你需要获取某个网络设备，并为该设备创建链路利用率跟踪器：

```cpp
Ptr<PointToPointNetDevice> networkDevice = ... // 从某处获取网络设备
int64_t utilization_interval_ns = 100000000; // 100毫秒
Ptr<PtopLinkUtilizationTracker> tracker = CreateObject<PtopLinkUtilizationTracker>(networkDevice, utilization_interval_ns);
// ... 存储 tracker 以保持其生命期，稍后可以获取其结果
```

#### 3.3 获取利用率结果

仿真结束后，你可以通过以下代码获取并处理链路利用率数据：

```cpp
const std::vector<std::tuple<int64_t, int64_t, int64_t>> intervals = tracker->FinalizeUtilization();
for (size_t j = 0; j < intervals.size(); j++) {
    int64_t interval_start_ns = std::get<0>(intervals[j]);
    int64_t interval_end_ns = std::get<1>(intervals[j]);
    int64_t interval_busy_time_ns = std::get<2>(intervals[j]);
    double interval_utilization = ((double) interval_busy_time_ns) / (double) (interval_end_ns - interval_start_ns);
    // ... 然后做一些处理，比如打印结果
}
```

这段代码会按时间间隔获取链路的利用率信息，并计算每个时间间隔内的利用率。

### 4. 链路利用率日志文件

在仿真运行后，你将会在 `logs_ns3` 文件夹中看到以下几个日志文件：

- **`link_utilization.csv`**：以 CSV 格式输出的链路利用率数据，按时间间隔记录。每一行的格式如下：
    ```
    <from>,<to>,<interval start (ns)>,<interval end (ns)>,<amount of busy in this interval (ns)>
    ```
    这表示从源节点到目标节点，在某个时间间隔内链路的忙碌时间。
- **`link_utilization_compressed.csv`**：类似于 `link_utilization.csv`，但会将相邻利用率大致相同的时间间隔合并，方便绘图。
- **`link_utilization_compressed.txt`**：人类可读的表格，显示了每个时间间隔的链路利用率，并将相邻利用率相同的间隔压缩在一起。
- **`link_utilization_summary.txt`**：每条链路的平均利用率，格式也是人类可读的表格。

## Link Queue Tracking

跟上面的`Link Utilization Tracking`几乎是一模一样，原理相同，调用形式也是极为相似！

笔者不做解释，建议直接看原文档 [link_qtrack doc](ns3-sat-sim/simulator/contrib/basic-sim/doc/link_queue_tracking.md)

## TCP Flow Application

> ns3-sat-sim/simulator/contrib/basic-sim/doc/tcp_flow_application.md

这里介绍了如何使用 **TCP 流应用（TCP Flow Application）**，它是一个用于模拟 TCP 流量的简单应用程序。

其主要功能是安排数据流的传输，从节点 A 向节点 B 传输一定量的数据，并将流的完成情况保存到文件中。以下是详细的解释：

### 1. TCP 流应用概述

**TCP 流应用** 是一种最简单的网络应用，用于模拟 TCP 流的传输。你可以通过设置流的大小和开始时间，安排从源节点 A 到目标节点 B 的数据传输。应用会记录流的完成情况，并将这些信息保存到文件中。

文档中提到的文件有：

- `tcp-flow-send-application.cc/h`：该应用打开一个 TCP 连接，并单向地通过该连接发送数据
- `tcp-flow-sink.cc/h`：该应用接受传入的 TCP 流并确认收到的数据，但不会回复数据
- `tcp-flow-send-helper.cc/h`：帮助安装流发送应用的助手类
- `tcp-flow-sink-helper.cc/h`：帮助安装流接收应用的助手类
- `tcp-flow-scheduler.cc/h`：读取流的调度计划，安排这些流在仿真中的开始时间，并在所有节点上安装流接收器。仿真结束后，可以将流的结果写入文件
- `tcp-flow-schedule-reader.cc/h`：用于从文件中读取流的调度计划

### 2. 开始使用：流调度器

**流调度器** 是一个用于管理多个 TCP 流的工具，可以根据指定的调度文件，在指定的时间启动流。

使用流调度器可以方便地模拟多个流并记录其执行过程!

#### 2.1 配置文件

首先，你需要在仿真文件夹中的 `config_ns3.properties` 配置文件中添加以下内容：

```
enable_tcp_flow_scheduler=true
tcp_flow_schedule_filename="tcp_flow_schedule.csv"
tcp_flow_enable_logging_for_tcp_flow_ids=set(0,1)
```

- `enable_tcp_flow_scheduler=true`：启用 TCP 流调度器
- `tcp_flow_schedule_filename="tcp_flow_schedule.csv"`：指定包含流调度信息的文件路径（例如，`tcp_flow_schedule.csv`）
- `tcp_flow_enable_logging_for_tcp_flow_ids=set(0,1)`：指定要记录日志的流 ID
    - 例如此处表示，记录流 0 和流 1 的日志
    - 日志会包括进度、拥塞窗口大小（cwnd）和往返时间（RTT）等信息

#### 2.2 流调度文件

创建一个名为 `tcp_flow_schedule.csv` 的文件，并将其放入仿真文件夹中，内容如下：

```
0,0,1,10000,0,,
1,0,1,3000,10000,,
2,0,1,34000,30000,,
```

每一行定义一个流，格式如下：

```
[tcp_flow_id],[from_node_id],[to_node_id],[size_byte],[start_time_ns],[additional_parameters],[metadata]
```

- `tcp_flow_id`：流的 ID，必须是递增的
- `from_node_id` 和 `to_node_id`：源节点和目标节点的 ID
- `size_byte`：流的大小（字节）
- `start_time_ns`：流开始的时间（单位：纳秒）
- `additional_parameters`：可以设置额外的参数（如不同的传输协议、优先级等）
- `metadata`：用于在后续日志中标识流的其他信息（如工作负载或协同流等）

#### 2.3 导入流调度器

在你的代码中导入 TCP 流调度器的头文件：

```cpp
#include "ns3/tcp-flow-scheduler.h"
```

#### 2.4 安装流调度器

在仿真开始之前，创建 TCP 流调度器并安装它：

```cpp
// 安装 TCP 流调度器
TcpFlowScheduler tcpFlowScheduler(basicSimulation, topology); // 需要启用 enable_tcp_flow_scheduler=true
```

#### 2.5 写入结果

仿真结束后，你可以将流的结果写入日志文件：

```cpp
// 写入 TCP 流结果
tcpFlowScheduler.WriteResults();
```

仿真结束后，流的日志文件将保存在 `logs_ns3` 文件夹中。

### 3. 开始使用：直接在应用上流调度

如果你不使用流调度器，而是希望手动为每个节点安装发送和接收应用，可以按照以下步骤进行：

#### 3.1 导入发送和接收应用的助手类

在代码中导入用于发送和接收流的助手类：

```cpp
#include "ns3/tcp-flow-send-helper.h"
#include "ns3/tcp-flow-sink-helper.h"
```

#### 3.2 安装接收应用

在目标节点（例如节点 B）上安装接收应用：

```cpp
// 在目标节点安装接收器
TcpFlowSinkHelper sink("ns3::TcpSocketFactory", InetSocketAddress(Ipv4Address::GetAny(), 1024));
ApplicationContainer app = sink.Install(node_b);
app.Start(Seconds(0.0));
```

#### 3.3 安装发送应用

在源节点（例如节点 A）上安装发送应用：

```cpp
// 在源节点安装发送应用
TcpFlowSendHelper source(
        "ns3::TcpSocketFactory",
        InetSocketAddress(node_b->GetObject<Ipv4>()->GetAddress(1,0).GetLocal(), 1024),
        1000000000, // 流大小（字节）
        0, // 流 ID（必须唯一）
        true, // 启用跟踪 cwnd / rtt / 进度
        m_basicSimulation->GetLogsDir() // 日志目录
);
ApplicationContainer app_flow_0 = source.Install(node_a);
app.Start(NanoSeconds(0)); // 流开始时间（纳秒）
```

#### 3.4 获取流状态

仿真结束后，你可以获取每个流的状态信息：

```cpp
Ptr<TcpFlowSendApplication> tcpFlowSendApp = app_flow_0->GetObject<TcpFlowSendApplication>();

// 获取流的状态
bool is_completed = tcpFlowSendApp->IsCompleted(); // 流发送完了没
bool is_conn_failed = tcpFlowSendApp->IsConnFailed(); // 连接有失败吗
bool is_closed_err = tcpFlowSendApp->IsClosedByError(); // 是因错误暂停的吗
bool is_closed_normal = tcpFlowSendApp->IsClosedNormally(); // 是正常关闭的吗
int64_t sent_byte = tcpFlowSendApp->GetAckedBytes();
int64_t fct_ns; // flow complete time 流完成时间，常表示拥塞程度，单位是ns
if (is_completed) {
    fct_ns = tcpFlowSendApp->GetCompletionTimeNs() - entry.start_time_ns;
} else {
    fct_ns = m_simulation_end_time_ns - entry.start_time_ns;
}
std::string finished_state;
if (is_completed) {
    finished_state = "YES";
} else if (is_conn_failed) {
    finished_state = "NO_CONN_FAIL";
} else if (is_closed_normal) {
    finished_state = "NO_BAD_CLOSE";
} else if (is_closed_err) {
    finished_state = "NO_ERR_CLOSE";
} else {
    finished_state = "NO_ONGOING";
}

// ... 根据这些信息进行处理
```

### 4. 流日志文件

仿真结束后，你将在 `logs_ns3` 文件夹中看到以下日志文件：

- **`tcp_flows.txt`**：以人类可读格式显示的流结果。
- **`tcp_flows.csv`**：以 CSV 格式显示的流结果，格式如下：
    ```
    tcp_flow_id,from_node_id,to_node_id,size_byte,start_time_ns,end_time_ns,duration_ns,amount_sent_byte,[finished: YES/CONN_FAIL/NO_BAD_CLOSE/NO_ERR_CLOSE/NO_ONGOING],metadata
    ```

`finished` 字段可能的值：

- `YES`：数据成功传输并正常关闭
- `NO_CONN_FAIL`：连接无法建立
- `NO_BAD_CLOSE`：套接字提前关闭，数据未完全传输
- `NO_ERR_CLOSE`：套接字由于错误关闭
- `NO_ONGOING`：流仍在进行中

这一部分量比较大，小结一下:

- **TCP 流应用** 用于模拟 TCP 流的传输，你可以通过调度和日志记录来跟踪流的进度。
- 使用 **流调度器** 可以简化多个流的管理，而 **手动安装应用** 可以让你更细粒度地控制每个流。
- 仿真结束后，所有流的结果将保存在日志文件中，可以方便地进行分析。

## UDP Burst Application

> ns3-sat-sim/simulator/contrib/basic-sim/doc/udp_burst_application.md

这份文档介绍了 **UDP burst 应用**，它用于模拟 UDP 流量的爆发传输，通常用于测试网络中 UDP 数据包的高带宽传输：

### 1. UDP Burst 应用概述

**UDP burst 应用** 是一个简单的应用，它根据调度，在指定的时间从节点 A 向节点 B 发送一定速率（以 `Mbit/s` 为单位）的 UDP 数据包，持续一定时间。每次 UDP burst 都会记录结果，保存到文件中。

文档中提到的文件包括：

- `udp-burst-application.cc/h`：该应用程序既作为客户端，也作为服务器
    - 每个节点上都会安装一个应用实例，用于管理出站和入站的 UDP burst
- `udp-burst-info.cc/h`：提供有关 UDP burst 的信息
- `id-seq-header.cc/h`：为 UDP burst 数据包添加头部，以便跟踪每个数据包属于哪个 burst 以及它在序列中的位置
- `udp-burst-helper.cc/h`：帮助安装 UDP burst 应用程序的助手类
- `udp-burst-scheduler-reader.cc/h`：读取 UDP burst 调度的文件
- `udp-burst-scheduler.cc/h`：读取和安排 UDP burst 的调度文件，并在所有相关节点上安装 UDP burst 应用。在仿真结束时，它可以检索并写入结果

你可以单独使用这些应用 (1)，也可以使用 **UDP burst 调度器 (2)**，后者是推荐的方式。

### 2. 开始使用：UDP burst 调度器

**UDP burst 调度器** 是用于管理多个 UDP burst 任务的工具。它可以读取调度文件，并安排多个 burst 在指定的时间开始。

#### 2.1 配置文件

首先，你需要在仿真文件夹中的 `config_ns3.properties` 配置文件中添加以下内容，配置要跟踪的 2 个 UDP burst：

```
enable_udp_burst_scheduler=true
udp_burst_schedule_filename="udp_burst_schedule.csv"
udp_burst_enable_logging_for_udp_burst_ids=set(0,1)
```

- **`enable_udp_burst_scheduler=true`**：启用 UDP burst 调度器
- **`udp_burst_schedule_filename="udp_burst_schedule.csv"`**：指定 UDP burst 调度文件的路径（例如，`udp_burst_schedule.csv`）
- **`udp_burst_enable_logging_for_udp_burst_ids=set(0,1)`**：指定要记录日志的 UDP burst ID
    - 在这个例子中，记录的是 burst 0 和 burst 1 的日志

#### 2.2 UDP burst 调度文件

创建一个名为 `udp_burst_schedule.csv` 的文件，并将其放入仿真文件夹中，内容如下：

```
0,1,2,50,0,5000000000,,
1,2,1,50,0,5000000000,,
```

文件中的每一行定义了一个 UDP burst，格式如下：

```
[udp_burst_id],[from_node_id],[to_node_id],[target_rate_megabit_per_s],[start_time_ns],[duration_ns],[additional_parameters],[metadata]
```

- `udp_burst_id`：UDP burst 的 ID，必须递增
- `from_node_id` 和 `to_node_id`：源节点和目标节点的 ID
- `target_rate_megabit_per_s`：目标传输速率（单位：Mbit/s）
- `start_time_ns`：UDP burst 开始的时间（单位：纳秒）
- `duration_ns`：UDP burst 持续的时间（单位：纳秒）
- `additional_parameters`：用于配置每个 burst 的附加参数（可选）
- `metadata`：用于标识流的额外信息（可选）

#### 2.3 导入 UDP burst 调度器

在代码中导入 UDP burst 调度器：

```cpp
#include "ns3/udp-burst-scheduler.h"
```

#### 2,4 安装 UDP burst 调度器

在仿真开始之前，创建 UDP burst 调度器并安装它：

```cpp
// 安装 UDP burst 调度器
UdpBurstScheduler udpBurstScheduler(basicSimulation, topology); // 需要启用 enable_udp_burst_scheduler=true
```

#### 2.5 写入结果

仿真结束后，你可以将 UDP burst 的结果写入日志文件：

```cpp
// 写入 UDP burst 结果
udpBurstScheduler.WriteResults();
```

仿真结束后，UDP burst 的日志文件将保存在 `logs_ns3` 文件夹中。

### 3. 开始使用：直接安装应用

如果你不使用调度器，而是希望手动为 __每个节点__ 安装 UDP burst 应用，你可以按照以下步骤进行：

#### 3.1 导入 UDP burst 助手类

在代码中导入 UDP burst 助手类：

```cpp
#include "ns3/udp-burst-helper.h"
```

#### 3.2 安装 UDP burst 应用

在每个节点上安装 UDP burst 应用。下面的代码示例设置了节点 A 和节点 B 的应用：

```cpp
// 在节点 A 和 B 上设置 UDP burst 应用
UdpBurstHelper udpBurstHelper(1026, m_basicSimulation->GetLogsDir());
ApplicationContainer app_a = udpBurstHelper.Install(node_a);
app_a.Start(Seconds(0.0));
ApplicationContainer app_b = udpBurstHelper.Install(node_b);
app_b.Start(Seconds(0.0));
```

#### 3.3 注册 UDP burst

在 *节点 A* 和 *节点 B* 上注册 UDP burst 的出站和入站：

```cpp
// 从节点 A 向节点 B 启动一个 burst,这是对burst的参数定义
UdpBurstInfo info = UdpBurstInfo(0, 6, 23, 50, 0, 1000000000, "", "");

// 注册节点 A 上的出站 burst
app_a.Get(0)->RegisterOutgoingBurst(
            info,
            InetSocketAddress(node_b->GetObject<Ipv4>()->GetAddress(1,0).GetLocal(), 1026),
            true   
);

// 注册节点 B 上的入站 burst
app_b.Get(0)->RegisterIncomingBurst(info, true);
```

#### 3.4 获取 UDP burst 状态

仿真结束后，你可以获取每个节点的 UDP burst 发送和接收状态：

```cpp
// 获取节点 A 发送的 UDP burst 信息
std::vector<std::tuple<UdpBurstInfo, uint64_t>> outgoing_bursts = app_a.Get(0)->GetOutgoingBurstsInformation();

// 获取节点 B 接收的 UDP burst 信息
std::vector<std::tuple<UdpBurstInfo, uint64_t>> incoming_bursts = app_b.Get(0)->GetIncomingBurstsInformation();
```

### 4. UDP burst 日志文件

仿真结束后，你将在 `logs_ns3` 文件夹中看到以下几个日志文件：

- **`udp_bursts_incoming.txt` 和 `udp_bursts_outgoing.txt`**：UDP burst 的结果，以人类可读的表格显示。
- **`udp_bursts_outgoing.csv`**：UDP 出站 burst 的 CSV 格式结果，包含每个 burst 的发送速率、数据包发送数量等信息。
- **`udp_bursts_incoming.csv`**：UDP 入站 burst 的 CSV 格式结果，包含每个 burst 的接收速率、数据包接收数量等信息。

这一部分跟上面的TCP基本一模一样，再做些总结:

- **UDP burst 应用** 用于模拟从一个节点到另一个节点的 UDP 数据流。
- 你可以选择 **UDP burst 调度器** 来集中管理多个 UDP burst，也可以手动为每个节点安装应用。
- 仿真结束后，所有 UDP burst 的结果将保存到日志文件中，便于进一步分析。

你会发现设计架构与原理都是相通的！

## Pingmesh Application

> ns3-sat-sim/simulator/contrib/basic-sim/doc/pingmesh_application.md

Pingmesh是一个基于ns-3网络模拟器的测量工具，通过持续发送UDP ping包来测量网络端点间的往返时间（RTT）。它包含两种使用模式：

1. **推荐模式**：通过调度器自动管理全流程
2. **手动模式**：直接安装客户端/服务器应用

关键测量指标包括：单向延迟（`latency_to_there_ns`/`latency_from_there_ns`）、往返时间（`rtt_ns`）和数据包丢失状态（`YES`/`LOST`）。

### 1. 核心代码文件

```
model/apps/
├── udp-rtt-client.cc/h  # UDP客户端：发送ping包+接收响应
├── udp-rtt-server.cc/h  # UDP服务端：接收ping包+立即回复
helper/apps/
├── udp-rtt-helper.cc/h    # 安装客户端/服务的辅助工具
└── pingmesh-scheduler.cc/h  # 自动化调度器（推荐使用）
```

### 2. 推荐模式：使用调度器

#### 2.1 配置文件设置

在`config_ns3.properties`中添加:

```properties
enable_pingmesh_scheduler=true      # 启用调度器
pingmesh_interval_ns=100000000      # 每100ms发送一次ping
pingmesh_endpoint_pairs=all          # 监控所有端点对
```

可选参数示例, `set(0->1, 5->6)`表示仅监控 0→1 和 5→6 的端点对:

```
pingmesh_endpoint_pairs=set(0->1, 5->6)
```

#### 2.2 代码集成

```cpp
// 在仿真开始前添加
#include "ns3/pingmesh-scheduler.h"
PingmeshScheduler pingmeshScheduler(basicSimulation, topology);

// 在仿真结束后添加
pingmeshScheduler.WriteResults();  // 输出测量结果
```

### 3. 手动模式：直接安装应用

不是特别推荐，但是它的自由度非常高，在后期做大型自定义项目肯定用得上！

#### 3.1 安装服务端（节点B）

```cpp
UdpRttServerHelper echoServerHelper(1025);  // 监听1025端口
ApplicationContainer app = echoServerHelper.Install(node_b);
app.Start(Seconds(0.0));  // 立即启动
```

#### 3.2 安装客户端（节点A）

```cpp
UdpRttClientHelper source(
    node_b_ip,    // 目标IP地址
    1025,         // 目标端口
    node_a_id,    // 源节点ID
    node_b_id     // 目标节点ID
);
source.SetAttribute("Interval", TimeValue(NanoSeconds(100000000))); // 100ms间隔
ApplicationContainer app_a_to_b = source.Install(node_a);
app_a_to_b.Start(NanoSeconds(start_time));  // 指定启动时间
```

#### 3.3 数据获取

```cpp
Ptr<UdpRttClient> client = app_a_to_b->GetObject<UdpRttClient>();

// 获取测量数据
std::vector<int64_t> send_times = client->GetSendRequestTimestamps(); // 发送时间戳
std::vector<int64_t> reply_times = client->GetReplyTimestamps();      // 回复时间戳
std::vector<int64_t> receive_times = client->GetReceiveReplyTimestamps(); // 接收时间戳

// 时间戳为-1表示数据包丢失
```

### 4. 输出结果解析

在`logs_ns3`文件夹中生成两个文件：

1. **pingmesh.txt**：人类可读表格格式
2. **pingmesh.csv**：结构化数据格式，每行包含：
    ```csv
    源节点ID,目标节点ID,序号,发送时间戳,回复时间戳,接收时间戳,
    前向延迟(ms),反向延迟(ms),往返时间(ms),[成功/丢失]
    ```

TL;DR

| 特性                | 调度器模式         | 手动模式         |
|---------------------|-------------------|------------------|
| 配置复杂度          | 低（自动配对）     | 高（手动配置）    |
| 灵活性              | 中等              | 高               |
| 多节点管理          | 自动处理          | 需自行实现        |
| 推荐使用场景        | 全网络监测        | 特定节点对测试    |

建议优先使用调度器模式，除非需要特殊定制的测量场景。

## TCP Optimizer

> ns3-sat-sim/simulator/contrib/basic-sim/doc/tcp_optimizer.md

这里介绍了如何使用 **TCP 优化器**，它可以用来调整默认的 TCP 参数，从而提高 TCP 性能。

TCP 优化器主要用于调整 TCP 协议栈的默认设置，以便优化 TCP 性能。它提供了一些方法来改变 TCP 的基本设置，甚至可以基于最坏情况下的 RTT（往返时间）进行调整。

文档中提到的主要文件：

- **`tcp-optimizer`**：TCP 优化器的实现文件。

要使用 TCP 优化器，你需要在代码中导入相关头文件，并选择适当的优化方法。文档提供了两种优化方式:

### 1. 导入 TCP 优化器

在你的仿真代码中导入 TCP 优化器的头文件：

```cpp
#include "ns3/tcp-optimizer.h"
```

### 2. 优化方法

你有两个选项来使用 TCP 优化器：

**选项 1：优化基础设置**

这种方式会优化 TCP 的一些基础参数设置，但不会修改与时序相关的设置。你可以通过调用以下函数来实现：

```cpp
// 通过设置一些常规值来优化 TCP
TcpOptimizer::OptimizeBasic(basicSimulation);
```

- **`OptimizeBasic`**：此方法将优化一些通用的 TCP 参数，如窗口大小、拥塞控制算法等，但不会调整与时间（如 RTT）相关的参数。

**选项 2：使用最坏情况 RTT 进行优化**

如果你希望在优化时不仅调整基本参数，还考虑最坏情况下的 RTT（往返时间），可以使用以下方法：

```cpp
// 使用最坏情况 RTT 评估来优化 TCP
TcpOptimizer::OptimizeUsingWorstCaseRtt(basicSimulation, topology->GetWorstCaseRttEstimateNs());
```

- **`OptimizeUsingWorstCaseRtt`**：这种方法在执行 **`OptimizeBasic`** 的基础上，额外考虑了最坏情况 RTT，进行进一步的优化
    - 主要是与时间相关的设置（如超时、重传等）。
- `topology->GetWorstCaseRttEstimateNs()` 会返回最坏情况下的 RTT 估算值
    - 这个值会用于进一步调整 TCP 的定时设置，以适应网络的最差情况。

## Future Work

> ns3-sat-sim/simulator/contrib/basic-sim/doc/future_work.md

列出了在 `basic-sim` 模块中可以进行的 **未来工作**，主要涉及拓展和优化当前模块的一些功能：

### 1. 链路过滤器

- **链路利用率跟踪器助手的链路过滤器（Link filters for link (net-device) utilization tracker helper）**：
    - 这项工作建议为链路利用率跟踪器（即监控链路的利用率）增加过滤功能。
    - 链路利用率跟踪器可以帮助你监控网络中各个链路的利用率，加入过滤器后，你可以选择性地监控某些特定链路的利用情况，而不是所有链路。
- **链路队列跟踪器助手的链路过滤器（Link filters for link (net-device) queue tracker helper）**：
    - 类似于链路利用率跟踪器，这项工作建议在链路队列跟踪器中加入过滤功能。
    - 链路队列跟踪器用于监控每个链路上的数据包队列情况，通过添加过滤器，你可以选择特定链路来监控队列长度，而不是全部链路的队列情况。

目前我们做的只有“全局视野”开关过滤器，我们希望能在未来实施“单个链路粒度”的过滤器！

### 2. 添加 RED 排队策略（RED Queueing Discipline）

为拓扑配置中的 RED 排队策略引入参数化选择器（Incorporate a parameterized selector for RED as queueing discipline in the topology configuration）：

**RED（Random Early Detection）** 是一种常用的队列管理策略，特别是在网络中处理拥塞时。

文档建议在拓扑配置中加入一个参数化的选择器，使得用户可以在仿真中灵活选择使用 RED 排队策略，并通过配置来设置 RED 的具体参数。这有助于更好地模拟现实网络中使用的队列策略。

### 3. 链路错误模型

**Add a mapping option for the error model for each link in the point-to-point topology）**

目的是：模拟实际网络中由于噪声或其他因素导致的错误

文档建议在点对点拓扑中加入链路错误模型配置。这允许你为每一条链路设置特定的错误模型。例如，可以设置链路的错误概率，模拟实际网络中由于噪声或其他因素导致的错误。示例中的配置方式是使用 `iid_uniform_random`（均匀随机错误模型），设置不同链路的错误率，如 `0.01` 或 `0.02`。

示例配置：

```c
link_arrival_error_model=map(0->1: iid_uniform_random(pkt, 0.01), 1->0: iid_uniform_random(pkt, 0.02))
```

这里，`0->1` 和 `1->0` 表示从节点 0 到节点 1 和从节点 1 到节点 0 的链路错误模型，`iid_uniform_random(pkt, 0.01)` 表示每个数据包的错误概率为 1%。

相关文档链接：

- [point-to-point-net-device_8cc_source.html](https://www.nsnam.org/doxygen/point-to-point-net-device_8cc_source.html)
- [simple-error-model_8cc_source.html](https://www.nsnam.org/doxygen/simple-error-model_8cc_source.html)
- [error-model_8cc_source.html](https://www.nsnam.org/doxygen/error-model_8cc_source.html)

### 4. 无缓冲版本的 `log-update-helper`

**无缓冲版本的 `log-update-helper`，直接写入文件（Unbuffered version of `log-update-helper.cc/h` which directly writes to file）**：

这项工作建议创建一个无缓冲的版本的 `log-update-helper`，该版本可以直接将日志写入文件。这样可以用于 TCP 的拥塞窗口（cwnd）、进度（progress）和往返时间（RTT）等数据的跟踪。

这对于高效地记录实时数据，避免因缓冲区大小问题而造成的延迟或数据丢失非常有用。

笔者认为可以类比`std::cerror`之于`std::cout`

### 5. 升级到新版本的 ns-3 时的检查项

需要在版本迭代时，确保模型一致性！

当将 ns-3 升级到新版本时，需要检查 **`PointToPointAbHelper`** 是否与 **`PointToPointHelper`** 代码保持一致。这是为了确保在新版本中，点对点模型的功能没有被破坏，避免出现不兼容的情况。

### 6. `basic-sim` 模块的目标与扩展建议

**`basic-sim`** 模块的目标是作为一个起点，用于在网络仿真中进行实验。它为用户提供了一个基础框架，允许用户快速开展实验，并可根据需要进行扩展。

以下是一些可以作为新模块扩展的建议，它们不属于 `basic-sim` 模块的一部分，但可以依赖于 `basic-sim` 模块进行实现：

- **Flowlet 路由器（Flowlet routing arbiter）**：一种基于流分块（flowlet）的路由算法，它可以在多路径路由中进行细粒度的流量调度。
- **长时间运行的 TCP 应用与调度器**：支持长时间运行的 TCP 流应用，可以用于测试长期连接的稳定性和性能。
- **DCTCP 示例**：DCTCP（Data Center TCP）是一种用于数据中心的 TCP 协议，支持高效的拥塞控制，可以作为实验的一部分。


