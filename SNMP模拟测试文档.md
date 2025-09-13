# SNMP 模拟器测试文档

## 环境说明

本文档描述如何在 macOS 环境下使用 Docker 启动 SNMP 模拟器，并配置 Telegraf 进行测试。

## Docker SNMP 模拟器启动

### 1. 使用默认数据启动

```bash
docker run -d -p 161:161/udp tandrup/snmpsim
```

此命令将启动一个使用内置数据的 SNMP 模拟器。

### 2. 使用自定义模拟数据启动

#### 2.1 下载 Cisco 路由器模拟数据

```bash
# 下载 Cisco C1900 路由器的 SNMP 模拟数据
curl -o cisco-router.snmprec \
    https://raw.githubusercontent.com/murrant/librenms-snmpsim/master/captures/cisco-c1900.snmprec
```

#### 2.2 使用自定义数据启动模拟器

```bash
# 创建数据目录并复制模拟数据
mkdir -p snmp-data
cp cisco-router.snmprec snmp-data/

# 启动模拟器并挂载数据目录
docker run -d \
    -p 161:161/udp \
    -v $(pwd)/snmp-data:/usr/share/snmpsim/data \
    tandrup/snmpsim
```

#### 2.3 验证自定义数据加载

```bash
# 查看模拟器日志，确认数据文件加载
docker logs $(docker ps -q --filter ancestor=tandrup/snmpsim)

# 测试特定于 Cisco C1900 的 OID
snmpwalk -v2c -c public localhost .1.3.6.1.2.1.1.1.0
```

### 3. 模拟器特性

#### 默认模拟器
- **设备类型**: 通用网络设备
- **Community**: cisco-router
- **端口**: 161/udp
- **MIB支持**: 基础 RFC1213-MIB

#### 使用 cisco-router.snmprec 数据
- **设备类型**: Cisco C1900 路由器
- **Community**: public
- **端口**: 161/udp
- **MIB支持**: 完整的 Cisco 企业 MIB + 标准 MIB
- **接口数量**: 多个以太网和串行接口
- **系统信息**: 真实的 Cisco IOS 系统描述

**注意**: 使用 cisco-router.snmprec 数据时，community 应设置为 "public"

### 4. 验证模拟器运行

```bash
# 检查容器运行状态
docker ps

# 测试 SNMP 连接 (默认数据)
snmpwalk -v2c -c cisco-router localhost .1.3.6.1.2.1.1.1.0

# 测试 SNMP 连接 (cisco-router.snmprec 数据)
snmpwalk -v2c -c public localhost .1.3.6.1.2.1.1.1.0

# 查看接口表数据
snmpwalk -v2c -c public localhost .1.3.6.1.2.1.2.2
```

## Telegraf 配置文件说明

### 1. 基础 SNMP 采集配置 (telegraf-snmp-simple.conf)

```toml
[[inputs.snmp]]
  agents = ["localhost:161"]
  version = 2
  community = "cisco-router"

  [[inputs.snmp.table]]
    name = "interface"
    inherit_tags = ["hostname"]
    oid = "1.3.6.1.2.1.2.2"

    [[inputs.snmp.table.field]]
      name = "ifIndex"
      oid = "1.3.6.1.2.1.2.2.1.1"
      is_tag = true

    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = "1.3.6.1.2.1.2.2.1.2"
      is_tag = true

    [[inputs.snmp.table.field]]
      name = "ifInOctets"
      oid = "1.3.6.1.2.1.2.2.1.10"

    [[inputs.snmp.table.field]]
      name = "ifOutOctets"
      oid = "1.3.6.1.2.1.2.2.1.16"

[[outputs.file]]
  files = ["stdout"]
  data_format = "influx"
```

**用途**: 基础的网络接口统计数据采集，适用于验证模拟器连接和数据获取。

### 2. 带速率计算的配置 (telegraf-snmp-test.conf)

在基础配置基础上增加:

```toml
[[aggregators.derivative]]
  namepass = ["interface"]
  period = "30s"
  suffix = "_rate"
  fields = ["ifInOctets", "ifOutOctets"]
```

**用途**: 计算每分钟的流量速率，将 SNMP 计数器转换为速率值。

### 3. OneNET 物联网平台集成配置 (telegraf-onenet-simple.conf)

完整的生产环境配置，包含:
- SNMP 数据采集
- 数据格式转换（Starlark 处理器）
- MQTT 输出到 OneNET 平台

**MQTT 连接参数**:
- 服务器: tcp://121.40.253.229:1883
- 用户名: zcddgDRs
- 密码: PDnqQoKju8DuFvpQ
- Topic: $SYS/zcddgDRs/cisco001/property/post

**输出格式**: OneNET 标准 JSON 格式

## 测试步骤

### 方案一: 使用默认数据

#### 1. 启动模拟器
```bash
docker run -d -p 161:161/udp tandrup/snmpsim
```

#### 2. 基础连接测试 (community: cisco-router)
```bash
./telegraf --config telegraf-snmp-simple.conf --test
```

### 方案二: 使用 Cisco C1900 真实数据

#### 1. 下载并启动模拟器
```bash
# 下载模拟数据
curl -o cisco-router.snmprec \
    https://raw.githubusercontent.com/murrant/librenms-snmpsim/master/captures/cisco-c1900.snmprec

# 创建数据目录
mkdir -p snmp-data
cp cisco-router.snmprec snmp-data/

# 启动模拟器
docker run -d \
    -p 161:161/udp \
    -v $(pwd)/snmp-data:/usr/share/snmpsim/data \
    tandrup/snmpsim
```

#### 2. 修改配置文件的 community
需要将配置文件中的 `community = "cisco-router"` 改为 `community = "public"`

#### 3. 测试步骤
```bash
# 基础连接测试
./telegraf --config telegraf-snmp-simple.conf --test

# 速率计算测试
./telegraf --config telegraf-snmp-test.conf --test

# OneNET 集成测试
./telegraf --config telegraf-onenet-simple.conf --test
```

## 故障排除

### 常见问题

1. **端口冲突**: 确保本机 161 端口未被占用
2. **Community 错误**:
   - 默认模拟器使用 "cisco-router"
   - cisco-router.snmprec 数据使用 "public"
3. **防火墙**: 检查 macOS 防火墙设置允许 UDP 161 端口
4. **数据文件未加载**: 检查 docker 挂载路径和文件权限
5. **模拟数据不匹配**: 确认使用正确的 community 字符串对应数据源

### 调试命令

```bash
# 检查 SNMP 连通性 (默认数据)
snmpget -v2c -c cisco-router localhost 1.3.6.1.2.1.1.1.0

# 检查 SNMP 连通性 (cisco-router.snmprec 数据)
snmpget -v2c -c public localhost 1.3.6.1.2.1.1.1.0

# 查看接口表 (cisco-router.snmprec 数据)
snmpwalk -v2c -c public localhost 1.3.6.1.2.1.2.2

# 测试特定接口的流量计数器
snmpget -v2c -c public localhost 1.3.6.1.2.1.2.2.1.10.1
snmpget -v2c -c public localhost 1.3.6.1.2.1.2.2.1.16.1

# 查看模拟器日志
docker logs $(docker ps -q --filter ancestor=tandrup/snmpsim)

# 查看模拟器内部数据文件
docker exec -it $(docker ps -q --filter ancestor=tandrup/snmpsim) ls -la /usr/share/snmpsim/data/
```

## 物模型配置

物模型文件: `华为网络交换机物模型.json`

- **属性**: 扁平化的网络设备统计数据
- **事件**: 基于 SNMP Trap 的网络事件
- **无服务**: 不包含设备控制功能

配合 Telegraf 的 Starlark 处理器将 SNMP 表格数据转换为物模型要求的扁平格式。