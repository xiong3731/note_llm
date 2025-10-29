# [MCP(Model Context Protocol)](https://modelcontextprotocol.io/docs/getting-started/intro)

> MCP（模型上下文协议）是一种将 AI 应用程序连接到外部系统的开源标准。

架构 https://modelcontextprotocol.io/docs/learn/architecture

## 构建mcp服务器

介绍 https://modelcontextprotocol.io/docs/learn/server-concepts#resources

qs https://modelcontextprotocol.io/docs/develop/connect-local-servers



### 安装uv

参考[文档](../../notes/研发/PYTHON/uv.md)

### 开始

```bash
# Create a new directory for our project
uv init weather
cd weather

# Create virtual environment and activate it
uv venv
source .venv/bin/activate

# Install dependencies
uv add "mcp[cli]" httpx

# Create our server file
touch weather.py
```

> `"mcp[cli]"`
>  表示安装 **`mcp` 包** 及其 `cli` 可选依赖（相当于 `pip install mcp[cli]`）。
>  这是 OpenAI 的 **Model Context Protocol (MCP)** 相关工具包。
>
> `httpx`
>  是一个现代的异步 HTTP 客户端，用来发网络请求。

### 编辑 weather.py

```python
# ==============================
# weather.py - MCP 天气服务端示例
# ==============================

from typing import Any          # 导入类型标注模块，用于指定参数与返回值类型
import httpx                    # 导入 httpx 库，用于执行异步 HTTP 请求
from mcp.server.fastmcp import FastMCP  # 从 mcp.server.fastmcp 导入 FastMCP，用于快速创建 MCP 服务器


# ==============================
# 1. 初始化 FastMCP 服务器实例
# ==============================

# 创建 MCP 服务器实例，命名为 "weather"
# 该名称将在 MCP 客户端（如 Claude、ChatGPT）中显示为工具名
mcp = FastMCP("weather")


# ==============================
# 2. 定义常量
# ==============================

NWS_API_BASE = "https://api.weather.gov"   # 美国国家气象局（NWS）的 API 根地址
USER_AGENT = "weather-app/1.0"             # 自定义 User-Agent，用于在 HTTP 请求头中标识客户端


# ==============================
# 3. 通用请求函数（异步）
# ==============================

async def make_nws_request(url: str) -> dict[str, Any] | None:
    """向 NWS（美国国家气象局）API 发起请求，并带有错误处理。"""
    
    # 设置 HTTP 请求头
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/geo+json"  # 指定期望的返回格式
    }
    
    # 使用异步客户端发送请求
    async with httpx.AsyncClient() as client:
        try:
            # 向目标 URL 发起 GET 请求，带超时限制（30 秒）
            response = await client.get(url, headers=headers, timeout=30.0)
            
            # 如果响应状态码不是 2xx，会自动抛出异常
            response.raise_for_status()
            
            # 返回解析后的 JSON 数据
            return response.json()
        
        except Exception:
            # 捕获所有异常（包括网络错误、超时、解析失败等）
            return None


# ==============================
# 4. 警报信息格式化函数
# ==============================

def format_alert(feature: dict) -> str:
    """将 NWS 返回的警报 feature 转换为格式化文本字符串。"""
    
    # 从返回数据中提取主要属性
    props = feature["properties"]
    
    # 返回多行可读字符串
    return f"""
Event: {props.get('event', 'Unknown')}                      # 事件名称（如 Tornado Warning）
Area: {props.get('areaDesc', 'Unknown')}                    # 影响区域
Severity: {props.get('severity', 'Unknown')}                # 严重程度
Description: {props.get('description', 'No description available')}  # 详细描述
Instructions: {props.get('instruction', 'No specific instructions provided')}  # 官方指令或建议
"""


# ==============================
# 5. MCP 工具 1：获取州警报
# ==============================

@mcp.tool()
async def get_alerts(state: str) -> str:
    """获取指定美国州（两位代码）的天气警报信息。
    
    Args:
        state: 州代码（如 CA, NY, TX 等）
    """
    # 构造 NWS 警报接口 URL
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    
    # 发起异步请求
    data = await make_nws_request(url)

    # 错误处理：请求失败或数据结构不符合预期
    if not data or "features" not in data:
        return "Unable to fetch alerts or no alerts found."

    # 若该州没有任何活跃警报
    if not data["features"]:
        return "No active alerts for this state."

    # 格式化所有警报内容
    alerts = [format_alert(feature) for feature in data["features"]]

    # 拼接输出文本，用 "---" 分隔多个警报
    return "\n---\n".join(alerts)


# ==============================
# 6. MCP 工具 2：获取天气预报
# ==============================

@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """根据经纬度坐标获取天气预报。
    
    Args:
        latitude: 纬度
        longitude: 经度
    """
    # 第一步：根据坐标查询对应的天气预报接口地址
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return "Unable to fetch forecast data for this location."

    # 提取 forecast 预报接口 URL
    forecast_url = points_data["properties"]["forecast"]

    # 第二步：获取具体的天气预报数据
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "Unable to fetch detailed forecast."

    # 提取各个时间段的天气信息
    periods = forecast_data["properties"]["periods"]

    forecasts = []
    for period in periods[:5]:  # 仅显示最近 5 个预报时段
        forecast = f"""
{period['name']}:
Temperature: {period['temperature']}°{period['temperatureUnit']}
Wind: {period['windSpeed']} {period['windDirection']}
Forecast: {period['detailedForecast']}
"""
        forecasts.append(forecast)

    # 拼接为最终输出字符串
    return "\n---\n".join(forecasts)


# ==============================
# 7. 启动 MCP 服务器
# ==============================

def main():
    """运行 MCP 服务（通过标准输入输出与客户端通信）"""
    mcp.run(transport='stdio')


# ==============================
# 8. 程序入口
# ==============================

if __name__ == "__main__":
    main()  # 启动服务

```



### 运行

```
# ~/.claude.json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
        "--directory",
        "/data/xiong/prj/mcp-quickstart/weather",
        "run",
        "weather.py"
      ]
    }
  }
}
```

```go
# 调用claude code 即可使用mcp
```







## 构建mcp客户端
