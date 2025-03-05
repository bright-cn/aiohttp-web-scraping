# 使用 AIOHTTP 在 Python 中进行网页抓取

[![Promo](https://github.com/bright-cn/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://www.bright.cn) 

本指南将介绍在 Python 中使用 AIOHTTP 进行网页抓取的基础知识。

- [什么是 AIOHTTP？](#什么是-aiohttp)
- [使用 AIOHTTP 进行抓取：分步教程](#使用-aiohttp-进行抓取分步教程)
  - [步骤 #1：设置抓取项目](#步骤-1设置抓取项目)
  - [步骤 #2：设置抓取所需的库](#步骤-2设置抓取所需的库)
  - [步骤 #3：获取目标页面的 HTML](#步骤-3获取目标页面的-html)
  - [步骤 #4：解析 HTML](#步骤-4解析-html)
  - [步骤 #5：编写数据提取逻辑](#步骤-5编写数据提取逻辑)
  - [步骤 #6：导出抓取到的数据](#步骤-6导出抓取到的数据)
  - [步骤 #7：将所有步骤整合到一起](#步骤-7将所有步骤整合到一起)
- [AIOHTTP 用于网页抓取：高级功能和技巧](#aiohttp-用于网页抓取高级功能和技巧)
  - [设置自定义 Headers](#设置自定义-headers)
  - [设置自定义 User Agent](#设置自定义-user-agent)
  - [设置 Cookies](#设置-cookies)
  - [代理集成](#代理集成)
  - [错误处理](#错误处理)
  - [重试失败的请求](#重试失败的请求)
- [AIOHTTP 与 Requests 在网页抓取上的比较](#aiohttp-与-requests-在网页抓取上的比较)
- [结论](#结论)

## 什么是 AIOHTTP？

[AIOHTTP](https://docs.aiohttp.org/en/stable/) 是一个基于 Python [`asyncio`](https://docs.python.org/3/library/asyncio.html) 库的异步客户端/服务器 HTTP 框架。与传统的 HTTP 客户端不同，AIOHTTP 使用客户端会话来管理多次请求之间的连接，非常适合高并发、基于会话的任务。

**⚙️ 特性**

- 同时支持客户端和服务器的 HTTP 协议实现。  
- 原生支持 WebSockets（客户端和服务器）。  
- 提供中间件和可插拔路由以构建 Web 服务器。  
- 高效管理大规模数据流。  
- 包含客户端会话持久化，可重复使用连接，从而在大量请求时减少开销。  

## 使用 AIOHTTP 进行抓取：分步教程

在网页抓取场景中，AIOHTTP 只是获取页面原始 HTML 内容的 HTTP 客户端。要解析并从该 HTML 中提取数据，需要使用类似 [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) 的 HTML 解析库。

> **注意**：  
> 尽管 AIOHTTP 主要用于抓取过程的初始阶段，本指南将介绍整个抓取流程。如果你想要了解更多 AIOHTTP 在网页抓取中的高级用法，可以在完成第 3 步后直接跳到下一章节。

### 步骤 #1：设置抓取项目

安装 Python3+，并为 AIOHTTP 抓取项目创建一个目录：

```bash
mkdir aiohttp-scraper
```

进入该目录，并设置一个 [虚拟环境](https://docs.python.org/3/library/venv.html)：

```bash
cd aiohttp-scraper
python -m venv env
```

在你喜欢的 Python IDE 中打开该项目文件夹，并在项目文件夹中创建一个名为 `scraper.py` 的文件。

在 IDE 的终端里激活虚拟环境。对于 Linux 或 macOS，使用：

```bash
./env/bin/activate
```

如果使用 Windows，运行：

```powershell
env/Scripts/activate
```

### 步骤 #2：设置抓取所需的库

安装 AIOHTTP 和 BeautifulSoup：

```bash
pip install aiohttp beautifulsoup4
```

在 `scraper.py` 中导入刚才安装的 [`aiohttp`](https://docs.aiohttp.org/en/stable/) 和 [`beautifulsoup4`](https://pypi.org/project/beautifulsoup4/) 依赖：

```python
import asyncio
import aiohttp 
from bs4 import BeautifulSoup
```

> **注意**：  
> `aiohttp` 需要依赖 `asyncio` 来运行。

现在，在 `scrper.py` 文件中添加如下的 `async` 函数流程：

```python
async def scrape_quotes():
    # Scraping logic...

# Run the asynchronous function
asyncio.run(scrape_quotes())
```

`scrape_quotes()` 定义了一段异步函数用于执行抓取逻辑，能够在并发环境下执行且不会阻塞。最后通过 `asyncio.run(scrape_quotes())` 启动并运行该异步函数。

### 步骤 #3：获取目标页面的 HTML

接下来以 [“Quotes to Scrape”](https://quotes.toscrape.com/) 网站为例：

![目标网站](https://github.com/bright-cn/aiohttp-web-scraping/blob/main/Images/s_C07E0B72CB9153F9B6E6EF6B76FDCD439C9910ACC1C4E94E70E103EE716CD2E2_1737465124750_image.png)

使用 Requests 或 AIOHTTP 等库时，直接发起 GET 请求即可获取页面的 HTML 内容。不过，与传统的同步请求库不同，AIOHTTP 基于 [不同的请求生命周期](https://docs.aiohttp.org/en/stable/http_request_lifecycle.html)。

AIOHTTP 的核心组件是 [`ClientSession`](https://docs.aiohttp.org/en/stable/client_reference.html)，它管理一个连接池并默认支持 [`Keep-Alive`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Keep-Alive)。也就是说，AIOHTTP 不会在每次请求时都新建连接，而是会重用现有连接，从而提高性能。

发起一次请求具体一般包括三个关键步骤：

1. 通过 `ClientSession()` 打开一个会话。
2. 使用 [`session.get()`](https://docs.aiohttp.org/en/stable/client_reference.html#aiohttp.ClientSession.get) 异步发送 GET 请求。
3. 使用 `await response.text()` 等方法获取响应数据。

这种设计让事件循环在不同的 [`with` 上下文](https://docs.python.org/3/reference/datamodel.html#context-managers)之间调度执行，不会被阻塞，非常适合高并发场景。

以下是使用 AIOHTTP 抓取首页 HTML 的示例：

```python
async with aiohttp.ClientSession() as session:
    async with session.get("http://quotes.toscrape.com") as response:
        # Access the HTML of the target page
        html = await response.text()
```

在幕后，AIOHTTP 会将请求发送给服务器并等待服务器的响应，这其中就包含页面的 HTML 内容。接收到响应后，`await response.text()` 将把这段 HTML 内容作为字符串返回。

如果打印 `html` 变量，你会看到类似：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Quotes to Scrape</title>
    <link rel="stylesheet" href="/static/bootstrap.min.css">
    <link rel="stylesheet" href="/static/main.css">
</head>
<body>
    <!-- 省略部分... -->
</body>
</html>
```

### 步骤 #4：解析 HTML

将 HTML 内容传给 BeautifulSoup 构造器解析：

```python
# Parse the HTML content using BeautifulSoup
soup = BeautifulSoup(html, "html.parser")
```

[`html.parser`](https://docs.python.org/3/library/html.parser.html) 是默认的 Python HTML 解析器，用来解析字符串中的 HTML 内容。

`soup` 对象中就包含了已解析的 HTML，并提供了丰富的方法来提取所需数据。

### 步骤 #5：编写数据提取逻辑

以下示例展示了如何从页面中抓取名言（quotes）数据：

```python
# Where to store the scraped data
quotes = []

# Extract all quotes from the page
quote_elements = soup.find_all("div", class_="quote")

# Loop through quotes and extract text, author, and tags
for quote_element in quote_elements:
    text = quote_element.find("span", class_="text").get_text().get_text().replace("“", "").replace("”", "")
    author = quote_element.find("small", class_="author")
    tags = [tag.get_text() for tag in quote_element.find_all("a", class_="tag")]

    # Store the scraped data
    quotes.append({
        "text": text,
        "author": author,
        "tags": tags
    })
```

上面的代码首先创建一个名为 `quotes` 的列表用于存储抓取结果。之后，它获取页面的所有“名言”HTML 元素，逐一遍历并提取名言文本、作者和标签等细节。每条名言信息以字典的形式存入 `quotes` 列表，方便后续访问或导出。

### 步骤 #6：导出抓取到的数据

以下代码展示了如何将抓取到的数据导出为 CSV 文件：

```python
# Open the file for export
with open("quotes.csv", mode="w", newline="", encoding="utf-8") as file:
    writer = csv.DictWriter(file, fieldnames=["text", "author", "tags"])
    
    # Write the header row
    writer.writeheader()
    
    # Write the scraped quotes data
    writer.writerows(quotes)
```

上述代码会打开名为 `quotes.csv` 的文件，并设置列名（`text`、`author`、`tags`），写入表头后，将 `quotes` 列表中的每个字典写入 CSV 文件。

使用 [`csv.DictWriter`](https://docs.python.org/3/library/csv.html#csv.DictWriter) 可以简化数据格式化的过程，更方便地存储结构化数据。要使用该功能，需要先从标准库中导入 `csv`：

```python
import csv
```

### 步骤 #7：将所有步骤整合到一起

下面是一段完整的 AIOHTTP 网页抓取示例脚本：

```python
import asyncio
import aiohttp
from bs4 import BeautifulSoup
import csv

# Define an asynchronous function to make the HTTP GET request
async def scrape_quotes():
    async with aiohttp.ClientSession() as session:
        async with session.get("http://quotes.toscrape.com") as response:
            # Access the HTML of the target page
            html = await response.text()

            # Parse the HTML content using BeautifulSoup
            soup = BeautifulSoup(html, "html.parser")

            # List to store the scraped data
            quotes = []

            # Extract all quotes from the page
            quote_elements = soup.find_all("div", class_="quote")

            # Loop through quotes and extract text, author, and tags
            for quote_element in quote_elements:
                text = quote_element.find("span", class_="text").get_text().replace("“", "").replace("”", "")
                author = quote_element.find("small", class_="author").get_text()
                tags = [tag.get_text() for tag in quote_element.find_all("a", class_="tag")]

                # Store the scraped data
                quotes.append({
                    "text": text,
                    "author": author,
                    "tags": tags
                })

            # Open the file name for export
            with open("quotes.csv", mode="w", newline="", encoding="utf-8") as file:
                writer = csv.DictWriter(file, fieldnames=["text", "author", "tags"])

                # Write the header row
                writer.writeheader()

                # Write the scraped quotes data
                writer.writerows(quotes)

# Run the asynchronous function
asyncio.run(scrape_quotes())
```

运行方式：

```bash
python scraper.py
```

或在 Linux/macOS 系统下：

```bash
python3 scraper.py
```

然后在项目根目录下就会生成一个 `quotes.csv` 文件。打开后可见：

![最终的 quotes 文件](https://github.com/bright-cn/aiohttp-web-scraping/blob/main/Images/s_C07E0B72CB9153F9B6E6EF6B76FDCD439C9910ACC1C4E94E70E103EE716CD2E2_1737466185816_image.png)

## AIOHTTP 用于网页抓取：高级功能和技巧

在接下来的示例中，目标网站换成了 [HTTPBin.io `/anything` endpoint](https://httpbin.io/anything)。该 API 会返回请求者的 IP 地址、Headers 以及其他信息。

### 设置自定义 Headers

可以在 AIOHTTP 中通过 `headers` 参数来 [设置自定义 Headers](https://docs.aiohttp.org/en/stable/client_advanced.html#custom-request-headers)：

```python
import aiohttp
import asyncio

async def fetch_with_custom_headers():
    # Custom headers for the request
    headers = {
        "Accept": "application/json",
        "Accept-Language": "en-US,en;q=0.9,fr-FR;q=0.8,fr;q=0.7,es-US;q=0.6,es;q=0.5,it-IT;q=0.4,it;q=0.3"
    }

    async with aiohttp.ClientSession() as session:
        # Make a GET request with custom headers
        async with session.get("https://httpbin.io/anything", headers=headers) as response:
            data = await response.json()
            # Handle the response...
            print(data)

# Run the event loop
asyncio.run(fetch_with_custom_headers())
```

通过这种方式，AIOHTTP 会携带 [`Accept`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept) 和 [`Accept-Language`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language) 等自定义头部进行 HTTP GET 请求。

### 设置自定义 User Agent

[`User-Agent`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent) 是 [网页抓取中最常见的 HTTP 头部之一](https://www.bright.cn/blog/web-data/http-headers-for-web-scraping)。AIOHTTP 默认为以下 `User-Agent`：

```
Python/<PYTHON_VERSION> aiohttp/<AIOHTTP_VERSION>
```

默认为上述值时，很容易暴露请求来自自动化脚本，因而被目标站点拦截。可以通过设置更像常见浏览器的 `User-Agent` 来降低被拦截的几率：

```python
import aiohttp
import asyncio

async def fetch_with_custom_user_agent():
    # Define a Chrome-like custom User-Agent
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36"
    }

    async with aiohttp.ClientSession(headers=headers) as session:
        # Make a GET request with the custom User-Agent
        async with session.get("https://httpbin.io/anything") as response:
            data = await response.text()
            # Handle the response...
            print(data)

# Run the event loop
asyncio.run(fetch_with_custom_user_agent())
```

### 设置 Cookies

与设置 HTTP 头部类似，你也可以在 `ClientSession()` 中通过 `cookies` 字典 [设置自定义 Cookies](https://docs.aiohttp.org/en/v3.7.3/client_advanced.html#custom-cookies)：

```python
import aiohttp
import asyncio

async def fetch_with_custom_cookies():
    # Define cookies as a dictionary
    cookies = {
        "session_id": "9412d7hdsa16hbda4347dagb",
        "user_preferences": "dark_mode=false"
    }

    async with aiohttp.ClientSession(cookies=cookies) as session:
        # Make a GET request with custom cookies
        async with session.get("https://httpbin.io/anything") as response:
            data = await response.text()
            # Handle the response...
            print(data)

# Run the event loop
asyncio.run(fetch_with_custom_cookies())
```

Cookies 用来在请求中携带会话信息，对于网页抓取来说尤为重要。

> **注意**：  
> 在 `ClientSession` 中设置的 Cookies 会在该会话下所有请求中共享。如需访问会话的 Cookies，可参考 [`ClientSession.cookie_jar`](https://docs.aiohttp.org/en/v3.7.3/client_reference.html#aiohttp.ClientSession.cookie_jar)。

### 代理集成

在 AIOHTTP 中，可以通过 [`proxy` 参数](https://docs.aiohttp.org/en/v3.7.3/client_advanced.html#proxy-support) 将请求转发到代理服务器，从而降低 IP 被封禁的风险。示例：

```python
import aiohttp
import asyncio

async def fetch_through_proxy():
    # Replace with the URL of your proxy server
    proxy_url = "<YOUR_PROXY_URL>"

    async with aiohttp.ClientSession() as session:
        # Make a GET request through the proxy server
        async with session.get("https://httpbin.io/anything", proxy=proxy_url) as response:
            data = await response.text()
            # Handle the response...
            print(data)

# Run the event loop
asyncio.run(fetch_through_proxy())
```

### 错误处理

默认为网络或连接问题时，AIOHTTP 才会抛出异常。要让其在接收 [`4xx`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses) 和 [`5xx`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#server_error_responses) 状态码时抛出异常，可以采用以下方式之一：

1. **在创建 `ClientSession` 时指定 `raise_for_status=True`**：对所有通过该会话发起的请求，在遇到 `4xx` 或 `5xx` 状态码时自动抛出异常。  
2. **在请求方法中单独传入 `raise_for_status=True`**：只对某个请求生效，而不影响其他请求。  
3. **手动调用 `response.raise_for_status()`**：在收到响应后决定是否抛出异常，拥有更细粒度的掌控权。

方式 #1 示例：

```python
import aiohttp
import asyncio

async def fetch_with_session_error_handling():
    async with aiohttp.ClientSession(raise_for_status=True) as session:
        try:
            async with session.get("https://httpbin.io/anything") as response:
                # No need to call response.raise_for_status(), as it is automatic
                data = await response.text()
                print(data)
        except aiohttp.ClientResponseError as e:
            print(f"HTTP error occurred: {e.status} - {e.message}")
        except aiohttp.ClientError as e:
            print(f"Request error occurred: {e}")

# Run the event loop
asyncio.run(fetch_with_session_error_handling())
```

当在会话层面设置了 `raise_for_status=True`，如果服务器返回 `4xx` 或 `5xx`，就会抛出 `aiohttp.ClientResponseError`。

方式 #2 示例：

```python
import aiohttp
import asyncio

async def fetch_with_raise_for_status():
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get("https://httpbin.io/anything", raise_for_status=True) as response:
                # No need to manually call response.raise_for_status(), it is automatic
                data = await response.text()
                print(data)
        except aiohttp.ClientResponseError as e:
            print(f"HTTP error occurred: {e.status} - {e.message}")
        except aiohttp.ClientError as e:
            print(f"Request error occurred: {e}")

# Run the event loop
asyncio.run(fetch_with_raise_for_status())
```

这里将 `raise_for_status=True` 直接传给 `session.get()`，只对该请求启用自动抛出异常。

方式 #3 示例：

```python
import aiohttp
import asyncio

async def fetch_with_manual_error_handling():
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get("https://httpbin.io/anything") as response:
                response.raise_for_status()  # Manually raises error for 4xx/5xx
                data = await response.text()
                print(data)
        except aiohttp.ClientResponseError as e:
            print(f"HTTP error occurred: {e.status} - {e.message}")
        except aiohttp.ClientError as e:
            print(f"Request error occurred: {e}")

# Run the event loop
asyncio.run(fetch_with_manual_error_handling())
```

如果你想在每个请求完成后再决定是否抛出异常，可以通过手动调用 `response.raise_for_status()` 的方式。

### 重试失败的请求

AIOHTTP 本身并未提供自动重试的功能，需要自行编写逻辑或借助第三方库（如 [`aiohttp-retry`](https://github.com/inyutin/aiohttp_retry)）。该库允许你在请求失败时进行配置化的重试机制，以应对网络波动、超时或限流等问题。

安装 [`aiohttp-retry`](https://pypi.org/project/aiohttp-retry/)：

```bash
pip install aiohttp-retry
```

示例使用方式：

```python
import asyncio
from aiohttp_retry import RetryClient, ExponentialRetry

async def main():
    retry_options = ExponentialRetry(attempts=1)
    retry_client = RetryClient(raise_for_status=False, retry_options=retry_options)
    async with retry_client.get("https://httpbin.io/anything") as response:
        print(response.status)
        
    await retry_client.close()
```

这里配置了指数退避的重试策略。[官方文档](https://github.com/inyutin/aiohttp_retry?tab=readme-ov-file#documentation)中有更详细的说明。

## AIOHTTP 与 Requests 在网页抓取上的比较

下表总结了 AIOHTTP 与 [Requests](https://www.bright.cn/blog/web-data/python-requests-guide) 在网页抓取上的主要差异：

| **特性**                  | **AIOHTTP** | **Requests** |
|---------------------------|------------|--------------|
| **GitHub Stars**          | 15.3k      | 52.4k        |
| **客户端支持**            | ✔️          | ✔️            |
| **同步支持**              | ❌          | ✔️            |
| **异步支持**              | ✔️          | ❌            |
| **服务器支持**            | ✔️          | ❌            |
| **连接池**                | ✔️          | ✔️            |
| **HTTP/2 支持**           | ❌          | ❌            |
| **自定义 User-Agent**     | ✔️          | ✔️            |
| **代理支持**              | ✔️          | ✔️            |
| **Cookie 处理**          | ✔️          | ✔️            |
| **重试机制**             | 仅通过三方库提供     | 通过 `HTTPAdapter` 可用 |
| **性能**                  | 高          | 中            |
| **社区支持与人气**        | 中          | 大            |

更多对比可参考我们的博文 [Requests vs HTTPX vs AIOHTTP](https://www.bright.cn/blog/web-data/requests-vs-httpx-vs-aiohttp)。

## 结论

AIOHTTP 在获取在线数据方面速度快且稳定。但需要注意的是，自动化的 HTTP 请求会暴露你的公网 IP 地址。为保护隐私与安全，可以考虑使用 Bright Data 的代理服务器来隐藏 IP 地址。

- [数据中心代理](https://www.bright.cn/proxy-types/datacenter-proxies) – 超过 770,000 个数据中心 IP。  
- [住宅代理](https://www.bright.cn/proxy-types/residential-proxies) – 超过 72M 个住宅 IP，覆盖 195+ 个国家和地区。  
- [ISP 代理](https://www.bright.cn/proxy-types/isp-proxies) – 超过 700,000 个 ISP IP。  
- [移动代理](https://www.bright.cn/proxy-types/mobile-proxies) – 超过 7M 个移动 IP。  

立即创建一个免费的 Bright Data 账号，试用我们的代理和抓取解决方案吧！
