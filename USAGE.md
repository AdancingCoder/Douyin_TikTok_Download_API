# 📖 抖音/TikTok下载API - 使用手册

## 一、项目简介

**Douyin_TikTok_Download_API** 是一个免费开源的高性能异步API服务，支持以下平台的视频/图文数据采集和无水印下载：

| 平台 | 网址 |
|------|------|
| 抖音 | https://www.douyin.com |
| TikTok | https://www.tiktok.com |
| B站 | https://www.bilibili.com |

**主要功能：**
- 解析视频元数据（标题、作者、描述、统计数据）
- 下载无水印视频/图片
- 批量URL解析
- RESTful API接口
- Web图形界面
- iOS快捷指令集成

---

## 二、快速安装

### 方式一：Python直接运行

**1. 安装依赖**
```bash
pip install -r requirements.txt
```

**2. 启动服务**
```bash
python start.py
```

服务将在 `http://0.0.0.0:80` 启动

### 方式二：Docker部署

```bash
# 拉取镜像
docker pull evil0ctal/douyin_tiktok_download_api:latest

# 运行容器
docker run -d --name douyin_api -p 80:80 evil0ctal/douyin_tiktok_download_api
```

### 方式三：Linux一键部署

```bash
wget -O install.sh https://raw.githubusercontent.com/Evil0ctal/Douyin_TikTok_Download_API/main/bash/install.sh
sudo bash install.sh
```

---

## 三、配置说明

### 主配置文件：`config.yaml`

```yaml
# Web界面配置
Web:
  PyWebIO_Enable: true           # 启用Web界面
  Domain: https://douyin.wtf     # 你的域名
  PyWebIO_Theme: minty           # 界面主题
  Max_Take_URLs: 30              # 批量解析最大URL数

# API配置
API:
  Host_IP: 0.0.0.0               # 监听IP
  Host_Port: 80                  # 监听端口
  Download_Switch: true          # 启用下载功能
  Download_Path: "./download"    # 下载目录
  Download_File_Prefix: "douyin.wtf_"  # 文件名前缀
```

### 重要：更新Cookie

**必须更新抖音Cookie才能正常使用！**

编辑 `crawlers/douyin/web/config.yaml`：

```yaml
TokenManager:
  douyin:
    headers:
      Cookie: 你的抖音Cookie  # 从浏览器获取
```

**获取Cookie方法：**
1. 浏览器登录抖音网页版
2. 按F12打开开发者工具
3. 切换到Network标签
4. 刷新页面，找到任意请求
5. 复制Request Headers中的Cookie值

---

## 四、API使用指南

### 访问API文档

启动服务后访问：
- Swagger文档：`http://localhost/docs`
- ReDoc文档：`http://localhost/redoc`

### 常用API接口

#### 1. 混合解析（自动识别平台）

```
GET /api/hybrid/video_data?url={分享链接}
```

**示例：**
```bash
curl "http://localhost/api/hybrid/video_data?url=https://v.douyin.com/L4FJNR3/"
```

**响应：**
```json
{
  "code": 200,
  "data": {
    "platform": "douyin",
    "aweme_id": "视频ID",
    "title": "视频标题",
    "author": {...},
    "statistics": {...},
    "video_url": "无水印下载地址"
  }
}
```

#### 2. 视频下载

```
GET /api/download?url={分享链接}&prefix=true&with_watermark=false
```

直接返回视频文件流

#### 3. 抖音专用接口

| 接口 | 说明 |
|------|------|
| `/api/douyin/web/video_data?video_id=xxx` | 获取视频详情 |
| `/api/douyin/web/user_info?user_id=xxx` | 获取用户信息 |
| `/api/douyin/web/user_posts?user_id=xxx&cursor=0` | 获取用户作品列表 |
| `/api/douyin/web/user_likes?user_id=xxx` | 获取用户喜欢列表 |
| `/api/douyin/web/user_collection?user_id=xxx` | 获取用户收藏 |

#### 4. TikTok专用接口

| 接口 | 说明 |
|------|------|
| `/api/tiktok/web/video_data?video_id=xxx` | 获取视频详情 |
| `/api/tiktok/web/user_info?user_id=xxx` | 获取用户信息 |
| `/api/tiktok/web/user_posts?user_id=xxx` | 获取用户作品 |

#### 5. B站专用接口

| 接口 | 说明 |
|------|------|
| `/api/bilibili/web/video_data?bvid=xxx` | 获取视频详情 |
| `/api/bilibili/web/video_stream?bvid=xxx&cid=xxx` | 获取视频流地址 |
| `/api/bilibili/web/user_info?uid=xxx` | 获取用户信息 |

---

## 五、Web界面使用

访问 `http://localhost/` 进入Web界面

**功能：**
- **批量解析** - 粘贴多个链接（每行一个，最多30个）
- **实时下载** - 在线预览和下载视频/图片
- **iOS快捷指令** - 获取iOS快捷指令安装链接

---

## 六、iOS快捷指令

**安装地址：**
https://www.icloud.com/shortcuts/06f891a026df40cfa967a907feaea632

**使用方法：**
1. 在抖音/TikTok App中分享视频
2. 选择"快捷指令"
3. 自动下载无水印视频到相册

---

## 七、支持的链接格式

### 抖音
```
# 短链接
https://v.douyin.com/L4FJNR3/

# 完整链接
https://www.douyin.com/video/7145917813812840712

# 用户主页
https://www.douyin.com/user/MS4wLjABAAAA...
```

### TikTok
```
# 短链接
https://vm.tiktok.com/ZMxxxxxx/

# 完整链接
https://www.tiktok.com/@username/video/7145917813812840712
```

### B站
```
# BV号
https://www.bilibili.com/video/BV1xx411c7mD

# 短链接
https://b23.tv/xxxxxx
```

---

## 八、常见问题

### Q1: 提示Cookie失效？
更新 `crawlers/douyin/web/config.yaml` 中的Cookie

### Q2: 请求被限流？
- 使用已登录账号的Cookie
- 配置代理：
```yaml
proxies:
  http: http://127.0.0.1:7890
  https: http://127.0.0.1:7890
```

### Q3: TikTok无法访问？
需要海外网络环境或配置代理

### Q4: 如何修改端口？
编辑 `config.yaml`：
```yaml
API:
  Host_Port: 8080  # 改为你需要的端口
```

---

## 九、项目结构

```
├── app/                    # 应用层
│   ├── main.py            # FastAPI入口
│   ├── api/endpoints/     # API路由
│   └── web/views/         # Web界面
├── crawlers/              # 爬虫核心
│   ├── douyin/web/        # 抖音爬虫
│   ├── tiktok/web/        # TikTok爬虫
│   ├── bilibili/web/      # B站爬虫
│   └── hybrid/            # 混合解析器
├── config.yaml            # 主配置
├── requirements.txt       # 依赖
└── start.py              # 启动脚本
```

---

## 十、技术支持

- **GitHub**: https://github.com/Evil0ctal/Douyin_TikTok_Download_API
- **问题反馈**: https://github.com/Evil0ctal/Douyin_TikTok_Download_API/issues
- **在线演示**: https://douyin.wtf

**许可证**: Apache 2.0