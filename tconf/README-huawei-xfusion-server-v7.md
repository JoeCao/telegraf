# 华为xFusion Server v7 Telegraf监控配置

## 概述

此配置文件 `telegraf-huawei-xfusion-server-v7.conf` 是基于Zabbix模板 "Huawei xFusion Server v7" 忠实转换而来的Telegraf配置，用于监控华为超聚变服务器。

## 配置文件特性

### 1. 监控范围
该配置涵盖了原Zabbix模板的所有监控项目：

**基础系统监控：**
- 系统信息（联系人、描述、位置、名称、序列号等）
- 系统运行时间和CPU空闲率
- CPU使用率和内存使用率
- 系统总功耗
- ICMP连通性测试

**硬件健康状态监控：**
- 整体健康状态
- CPU健康状态
- 内存健康状态
- 磁盘健康状态
- 风扇健康状态
- 电源健康状态
- 温度健康状态
- PCIe设备健康状态

**许可证和固件：**
- 许可证到期时间
- 固件升级状态

### 2. 自动发现功能
配置包含了完整的硬件组件自动发现：

- **CPU发现：** 监控每个CPU的状态和厂商信息
- **内存发现：** 监控每个内存模块的容量、类型、状态
- **物理磁盘发现：** 监控每个磁盘的容量、状态、厂商
- **磁盘阵列控制器发现：** 监控RAID控制器和BBU状态
- **风扇发现：** 监控每个风扇的转速和状态
- **电源发现：** 监控每个电源的功率和状态
- **温度传感器发现：** 监控各个温度传感器读数
- **网络接口发现：** 监控网络接口流量和状态

### 3. 数据处理
配置包含了完整的数据预处理功能：

**单位转换：**
- 内存大小：KB → B，MB → B
- 磁盘大小：GB → B
- 系统运行时间：百分之一秒 → 秒
- 温度值：十分之一摄氏度 → 摄氏度
- 功耗：kWh → Wh
- 网络速度：Mbps → bps

**状态映射：**
- 健康状态：1=ok, 2=minor, 3=major, 4=critical, 5=absence, 6=unknown
- 电源状态：1=normalPowerOff, 2=powerOn, 3=forcedSystemReset等
- 网络接口状态：1=up, 2=down, 3=testing等

## 使用方法

### 1. 配置修改
在使用前，请修改以下参数：

```toml
# 在所有SNMP配置中修改：
agents = ["udp://YOUR_SERVER_IP:161"]  # 替换为实际服务器IP
community = "YOUR_COMMUNITY"           # 替换为实际SNMP社区名

# 在ping配置中修改：
urls = ["YOUR_SERVER_IP"]              # 替换为实际服务器IP
```

### 2. 输出配置
配置文件默认使用InfluxDB v2作为输出，需要配置：

```toml
[[outputs.influxdb_v2]]
  urls = ["http://your-influxdb:8086"]
  token = "your-token-here"
  organization = "your-org"
  bucket = "huawei_servers"
```

或者可以取消注释文件输出用于调试：

```toml
[[outputs.file]]
  files = ["/tmp/telegraf-huawei-xfusion.out"]
  data_format = "json"
```

### 3. 启动监控
```bash
# 测试配置
telegraf --config telegraf-huawei-xfusion-server-v7.conf --test

# 运行监控
telegraf --config telegraf-huawei-xfusion-server-v7.conf
```

## 对应的OID映射

### 系统信息OID
- 系统联系人: 1.3.6.1.2.1.1.4.0
- 系统描述: 1.3.6.1.2.1.1.1.0
- 系统位置: 1.3.6.1.2.1.1.6.0
- 系统名称: 1.3.6.1.2.1.1.5.0
- 系统运行时间: 1.3.6.1.2.1.1.3.0

### 华为专有OID（企业OID: 1.3.6.1.4.1.58132.2.235.1.1）
- 整体健康状态: .1.1.0
- 设备名称: .1.6.0
- 硬件序列号: .1.7.0
- 电源状态: .1.12.0
- CPU使用率: .1.23.0
- 内存使用率: .1.25.0
- 系统功耗: .20.4.0

### 硬件组件健康状态OID
- CPU健康: .15.1.0
- 内存健康: .16.1.0
- 磁盘健康: .18.1.0
- 风扇健康: .8.3.0
- 电源健康: .6.1.0
- 温度健康: .26.1.0
- PCIe设备健康: .24.1.0

## 标签体系

### 全局标签
```
device_type = "huawei_xfusion_server"
vendor = "huawei"
model = "v7"
```

### 组件标签
每个硬件组件都包含相应的component标签：
- `component = "cpu"` - CPU相关指标
- `component = "memory"` - 内存相关指标
- `component = "physicaldisk"` - 物理磁盘相关指标
- `component = "controller"` - 磁盘控制器相关指标
- `component = "fan"` - 风扇相关指标
- `component = "power"` - 电源相关指标
- `component = "temperature"` - 温度相关指标
- `component = "network"` - 网络相关指标

### 实例标签
对于发现的硬件组件，包含实例标识标签：
- `cpu_index`, `cpu_name` - CPU实例
- `memory_index`, `memory_name`, `memory_type` - 内存实例
- `disk_index`, `disk_name` - 磁盘实例
- `psu_index`, `psu_name` - 电源实例
- `temp_index`, `temp_name` - 温度传感器实例
- `interface_index`, `interface_name` - 网络接口实例

## 性能和优化

### 采集间隔
- 默认采集间隔：60秒
- PING监控：实时
- 可根据需要调整interval参数

### 数据量估算
对于单台服务器，预计每分钟产生约500-1000个数据点，具体取决于：
- CPU核心数量
- 内存条数量
- 磁盘数量
- 网络接口数量
- 温度传感器数量

### 优化建议
- 对于大规模部署，建议适当调整采集间隔
- 可以选择性禁用某些不需要的发现规则
- 考虑使用数据过滤减少不必要的指标

## 故障排除

### 常见问题
1. **SNMP连接失败**
   - 检查IP地址和社区名配置
   - 确认目标设备SNMP服务已启用
   - 检查防火墙设置

2. **部分OID无法获取数据**
   - 确认设备固件版本支持相应OID
   - 检查SNMP权限配置

3. **数据类型错误**
   - 检查conversion参数设置
   - 某些字符串类型数据可能需要特殊处理

### 调试方法
```bash
# 启用调试模式
telegraf --config telegraf-huawei-xfusion-server-v7.conf --debug

# 仅测试SNMP输入
telegraf --config telegraf-huawei-xfusion-server-v7.conf --test --input-filter snmp
```

## 版本信息

- **配置版本：** v1.0
- **兼容设备：** 华为超聚变 8600v7
- **兼容固件：** iBMC 3.07.03.39
- **基于MIB：** 2288HV6-16DIMM-iBMC_3.07.03.39_MIB
- **创建时间：** 2025-06-17
- **创建人：** 王磊（基于Zabbix模板转换）

## 许可证

此配置基于开源项目Telegraf，遵循MIT许可证。