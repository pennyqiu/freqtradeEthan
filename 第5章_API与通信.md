# 第5章：API与通信

## 一、Telegram 机器人

### 1.1 配置 Telegram

#### 第一步：创建机器人

1. 在 Telegram 中搜索 `@BotFather`
2. 发送命令 `/newbot`
3. 输入机器人名称（如：My Trading Bot）
4. 输入机器人用户名（如：my_trading_bot）
5. 获得 Token（类似：`1234567890:ABCdefGhIJKlmNoPQRsTUVwxyZ`）

#### 第二步：获取 Chat ID

1. 向你的机器人发送任意消息
2. 在浏览器访问：
```
https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
```
3. 找到 `"chat":{"id":123456789}` 中的数字

#### 第三步：配置 Freqtrade

```json
{
    "telegram": {
        "enabled": true,
        "token": "1234567890:ABCdefGhIJKlmNoPQRsTUVwxyZ",
        "chat_id": "123456789",
        "allow_custom_messages": true,
        "notification_settings": {
            "status": "on",
            "warning": "on",
            "startup": "on",
            "entry": "on",
            "entry_fill": "on",
            "entry_cancel": "on",
            "exit": {
                "roi": "on",
                "stop_loss": "on",
                "exit_signal": "on",
                "trailing_stop_loss": "on",
                "custom_exit": "on",
                "partial_exit": "on"
            },
            "exit_fill": "on",
            "exit_cancel": "on",
            "protection_trigger": "on",
            "protection_trigger_global": "on",
            "strategy_msg": "on"
        },
        "reload": true,
        "balance_dust_level": 0.01
    }
}
```

### 1.2 Telegram 命令

启动机器人后，在 Telegram 中可以使用以下命令：

#### 基础命令

```
/start          - 启动机器人
/stop           - 停止机器人（保留当前持仓）
/stopentry      - 停止开新仓（但继续处理现有仓位）
/reload_config  - 重新加载配置文件
/status         - 查看当前持仓
/count          - 查看持仓数量统计
/balance        - 查看账户余额
/help           - 查看帮助信息
```

#### 信息查询命令

```
/profit         - 查看盈利统计（今日/本周/本月）
/stats          - 查看交易统计
/daily <n>      - 查看最近n天的每日盈利
/weekly <n>     - 查看最近n周的每周盈利
/monthly <n>    - 查看最近n月的每月盈利
/performance    - 查看各交易对的表现
/whitelist      - 查看当前交易对白名单
/blacklist      - 查看黑名单
/locks          - 查看当前的交易锁定
```

#### 交易控制命令（谨慎使用！）

```
/forcebuy <pair> [rate]    - 强制买入指定交易对
/forceexit <trade_id>      - 强制退出指定持仓
/forceexit all             - 强制退出所有持仓
/delete <trade_id>         - 删除指定交易记录
```

#### 配置命令

```
/show_config    - 显示当前配置
/logs           - 显示最新日志
/version        - 显示版本信息
```

### 1.3 在策略中发送自定义消息

```python
from freqtrade.strategy import IStrategy
from freqtrade.enums import RPCMessageType

class TelegramStrategy(IStrategy):
    
    def populate_entry_trend(self, dataframe, metadata):
        # 正常的买入逻辑
        dataframe.loc[
            (dataframe['rsi'] < 30),
            'enter_long'
        ] = 1
        
        # 检测到重要信号时发送通知
        last_candle = dataframe.iloc[-1]
        if last_candle['enter_long'] == 1:
            # 发送自定义消息到 Telegram
            self.dp.send_msg({
                'type': RPCMessageType.STRATEGY_MSG,
                'msg': f"⚠️ {metadata['pair']} 检测到买入信号!\n"
                       f"价格: {last_candle['close']:.2f}\n"
                       f"RSI: {last_candle['rsi']:.2f}"
            })
        
        return dataframe
    
    def custom_exit(self, pair, trade, current_time, current_rate, current_profit, **kwargs):
        # 发送持仓提醒
        if current_profit > 0.10:  # 盈利超过10%
            self.dp.send_msg({
                'type': RPCMessageType.STRATEGY_MSG,
                'msg': f"🎉 {pair} 已盈利 {current_profit*100:.2f}%，考虑获利了结！"
            })
        
        return None
```

---

## 二、REST API

### 2.1 启用 API Server

配置文件：

```json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "127.0.0.1",  // 仅本地访问
        // "listen_ip_address": "0.0.0.0",  // 允许外部访问（谨慎！）
        "listen_port": 8080,
        "verbosity": "info",
        "username": "freqtrader",
        "password": "SuperSecurePassword123!",  // 修改为强密码
        "jwt_secret_key": "randomly-generated-secret-key",
        "CORS_origins": ["http://localhost:3000"],
        "ws_token": "your-websocket-token"
    }
}
```

启动后访问：`http://localhost:8080`

### 2.2 API 认证

所有API请求都需要JWT认证：

```python
import requests
from requests.auth import HTTPBasicAuth

API_URL = "http://localhost:8080"
USERNAME = "freqtrader"
PASSWORD = "SuperSecurePassword123!"

# 1. 获取Token
def get_token():
    response = requests.post(
        f"{API_URL}/api/v1/token/login",
        auth=HTTPBasicAuth(USERNAME, PASSWORD)
    )
    if response.status_code == 200:
        return response.json()['access_token']
    else:
        raise Exception(f"认证失败: {response.text}")

# 使用Token
token = get_token()
headers = {
    'Authorization': f'Bearer {token}',
    'Content-Type': 'application/json'
}
```

### 2.3 常用 API 接口

#### 查询状态

```python
# 获取机器人状态
def get_status():
    response = requests.get(f"{API_URL}/api/v1/status", headers=headers)
    return response.json()

status = get_status()
for trade in status:
    print(f"{trade['pair']}: {trade['profit_pct']:.2f}% "
          f"(开仓: {trade['open_date']}, 价格: {trade['open_rate']:.2f})")
```

#### 查询盈利

```python
# 获取盈利统计
def get_profit():
    response = requests.get(f"{API_URL}/api/v1/profit", headers=headers)
    return response.json()

profit = get_profit()
print(f"总盈利: {profit['profit_all_coin']:.4f} {profit['stake_currency']}")
print(f"总盈利百分比: {profit['profit_all_percent_mean']:.2f}%")
print(f"胜率: {profit['winning_trades']/profit['trade_count']*100:.2f}%")
```

#### 查询余额

```python
# 获取账户余额
def get_balance():
    response = requests.get(f"{API_URL}/api/v1/balance", headers=headers)
    return response.json()

balance = get_balance()
for currency, data in balance['currencies'].items():
    if data['total'] > 0:
        print(f"{currency}: {data['free']:.4f} (可用) / {data['total']:.4f} (总计)")
```

#### 查询性能

```python
# 获取各交易对的表现
def get_performance():
    response = requests.get(f"{API_URL}/api/v1/performance", headers=headers)
    return response.json()

performance = get_performance()
for pair_data in sorted(performance, key=lambda x: x['profit_pct'], reverse=True)[:10]:
    print(f"{pair_data['pair']}: {pair_data['profit_pct']:.2f}% ({pair_data['count']} 笔)")
```

#### 控制机器人

```python
# 启动机器人
def start_bot():
    response = requests.post(f"{API_URL}/api/v1/start", headers=headers)
    return response.json()

# 停止机器人
def stop_bot():
    response = requests.post(f"{API_URL}/api/v1/stop", headers=headers)
    return response.json()

# 停止开新仓
def stopentry():
    response = requests.post(f"{API_URL}/api/v1/stopentry", headers=headers)
    return response.json()

# 重载配置
def reload_config():
    response = requests.post(f"{API_URL}/api/v1/reload_config", headers=headers)
    return response.json()
```

#### 强制交易（谨慎！）

```python
# 强制买入
def force_buy(pair, price=None):
    data = {"pair": pair}
    if price:
        data["price"] = price
    
    response = requests.post(
        f"{API_URL}/api/v1/forcebuy",
        headers=headers,
        json=data
    )
    return response.json()

# 强制卖出
def force_exit(trade_id, ordertype='market'):
    data = {
        "tradeid": str(trade_id),
        "ordertype": ordertype
    }
    
    response = requests.post(
        f"{API_URL}/api/v1/forceexit",
        headers=headers,
        json=data
    )
    return response.json()

# 使用示例（谨慎！）
# result = force_buy("BTC/USDT")
# result = force_exit(123)
```

### 2.4 完整的监控脚本

```python
#!/usr/bin/env python3
"""
Freqtrade 监控脚本
每分钟检查一次状态，异常时发送告警
"""

import requests
import time
from datetime import datetime
from requests.auth import HTTPBasicAuth

# 配置
API_URL = "http://localhost:8080"
USERNAME = "freqtrader"
PASSWORD = "SuperSecurePassword123!"
CHECK_INTERVAL = 60  # 检查间隔（秒）

def get_token():
    response = requests.post(
        f"{API_URL}/api/v1/token/login",
        auth=HTTPBasicAuth(USERNAME, PASSWORD)
    )
    return response.json()['access_token']

def check_bot_status(headers):
    """检查机器人状态"""
    try:
        response = requests.get(f"{API_URL}/api/v1/ping", headers=headers, timeout=5)
        return response.status_code == 200
    except:
        return False

def get_open_trades(headers):
    """获取开仓数量"""
    response = requests.get(f"{API_URL}/api/v1/status", headers=headers)
    return len(response.json())

def get_profit_summary(headers):
    """获取盈利摘要"""
    response = requests.get(f"{API_URL}/api/v1/profit", headers=headers)
    return response.json()

def main():
    print(f"[{datetime.now()}] 启动监控...")
    
    token = get_token()
    headers = {'Authorization': f'Bearer {token}'}
    
    while True:
        try:
            # 检查机器人是否在线
            if not check_bot_status(headers):
                print(f"[{datetime.now()}] ⚠️  机器人离线！")
                # 这里可以发送告警（邮件/短信/webhook）
            else:
                # 获取统计信息
                open_trades = get_open_trades(headers)
                profit = get_profit_summary(headers)
                
                print(f"[{datetime.now()}] ✓ 机器人运行中")
                print(f"  开仓数: {open_trades}")
                print(f"  总盈利: {profit['profit_all_coin']:.4f} {profit['stake_currency']}")
                print(f"  今日盈利: {profit['profit_today_coin']:.4f} {profit['stake_currency']}")
                
                # 检查异常情况
                if open_trades == 0:
                    print(f"  ⚠️  当前无开仓，策略可能无信号")
                
                # 检查大额亏损
                if profit['profit_today_coin'] < -100:  # 今日亏损超过100 USDT
                    print(f"  ⚠️  今日亏损较大！")
        
        except Exception as e:
            print(f"[{datetime.now()}] ❌ 检查失败: {e}")
            # 可能Token过期，重新获取
            try:
                token = get_token()
                headers = {'Authorization': f'Bearer {token}'}
            except:
                pass
        
        # 等待下一次检查
        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    main()
```

---

## 三、Webhook 通知

### 3.1 配置 Webhook

```json
{
    "webhook": {
        "enabled": true,
        "url": "https://your-webhook-url.com/freqtrade",
        "format": "json",  // 或 "form", "raw"
        "webhookentry": {
            "value1": "Entered {pair}",
            "value2": "at {open_rate:.2f}",
            "value3": "{enter_tag}"
        },
        "webhookexit": {
            "value1": "Exited {pair}",
            "value2": "Profit: {profit_ratio:.2%}",
            "value3": "{exit_reason}"
        },
        "webhookstatus": {
            "value1": "Status: {status}",
            "value2": "Open trades: {open_trades}"
        }
    }
}
```

### 3.2 接收 Webhook 的服务器示例

```python
#!/usr/bin/env python3
"""
简单的Webhook接收服务器
"""

from flask import Flask, request
import json

app = Flask(__name__)

@app.route('/freqtrade', methods=['POST'])
def webhook_handler():
    """处理Freqtrade的Webhook"""
    
    # 获取数据
    if request.is_json:
        data = request.get_json()
    else:
        data = request.form.to_dict()
    
    # 打印接收到的数据
    print(f"收到Webhook: {json.dumps(data, indent=2)}")
    
    # 这里可以添加自定义逻辑
    # 例如：发送到其他平台、记录到数据库、触发其他操作等
    
    return {'status': 'success'}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 四、Discord 集成

### 4.1 配置 Discord

```json
{
    "discord": {
        "enabled": true,
        "webhook_url": "https://discord.com/api/webhooks/YOUR_WEBHOOK_URL",
        "exit_fill": true,
        "entry_fill": true,
        "exit_cancel": true,
        "entry_cancel": true
    }
}
```

获取 Discord Webhook URL：
1. 在 Discord 服务器设置中选择"集成"
2. 创建 Webhook
3. 复制 Webhook URL

---

## 五、Web UI（FreqUI）

### 5.1 启动 Web UI

Web UI 会自动随 API Server 启动：

```bash
freqtrade trade --config config.json --strategy MyStrategy
```

访问：`http://localhost:8080`

### 5.2 Web UI 功能

- **仪表板**：总览统计、权益曲线
- **交易列表**：查看所有开仓和历史交易
- **性能分析**：各交易对表现、每日盈利
- **图表分析**：K线图、指标图
- **日志查看**：实时日志
- **手动控制**：强制买卖、暂停/恢复

### 5.3 远程访问配置

如果需要从外网访问（例如手机监控）：

```json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",  // 允许所有IP访问
        "listen_port": 8080,
        "username": "admin",
        "password": "VeryStrongPassword123!@#",  // 务必使用强密码
        "jwt_secret_key": "very-long-random-secret-key"
    }
}
```

**安全建议：**
1. 使用强密码
2. 考虑使用反向代理（Nginx）并启用 HTTPS
3. 配置防火墙规则
4. 定期更换密码和JWT密钥

---

## 六、实战：完整的监控系统

### 6.1 多渠道通知策略

```python
class MonitoringStrategy(IStrategy):
    """
    带完整监控的策略
    """
    
    def bot_loop_start(self, current_time, **kwargs):
        """每个循环检查一次"""
        
        # 检查异常情况
        open_trades = Trade.get_open_trades()
        
        # 1. 检查是否有持仓时间过长的交易
        for trade in open_trades:
            duration = (current_time - trade.open_date_utc).total_seconds() / 3600
            if duration > 48:  # 超过48小时
                self.dp.send_msg({
                    'type': RPCMessageType.STRATEGY_MSG,
                    'msg': f"⚠️ {trade.pair} 已持仓 {duration:.1f} 小时，请关注！"
                })
        
        # 2. 检查大额亏损
        for trade in open_trades:
            current_profit = trade.calc_profit_ratio(
                self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)[0].iloc[-1]['close']
            )
            if current_profit < -0.08:  # 亏损超过8%
                self.dp.send_msg({
                    'type': RPCMessageType.STRATEGY_MSG,
                    'msg': f"🚨 {trade.pair} 亏损 {current_profit*100:.2f}%，即将触及止损！"
                })
        
        # 3. 每天发送总结（在每天00:00）
        if current_time.hour == 0 and current_time.minute < 5:
            self.send_daily_summary()
    
    def send_daily_summary(self):
        """发送每日总结"""
        from datetime import timedelta
        
        today = datetime.now().date()
        today_trades = Trade.get_trades_proxy(
            open_date=datetime.combine(today, datetime.min.time())
        )
        
        if today_trades:
            total_profit = sum([t.close_profit_abs for t in today_trades if t.close_profit_abs])
            win_count = len([t for t in today_trades if t.close_profit_abs > 0])
            
            self.dp.send_msg({
                'type': RPCMessageType.STRATEGY_MSG,
                'msg': f"📊 今日总结\n"
                       f"交易次数: {len(today_trades)}\n"
                       f"盈利次数: {win_count}\n"
                       f"胜率: {win_count/len(today_trades)*100:.1f}%\n"
                       f"总盈利: {total_profit:.2f} USDT"
            })
```

---

## 本章小结

本章学习了 Freqtrade 的多种通信和监控方式：

1. **Telegram 机器人**：最常用的监控方式，支持丰富的命令
2. **REST API**：程序化控制，可集成到其他系统
3. **Webhook**：推送通知到其他服务
4. **Discord 集成**：团队协作场景
5. **Web UI**：可视化界面，直观便捷

合理使用这些工具，可以建立完善的监控和告警系统。

下一章我们将学习常见问题的排查和调试技巧。

