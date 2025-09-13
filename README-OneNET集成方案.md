# Telegraf OneNET IoT 平台集成方案

## 📋 方案概述

这是一套完整的 SNMP 网络设备监控数据上报到 OneNET IoT 平台的解决方案。

## 🎯 功能特点

✅ **SNMP 数据采集** - 支持系统信息、接口统计、流量监控
✅ **流量速率计算** - 自动计算 bytes/second 和 Mbps
✅ **OneNET 格式转换** - 完全符合 OneNET 平台标准
✅ **MQTT 可靠传输** - QoS=1，确保数据不丢失
✅ **物模型支持** - 支持属性上报和事件上报

## 📄 文件清单

### 配置文件
- `telegraf-onenet-simple.conf` - **推荐使用**，简化版配置
- `telegraf-onenet-final.conf` - 完整版配置（包含事件处理）
- `telegraf-onenet-test.conf` - 测试配置（用于验证格式）

### 物模型文件
- `SNMP网络设备物模型_v2.json` - IoT 平台物模型定义

## 🚀 快速开始

### 1. 启动 SNMP 模拟器
```bash
docker run -d -p 161:161/udp \
  -v $(pwd)/snmp-data:/usr/local/snmpsim/data \
  tandrup/snmpsim
```

### 2. 运行 Telegraf
```bash
# 测试配置
./telegraf --config telegraf-onenet-simple.conf --test

# 正式运行
./telegraf --config telegraf-onenet-simple.conf
```

## 📊 OneNET 上报格式

### 属性数据示例
```json
{
  "id": "1757754090",
  "version": "1.0",
  "params": {
    "system_description": {
      "value": "Cisco IOS Software",
      "time": 1757754090
    },
    "system_uptime": {
      "value": 13818813,
      "time": 1757754090
    },
    "interface_count": {
      "value": 6,
      "time": 1757754090
    },
    "system_name": {
      "value": "testhonmsnetHQ",
      "time": 1757754090
    },
    "total_inbound_rate": {
      "value": 125.5,
      "time": 1757754090
    },
    "total_outbound_rate": {
      "value": 89.3,
      "time": 1757754090
    }
  }
}
```

### 事件数据示例
```json
{
  "id": "1757754120",
  "version": "1.0",
  "params": {
    "link_down": {
      "value": {
        "if_index": 2,
        "if_descr": "GigabitEthernet0/1"
      },
      "time": 1757754120
    }
  }
}
```

## 🔧 MQTT 连接配置

| 参数 | 值 |
|------|-----|
| 服务器 | `tcp://121.40.253.229:1883` |
| 用户名 | `zcddgDRs` |
| 密码 | `PDnqQoKju8DuFvpQ` |
| 客户端ID | `cisco001` |
| 属性主题 | `$SYS/zcddgDRs/cisco001/property/post` |
| 事件主题 | `$SYS/zcddgDRs/cisco001/event/post` |

## 📈 监控指标

### 系统信息
- `system_description` - 设备描述
- `system_name` - 设备名称
- `system_uptime` - 运行时间（秒）
- `interface_count` - 接口总数

### 流量统计
- `total_inbound_rate` - 总入流量速率（Mbps）
- `total_outbound_rate` - 总出流量速率（Mbps）
- `total_inbound_bytes` - 总入流量字节数
- `total_outbound_bytes` - 总出流量字节数

### 关键接口（可配置）
- `uplink_interface_name` - 上联接口名称
- `uplink_inbound_rate` - 上联入流量速率
- `uplink_outbound_rate` - 上联出流量速率
- `uplink_bandwidth` - 上联接口带宽
- `uplink_utilization` - 上联接口使用率

## 🛠️ 自定义配置

### 修改接口识别规则
在配置文件的 Starlark 脚本中修改：

```python
def is_uplink_interface(if_name):
    """定义上联接口匹配规则"""
    uplink_keywords = ["GigabitEthernet0/0", "TenGigabit", "uplink", "trunk"]
    for keyword in uplink_keywords:
        if keyword.lower() in if_name.lower():
            return True
    return False

def is_core_interface(if_name):
    """定义核心接口匹配规则"""
    core_keywords = ["Serial0/0/0", "GigabitEthernet0/1", "core", "backbone"]
    for keyword in core_keywords:
        if keyword.lower() in if_name.lower():
            return True
    return False
```

### 修改 SNMP 采集目标
```toml
[[inputs.snmp]]
  agents = ["192.168.1.1:161", "192.168.1.2:161"]  # 修改为实际设备IP
  version = 2
  community = "your_community"  # 修改为实际community
```

### 修改 OneNET 连接参数
```toml
[[outputs.mqtt]]
  servers = ["tcp://your_onenet_server:1883"]
  username = "your_product_key"
  password = "your_access_key"
  client_id = "your_device_id"
  topic = "$SYS/your_product_key/your_device_id/property/post"
```

## 🚨 故障排除

### 1. SNMP 连接失败
- 检查设备 IP 和 community 字符串
- 确认防火墙允许 UDP 161 端口
- 验证设备 SNMP 服务已启用

### 2. MQTT 连接失败
- 检查网络连接和端口 1883
- 验证 OneNET 平台认证信息
- 查看 Telegraf 日志中的错误信息

### 3. 数据格式错误
- 使用测试配置验证 JSON 格式
- 检查时间戳格式（秒级 vs 毫秒级）
- 确认物模型定义与上报数据一致

## 📝 日志和调试

### 查看 Telegraf 日志
```bash
./telegraf --config telegraf-onenet-simple.conf --debug
```

### 查看 MQTT 发送内容
配置中包含文件输出，可以查看：
```bash
tail -f /tmp/telegraf-onenet.log
```

### 验证 OneNET 格式
```bash
./telegraf --config telegraf-onenet-test.conf --test | jq
```

## 🔄 生产部署建议

1. **使用进程管理器**：如 systemd、supervisor
2. **配置日志轮转**：避免日志文件过大
3. **监控 Telegraf 进程**：确保服务稳定运行
4. **备份配置文件**：定期备份重要配置
5. **设置告警**：监控数据上报状态

## 📞 技术支持

如遇问题，请检查：
1. Telegraf 版本兼容性
2. OneNET 平台 API 文档
3. SNMP 设备配置
4. 网络连接状态

本方案已在 Telegraf v1.37.0 上测试通过。