# Schwab API 使用说明

**作者**: tommy.wang@accuweather.com  
**最后更新**: 2026-08-19  
**用途**: 股票分析 - 获取日内逐分钟数据和Daily数据

---

## 目录
1. [Python 依赖库](#python-依赖库)
2. [环境配置](#环境配置)
3. [OAuth 认证与长期 Token](#oauth-认证与长期-token)
4. [API 使用方法](#api-使用方法)
5. [数据获取限制](#数据获取限制)
6. [代码示例](#代码示例)
7. [故障排查](#故障排查)

---

## Python 依赖库

### 必需库
| 库名 | 版本 | 用途 | 获取方式 |
|------|------|------|---------|
| `schwab-py` | 最新 | Schwab API 官方 Python 封装 | `pip install schwab-py` |
| `python-dotenv` | >= 0.19.0 | 环境变量管理（API凭证） | `pip install python-dotenv` |
| `pandas` | >= 1.3.0 | 数据处理和聚合 | `pip install pandas` |
| `pytz` | >= 2021.1 | 时区处理（美东 ET） | `pip install pytz` |

### 可选库
| 库名 | 版本 | 用途 | 获取方式 |
|------|------|------|---------|
| `pandas-ta` | >= 0.3.0 | 技术指标计算（RSI、MACD、BB等） | `pip install pandas-ta` |
| `yfinance` | >= 0.1.70 | 备用数据源（历史Daily数据） | `pip install yfinance` |
| `sqlite3` | 内置 | 本地数据库存储 | Python 标准库 |

### 快速安装
```bash
pip install schwab-py python-dotenv pandas pytz pandas-ta yfinance
```

或使用 requirements.txt（项目根目录）：
```bash
pip install -r requirements.txt
```

---

## 环境配置

### 1. 获取 Schwab API 凭证

#### 步骤 1: 注册开发者账户
1. 访问 [Schwab Developer Portal](https://developer.schwabapi.com/)
2. 使用你的 Schwab 账户登录（需要已有 Schwab 账户）
3. 接受开发者协议

#### 步骤 2: 创建应用
1. 点击 "Create an app"
2. 填写应用信息：
   - **App Name**: 例如 "Stock Monitor"
   - **App Type**: 选择 "Individual"
   - **Callback URL**: 例如 `http://localhost:8888/callback`（本地开发）
3. 获得以下凭证：
   - **App Key** (API Key)
   - **App Secret** (API Secret)

### 2. 创建 .env 文件

在项目根目录创建 `.env` 文件（**不要提交到 git**）：

```env
SCHWAB_APP_KEY=your_app_key_here
SCHWAB_APP_SECRET=your_app_secret_here
SCHWAB_CALLBACK_URL=http://localhost:8888/callback
SCHWAB_TOKEN_PATH=./token.json
```

**注意**：
- `.env` 文件应加入 `.gitignore`
- `SCHWAB_TOKEN_PATH` 默认为 `token.json`（首次运行会自动生成）

---

## OAuth 认证与长期 Token

### 工作流程

#### 首次运行流程
```
1. 调用 schwab.auth.easy_client()
   ↓
2. 自动打开浏览器跳转到 Schwab 登录页面
   ↓
3. 用户输入 Schwab 账户凭证并授权
   ↓
4. Callback 返回授权码
   ↓
5. 交换为 Access Token 和 Refresh Token
   ↓
6. 保存到 token.json（自动）
```

#### 后续运行流程
```
1. 读取 token.json 中的 Refresh Token
   ↓
2. 自动刷新 Access Token（过期前）
   ↓
3. 继续使用 API
```

### 实现代码

```python
from schwab_client import get_client

# 首次运行：会打开浏览器进行 OAuth
client = get_client()

# 该 client 可直接使用，token 由库自动管理
quotes = client.get_quotes(['AAPL', 'MSFT'])
```

### Token 管理

#### 长期有效性
- **Access Token**: 30 分钟过期 → 自动刷新
- **Refresh Token**: 7 天过期 → 需要重新授权
- **存储位置**: `token.json`（与代码分离）

#### 手动刷新 Token
如果 refresh token 过期或需要重新授权：

```python
import os
os.remove("token.json")  # 删除过期 token

# 再次运行会触发新的 OAuth 流程
client = get_client()
```

#### 多机器/多用户场景
如果在不同机器运行，需要各自完成一次 OAuth：
```python
# 机器 A
SCHWAB_TOKEN_PATH=./token_machineA.json

# 机器 B
SCHWAB_TOKEN_PATH=./token_machineB.json
```

---

## API 使用方法

### 1. 获取实时报价

```python
from schwab_client import get_client, get_quotes

client = get_client()

# 批量获取多个股票的报价
symbols = ['AAPL', 'MSFT', 'TSLA']
quotes = get_quotes(client, symbols)

# 返回格式：
# {
#   'AAPL': {
#     'last': 150.25,           # 最新价
#     'change': 2.50,           # 价格变化 (与前一收盘比)
#     'change_pct': 1.69,       # 变化百分比
#     'volume': 5000000,        # 成交量
#     'bid': 150.24,            # 买价
#     'ask': 150.26             # 卖价
#   },
#   ...
# }
```

### 2. 获取 5 分钟 K 线

```python
from schwab_client import get_client, get_price_history_5m

client = get_client()

# 获取某股票最近 10 天的 5 分钟 K 线
rows = get_price_history_5m(client, 'AAPL')

# 返回格式（list of dict）：
# [
#   {
#     'symbol': 'AAPL',
#     'dt': 1692086100000,      # Unix timestamp (ms)
#     'open': 150.10,
#     'high': 150.50,
#     'low': 150.00,
#     'close': 150.25,
#     'volume': 45000
#   },
#   ...
# ]
```

### 3. 获取账户持仓

```python
from schwab_client import get_client, get_account_positions

client = get_client()

# 获取账户中所有持仓的 ticker（只包括股票和 ETF）
symbols = get_account_positions(client)
print(symbols)  # ['AAPL', 'MSFT', 'SPY', ...]
```

### 4. 原始 API 调用（高级）

```python
from schwab_client import get_client
import schwab

client = get_client()

# 使用 schwab 库的完整 API
PH = schwab.client.Client.PriceHistory

# 获取 1 分钟数据（如果支持）
response = client.get_price_history(
    'AAPL',
    period_type=PH.PeriodType.DAY,
    period=PH.Period.TEN_DAYS,
    frequency_type=PH.FrequencyType.MINUTE,
    frequency=PH.Frequency.EVERY_FIVE_MINUTES,
    need_extended_hours_data=False
)
response.raise_for_status()
print(response.json())
```

---

## 数据获取限制

### API 调用限制

| 限制类型 | 规格 | 说明 |
|---------|------|------|
| **请求频率** | 120/分钟 | 平均每秒 2 次请求 |
| **并发连接** | 10 | 同时最多 10 个连接 |
| **批量查询** | 最多 200 个 symbols | 单次 get_quotes 最多 200 个 |

### 数据范围限制

| 数据类型 | 时间范围 | 最小单位 | 备注 |
|---------|---------|---------|------|
| **5 分钟 K 线** | 最近 10 天 | 5 分钟 | 盘中数据（9:30-16:00 ET） |
| **1 分钟数据** | 需自己构建 | 1 分钟 | 通过实时报价轮询构建 |
| **实时报价** | 实时 | 实时 | 延迟 < 15 分钟（免费版） |
| **日线** | 无直接 API | 1 天 | 推荐使用 yfinance 补充 |

### 市场时间限制

```python
# 仅在这个时间段返回数据
MARKET_OPEN  = 09:30 (美东 ET)
MARKET_CLOSE = 16:00 (美东 ET)
```

- **盘前/盘后数据**: 可获取（如果设置 `need_extended_hours_data=True`），但不推荐
- **周末/节假日**: 无数据返回
- **时区**: 所有时间均为美东 ET（Eastern Time）

### 扩展数据获取

#### 获取 Daily 数据
由于 Schwab API 没有直接的 Daily 数据接口，使用 **yfinance 备用**：

```python
import yfinance as yf
from datetime import date, timedelta

ticker = yf.Ticker('AAPL')
df = ticker.history(
    start=(date.today() - timedelta(days=60)).strftime('%Y-%m-%d'),
    end=date.today().strftime('%Y-%m-%d'),
    interval='1d'
)
print(df)  # DataFrame: Date, Open, High, Low, Close, Volume
```

#### 获取 1 分钟数据
需要**自己构建**（通过轮询实时报价）：

```python
# fetcher.py 中的实现方式：
# - 每 60 秒调用一次 get_quotes()
# - 记录当前价格 = 1 分钟 K 线的 OHLCV
# - 累计成交量差分 = 分钟成交量
```

---

## 代码示例

### 示例 1: 连续获取实时报价（每 60 秒）

```python
import time
import os
from datetime import datetime, timezone
import pytz
from schwab_client import get_client, get_quotes

ET = pytz.timezone("America/New_York")
POLL_INTERVAL = 60  # 秒

def continuous_fetch():
    client = get_client()
    symbols = ['AAPL', 'MSFT', 'TSLA']
    
    while True:
        now_et = datetime.now(ET)
        
        # 仅在交易时段获取
        if 9 <= now_et.hour < 16:
            try:
                quotes = get_quotes(client, symbols)
                for symbol, data in quotes.items():
                    print(f"{symbol}: {data['last']} ({data['change_pct']:+.2f}%)")
            except Exception as e:
                print(f"Error: {e}")
        
        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    continuous_fetch()
```

### 示例 2: 批量下载 5 分钟 K 线到 CSV

```python
import csv
from datetime import datetime
from schwab_client import get_client, get_price_history_5m

def download_5m_to_csv(symbols: list, output_file: str = "candles_5m.csv"):
    client = get_client()
    
    with open(output_file, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['symbol', 'datetime', 'open', 'high', 'low', 'close', 'volume'])
        writer.writeheader()
        
        for symbol in symbols:
            try:
                rows = get_price_history_5m(client, symbol)
                for row in rows:
                    dt_str = datetime.fromtimestamp(row['dt'] / 1000).isoformat()
                    writer.writerow({
                        'symbol': row['symbol'],
                        'datetime': dt_str,
                        'open': row['open'],
                        'high': row['high'],
                        'low': row['low'],
                        'close': row['close'],
                        'volume': row['volume']
                    })
                print(f"{symbol}: {len(rows)} bars")
            except Exception as e:
                print(f"{symbol} error: {e}")

if __name__ == "__main__":
    download_5m_to_csv(['AAPL', 'MSFT', 'TSLA'])
```

### 示例 3: 与本地 SQLite 数据库集成

```python
import sqlite3
from datetime import datetime
from schwab_client import get_client, get_price_history_5m

DB_FILE = "stocks.db"

def init_db():
    conn = sqlite3.connect(DB_FILE)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS candles_5m (
            symbol TEXT NOT NULL,
            dt INTEGER NOT NULL,
            open REAL, high REAL, low REAL, close REAL, volume INTEGER,
            PRIMARY KEY (symbol, dt)
        )
    """)
    conn.commit()
    conn.close()

def save_to_db(rows: list):
    conn = sqlite3.connect(DB_FILE)
    conn.executemany("""
        INSERT OR REPLACE INTO candles_5m 
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, [(r['symbol'], r['dt'], r['open'], r['high'], r['low'], r['close'], r['volume']) for r in rows])
    conn.commit()
    conn.close()

def main():
    init_db()
    client = get_client()
    
    symbols = ['AAPL', 'MSFT']
    for symbol in symbols:
        rows = get_price_history_5m(client, symbol)
        save_to_db(rows)
        print(f"{symbol}: saved {len(rows)} bars")

if __name__ == "__main__":
    main()
```

---

## 故障排查

### 问题 1: "ModuleNotFoundError: No module named 'schwab'"

**解决方案**:
```bash
pip install schwab-py --upgrade
```

### 问题 2: "SCHWAB_APP_KEY not found in environment"

**原因**: `.env` 文件未被正确加载

**解决方案**:
```python
# 确保在代码最前面导入：
from dotenv import load_dotenv
load_dotenv()  # 必须在导入 schwab_client 之前

# 验证环境变量
import os
print(os.environ.get("SCHWAB_APP_KEY"))  # 应该显示你的 API Key
```

### 问题 3: "OAuth callback failed" 或浏览器无法重定向

**原因**: Callback URL 不匹配

**解决方案**:
1. 确保 `.env` 中的 `SCHWAB_CALLBACK_URL` 与开发者门户中注册的完全相同
2. 确保本地有一个能监听该 URL 的 HTTP 服务（schwab-py 会自动启动）
3. 如果使用远程服务器，使用公网 IP 而不是 localhost

### 问题 4: "Token expired" 或需要重新授权

**原因**: Refresh token 超过 7 天未使用

**解决方案**:
```bash
# 删除过期 token，重新授权
rm token.json

# 再次运行代码，会重新触发 OAuth
python your_script.py
```

### 问题 5: 返回的数据为空或 None

**原因**: 
- 在非交易时段请求
- 股票代码错误
- API 频率限制

**解决方案**:
```python
# 检查市场时间
from datetime import datetime
import pytz
ET = pytz.timezone("America/New_York")
now = datetime.now(ET)
print(f"Current ET time: {now.strftime('%H:%M')}")
print(f"Market open: {9 <= now.hour < 16}")

# 添加错误处理和重试
import time
try:
    rows = get_price_history_5m(client, 'AAPL')
    if not rows:
        print("No data returned - check market hours")
except Exception as e:
    print(f"API error: {e}")
    time.sleep(60)  # 等待后重试
```

### 问题 6: 频率限制（429 Too Many Requests）

**原因**: 超过 120 请求/分钟

**解决方案**:
```python
import time

# 批量请求时添加延迟
for symbol in symbols:
    rows = get_price_history_5m(client, symbol)
    # ... 处理数据 ...
    time.sleep(0.5)  # 每个请求间隔 0.5 秒
```

---

## 附录：Schwab API 官方资源

| 资源 | URL |
|------|-----|
| 开发者门户 | https://developer.schwabapi.com/ |
| API 文档 | https://developer.schwabapi.com/docs/swagger/ |
| Python 库 | https://github.com/alexgolec/schwab-py |
| 常见问题 | https://developer.schwabapi.com/docs/faq/ |
| 支持论坛 | https://developer.schwabapi.com/community/ |

---

**问题反馈**: 如有问题，请检查上述[故障排查](#故障排查)部分或访问官方文档。
