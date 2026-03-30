# `fetch_one_video` 完整调用流程详解

本文档详细分析 `app/api/endpoints/douyin_web.py` 第14行 `fetch_one_video` 接口的完整调用流程。

---

## 📊 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              用户请求                                            │
│                    GET /api/douyin/web/fetch_one_video?aweme_id=xxx             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  第1层: FastAPI 路由层                                                           │
│  app/api/endpoints/douyin_web.py                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ @router.get("/fetch_one_video")                                          │   │
│  │ async def fetch_one_video(aweme_id: str)                                 │   │
│  │     → 调用 DouyinWebCrawler.fetch_one_video(aweme_id)                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  第2层: 爬虫业务层                                                               │
│  crawlers/douyin/web/web_crawler.py                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ class DouyinWebCrawler:                                                  │   │
│  │   async def fetch_one_video(aweme_id):                                   │   │
│  │     1. get_douyin_headers() → 获取请求头和Cookie                          │   │
│  │     2. PostDetail(aweme_id) → 构建请求参数模型                            │   │
│  │     3. BogusManager.ab_model_2_endpoint() → 生成A-Bogus加密参数           │   │
│  │     4. BaseCrawler.fetch_get_json(endpoint) → 发起HTTP请求               │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  第3层: 基础爬虫层                                                               │
│  crawlers/base_crawler.py                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ class BaseCrawler:                                                       │   │
│  │   - httpx.AsyncClient 异步HTTP客户端                                      │   │
│  │   - 重试机制 (max_retries=3)                                              │   │
│  │   - 连接池管理 (max_connections=50)                                       │   │
│  │   - 超时控制 (timeout=10s)                                                │   │
│  │   - JSON响应解析                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  第4层: 抖音API服务器                                                            │
│  https://www.douyin.com/aweme/v1/web/aweme/detail/                              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 详细流程分解

### **第1步：FastAPI 路由接收请求**

**文件**: `app/api/endpoints/douyin_web.py` (第14-48行)

```python
@router.get("/fetch_one_video", response_model=ResponseModel, summary="获取单个作品数据")
async def fetch_one_video(request: Request,
                          aweme_id: str = Query(example="7372484719365098803", description="作品id")):
    try:
        # 调用爬虫层获取数据
        data = await DouyinWebCrawler.fetch_one_video(aweme_id)
        return ResponseModel(code=200, router=request.url.path, data=data)
    except Exception as e:
        # 错误处理
        raise HTTPException(status_code=400, detail=ErrorResponseModel(...))
```

**作用**：
- 定义 HTTP GET 接口 `/fetch_one_video`
- 接收 `aweme_id` 查询参数（视频ID）
- 调用爬虫层并封装响应

---

### **第2步：获取请求头配置**

**文件**: `crawlers/douyin/web/web_crawler.py` (第73-84行)

```python
async def get_douyin_headers(self):
    douyin_config = config["TokenManager"]["douyin"]
    kwargs = {
        "headers": {
            "Accept-Language": douyin_config["headers"]["Accept-Language"],
            "User-Agent": douyin_config["headers"]["User-Agent"],
            "Referer": douyin_config["headers"]["Referer"],
            "Cookie": douyin_config["headers"]["Cookie"],  # 关键：用户Cookie
        },
        "proxies": {
            "http://": douyin_config["proxies"]["http"],
            "https://": douyin_config["proxies"]["https"]
        },
    }
    return kwargs
```

**配置来源**: `crawlers/douyin/web/config.yaml`

```yaml
TokenManager:
  douyin:
    headers:
      Accept-Language: zh-CN,zh;q=0.8...
      User-Agent: Mozilla/5.0 (Windows NT 10.0...)
      Referer: https://www.douyin.com/
      Cookie: __ac_nonce=xxx; ttwid=xxx; ...  # 必须配置有效Cookie
    proxies:
      http:    # 可选代理
      https:
```

---

### **第3步：构建请求参数模型**

**文件**: `crawlers/douyin/web/models.py` (第184-185行)

```python
class PostDetail(BaseRequestModel):
    aweme_id: str  # 视频ID
```

**继承自 BaseRequestModel** (第8-42行)，包含大量默认参数：

```python
class BaseRequestModel(BaseModel):
    device_platform: str = "webapp"
    aid: str = "6383"
    channel: str = "channel_pc_web"
    pc_client_type: int = 1
    version_code: str = "290100"
    version_name: str = "29.1.0"
    cookie_enabled: str = "true"
    screen_width: int = 1920
    screen_height: int = 1080
    browser_language: str = "zh-CN"
    browser_platform: str = "Win32"
    browser_name: str = "Chrome"
    browser_version: str = "130.0.0.0"
    # ... 更多浏览器指纹参数
    msToken: str = TokenManager.gen_real_msToken()  # 动态生成msToken
```

**生成的参数字典示例**：
```python
{
    "aweme_id": "7372484719365098803",
    "device_platform": "webapp",
    "aid": "6383",
    "channel": "channel_pc_web",
    "version_code": "290100",
    "screen_width": 1920,
    "screen_height": 1080,
    "browser_name": "Chrome",
    "msToken": "xxxxxxx...",  # 128位动态Token
    # ... 约30个参数
}
```

---

### **第4步：生成 A-Bogus 加密签名**

**文件**: `crawlers/douyin/web/web_crawler.py` (第104-107行)

```python
# 生成A-Bogus加密参数
params_dict = params.dict()
params_dict["msToken"] = ''  # 清空msToken用于签名计算
a_bogus = BogusManager.ab_model_2_endpoint(params_dict, kwargs["headers"]["User-Agent"])
endpoint = f"{DouyinAPIEndpoints.POST_DETAIL}?{urlencode(params_dict)}&a_bogus={a_bogus}"
```

**BogusManager 签名生成** (`crawlers/douyin/web/utils.py` 第294-304行)：

```python
class BogusManager:
    @classmethod
    def ab_model_2_endpoint(cls, params: dict, user_agent: str) -> str:
        if not isinstance(params, dict):
            raise TypeError("参数必须是字典类型")
        try:
            ab_value = AB().get_value(params)  # 调用ABogus算法
        except Exception as e:
            raise RuntimeError("生成A-Bogus失败: {0})".format(e))
        return quote(ab_value, safe='')
```

**A-Bogus 算法**: `crawlers/douyin/web/abogus.py` - 这是抖音的反爬签名算法，用于验证请求合法性。

**最终生成的请求URL**：
```
https://www.douyin.com/aweme/v1/web/aweme/detail/
  ?aweme_id=7372484719365098803
  &device_platform=webapp
  &aid=6383
  &channel=channel_pc_web
  &version_code=290100
  &screen_width=1920
  &screen_height=1080
  &browser_name=Chrome
  &msToken=
  &a_bogus=DFSzswSO...  # 加密签名
```

---

### **第5步：创建异步HTTP客户端**

**文件**: `crawlers/base_crawler.py` (第56-102行)

```python
class BaseCrawler:
    def __init__(
            self,
            proxies: dict = None,
            max_retries: int = 3,        # 重试3次
            max_connections: int = 50,   # 最大50连接
            timeout: int = 10,           # 10秒超时
            max_tasks: int = 50,         # 最大50并发任务
            crawler_headers: dict = {},
    ):
        self.proxies = proxies
        self.crawler_headers = crawler_headers

        # 信号量控制并发
        self.semaphore = asyncio.Semaphore(max_tasks)

        # 连接池限制
        self.limits = httpx.Limits(max_connections=max_connections)

        # 底层连接重试
        self.atransport = httpx.AsyncHTTPTransport(retries=max_retries)

        # 超时配置
        self.timeout = httpx.Timeout(timeout)

        # 创建异步HTTP客户端
        self.aclient = httpx.AsyncClient(
            headers=self.crawler_headers,
            proxies=self.proxies,
            timeout=self.timeout,
            limits=self.limits,
            transport=self.atransport,
        )
```

---

### **第6步：发起HTTP请求并获取响应**

**文件**: `crawlers/base_crawler.py` (第115-125行, 第174-215行)

```python
async def fetch_get_json(self, endpoint: str) -> dict:
    """获取 JSON 数据"""
    response = await self.get_fetch_data(endpoint)
    return self.parse_json(response)

async def get_fetch_data(self, url: str):
    """获取GET端点数据，带重试机制"""
    for attempt in range(self._max_retries):
        try:
            response = await self.aclient.get(url, follow_redirects=True)

            # 检查响应是否为空
            if not response.text.strip() or not response.content:
                logger.warning(f"第 {attempt + 1} 次响应内容为空")
                if attempt == self._max_retries - 1:
                    raise APIRetryExhaustedError("获取端点数据失败, 次数达到上限")
                await asyncio.sleep(self._timeout)
                continue

            response.raise_for_status()
            return response

        except httpx.RequestError:
            raise APIConnectionError("连接端点失败，检查网络环境或代理")
        except httpx.HTTPStatusError as http_error:
            self.handle_http_status_error(http_error, url, attempt + 1)
```

---

### **第7步：解析JSON响应**

**文件**: `crawlers/base_crawler.py` (第139-172行)

```python
def parse_json(self, response: Response) -> dict:
    """解析JSON响应对象"""
    if response is not None and response.status_code == 200:
        try:
            return response.json()
        except json.JSONDecodeError as e:
            # 尝试用正则提取JSON
            match = re.search(r"\{.*\}", response.text)
            try:
                return json.loads(match.group())
            except json.JSONDecodeError:
                raise APIResponseError("解析JSON数据失败")
    else:
        raise APIResponseError("获取数据失败")
```

---

### **第8步：返回响应给用户**

**文件**: `app/api/endpoints/douyin_web.py` (第39-41行)

```python
return ResponseModel(
    code=200,
    router=request.url.path,
    data=data  # 抖音API返回的视频详情数据
)
```

**响应数据结构示例**：
```json
{
  "code": 200,
  "router": "/api/douyin/web/fetch_one_video",
  "data": {
    "status_code": 0,
    "aweme_detail": {
      "aweme_id": "7372484719365098803",
      "desc": "视频描述...",
      "author": {
        "nickname": "用户昵称",
        "uid": "123456789",
        "sec_uid": "MS4wLjAB..."
      },
      "video": {
        "play_addr": {
          "url_list": ["https://...无水印视频地址..."]
        },
        "cover": {...}
      },
      "statistics": {
        "digg_count": 12345,
        "comment_count": 678,
        "share_count": 90
      },
      "create_time": 1699999999
    }
  }
}
```

---

## 🔐 关键安全机制

### 1. **msToken 生成**
```python
# crawlers/douyin/web/utils.py (第89-151行)
class TokenManager:
    @classmethod
    def gen_real_msToken(cls) -> str:
        # 向字节跳动服务器请求真实msToken
        payload = {
            "magic": 538969122,
            "version": 1,
            "dataType": 8,
            "strData": "...",  # 加密数据
            "tspFromClient": get_timestamp(),
        }
        response = client.post("https://mssdk.bytedance.com/web/report", ...)
        return response.cookies.get("msToken")  # 返回128位Token
```

### 2. **A-Bogus 签名算法**
- 位于 `crawlers/douyin/web/abogus.py`
- 基于请求参数和User-Agent生成加密签名
- 抖音服务器用此验证请求是否来自合法客户端

### 3. **Cookie 验证**
- 必须提供有效的抖音网页版Cookie
- Cookie包含用户身份信息和会话状态
- Cookie失效会导致请求被拒绝

---

## 📁 涉及文件汇总

| 文件路径 | 作用 |
|---------|------|
| `app/api/endpoints/douyin_web.py` | FastAPI路由定义 |
| `crawlers/douyin/web/web_crawler.py` | 抖音爬虫业务逻辑 |
| `crawlers/douyin/web/models.py` | Pydantic请求参数模型 |
| `crawlers/douyin/web/endpoints.py` | 抖音API端点URL定义 |
| `crawlers/douyin/web/utils.py` | 工具类(Token/签名/ID提取) |
| `crawlers/douyin/web/abogus.py` | A-Bogus签名算法 |
| `crawlers/douyin/web/config.yaml` | Cookie和代理配置 |
| `crawlers/base_crawler.py` | 基础HTTP客户端封装 |

---

## 🔄 完整时序图

```
┌──────┐     ┌─────────────┐     ┌────────────────┐     ┌────────────┐     ┌──────────┐
│Client│     │FastAPI Route│     │DouyinWebCrawler│     │BaseCrawler │     │Douyin API│
└──┬───┘     └──────┬──────┘     └───────┬────────┘     └─────┬──────┘     └────┬─────┘
   │                │                    │                    │                 │
   │ GET /fetch_one_video?aweme_id=xxx   │                    │                 │
   │───────────────>│                    │                    │                 │
   │                │                    │                    │                 │
   │                │ fetch_one_video()  │                    │                 │
   │                │───────────────────>│                    │                 │
   │                │                    │                    │                 │
   │                │                    │ get_douyin_headers()                 │
   │                │                    │────────┐           │                 │
   │                │                    │        │ 读取config.yaml             │
   │                │                    │<───────┘           │                 │
   │                │                    │                    │                 │
   │                │                    │ PostDetail(aweme_id)                 │
   │                │                    │────────┐           │                 │
   │                │                    │        │ 构建参数模型                │
   │                │                    │<───────┘           │                 │
   │                │                    │                    │                 │
   │                │                    │ BogusManager.ab_model_2_endpoint()   │
   │                │                    │────────┐           │                 │
   │                │                    │        │ 生成A-Bogus签名             │
   │                │                    │<───────┘           │                 │
   │                │                    │                    │                 │
   │                │                    │ BaseCrawler()      │                 │
   │                │                    │───────────────────>│                 │
   │                │                    │                    │                 │
   │                │                    │ fetch_get_json()   │                 │
   │                │                    │───────────────────>│                 │
   │                │                    │                    │                 │
   │                │                    │                    │ GET /aweme/detail
   │                │                    │                    │────────────────>│
   │                │                    │                    │                 │
   │                │                    │                    │   JSON Response │
   │                │                    │                    │<────────────────│
   │                │                    │                    │                 │
   │                │                    │    parse_json()    │                 │
   │                │                    │<───────────────────│                 │
   │                │                    │                    │                 │
   │                │      data          │                    │                 │
   │                │<───────────────────│                    │                 │
   │                │                    │                    │                 │
   │ ResponseModel  │                    │                    │                 │
   │<───────────────│                    │                    │                 │
   │                │                    │                    │                 │
```

---

## ⚠️ 常见问题

### Q1: Cookie失效
**现象**: 返回空数据或错误码
**解决**: 更新 `crawlers/douyin/web/config.yaml` 中的Cookie

### Q2: 签名验证失败
**现象**: 返回 `status_code: 8` 或类似错误
**解决**: A-Bogus算法可能需要更新，检查 `abogus.py`

### Q3: 请求被限流
**现象**: 返回 429 状态码
**解决**:
- 配置代理
- 降低请求频率
- 使用已登录账号的Cookie

### Q4: 网络超时
**现象**: 请求长时间无响应
**解决**: 检查代理配置或网络环境