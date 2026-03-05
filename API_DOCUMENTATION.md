# RestreamerOnJava API 文档

本文档详细描述了 RestreamerOnJava 应用程序的所有 REST API 端点。

## 目录

- [字幕管理 API](#字幕管理-api)
  - [POST /api/subtitles/process](#post-apisubtitlesprocess)
  - [POST /api/subtitles/session/start](#post-apisubtitlessessionstart)
  - [POST /api/subtitles/session/end](#post-apisubtitlessessionend)
  - [GET /api/subtitles](#get-apisubtitles)
  - [GET /api/subtitles/{filename}](#get-apisubtitlesfilename)
  - [GET /api/subtitles/{filename}/download](#get-apisubtitlesfilenamedownload)
  - [DELETE /api/subtitles/{filename}](#delete-apisubtitlesfilename)
  - [POST /api/subtitles/batch-delete](#post-apisubtitlesbatch-delete)
  - [GET /api/subtitles/filter/{type}](#get-apisubtitlesfiltertype)
  - [GET /api/subtitles/channel/{channelName}](#get-apisubtitleschannelchannelname)
- [频道状态 API](#频道状态-api)
  - [POST /api/channel/set](#post-apichannelset)
  - [GET /api/channel/get/byid/{channelId}](#get-apichannelgetbyidchannelid)
  - [GET /api/channel/get/name](#get-apichannelgetname)
  - [POST /api/channel/monitor/start](#post-apichannelmonitorstart)
  - [POST /api/channel/monitor/stop](#post-apichannelmonitorstop)
  - [GET /api/channel/monitor/status](#get-apichannelmonitorstatus)
  - [GET /api/channel/monitor/sse](#get-apichannelmonitorsse)
- [翻译服务 API](#翻译服务-api)
  - [GET /api/translation/models](#get-apitranslationmodels)
  - [POST /api/translation/translate/stream](#post-apitranslationtranslatestream)
  - [POST /api/translation/translate](#post-apitranslationtranslate)
  - [POST /api/translation/session/start](#post-apitranslationsessionstart)（已废弃）
  - [POST /api/translation/session/end](#post-apitranslationsessionend)（已废弃）
- [设置管理 API](#设置管理-api)
  - [GET /api/settings](#get-apisettings)
  - [PUT /api/settings/set](#put-apisettingsset)
  - [POST /api/settings/save](#post-apisettingssave)
- [OBS 控制 API](#obs 控制-api)
  - [PUT /api/obs/stream/start](#put-apiobsstreamstart)
  - [PUT /api/obs/stream/stop](#put-apiobsstreamstop)
  - [PUT /api/obs/record/start](#put-apiobsrecordstart)
  - [PUT /api/obs/record/stop](#put-apiobsrecordstop)
  - [PUT /api/obs/scene/set](#put-apiobsscen eset)
  - [POST /api/obs/media-monitor/start](#post-apiobsmedia-monitorstart)
  - [POST /api/obs/media-monitor/stop](#post-apiobsmedia-monitorstop)
  - [GET /api/obs/media-monitor/status](#get-apiobsmedia-monitorstatus)
- [字幕样式 API](#字幕样式-api)
  - [PUT /api/subtitles/save-style/subtitle](#put-apisubtitlessave-stylesubtitle)
  - [PUT /api/subtitles/save-style/history](#put-apisubtitlessave-stylehistory)
  - [GET /api/subtitles/get-style/subtitle](#get-apisubtitlesget-stylesubtitle)
  - [GET /api/subtitles/get-style/history](#get-apisubtitlesget-stylehistory)

---

## 通用数据结构

### 错误响应格式

所有 API 在发生错误时返回统一的错误格式：

```json
{
  "message": "错误描述信息",
  "timestamp": 1709251200000
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| message | string | 错误描述信息 |
| timestamp | number | 错误发生时的 Unix 时间戳（毫秒） |

### HTTP 状态码

| 状态码 | 说明 |
|--------|------|
| 200 OK | 请求成功 |
| 400 Bad Request | 客户端错误（参数错误、状态不正确等） |
| 404 Not Found | 资源不存在 |
| 500 Internal Server Error | 服务器内部错误 |
| 503 Service Unavailable | 服务不可用 |

---

## 字幕管理 API

### POST /api/subtitles/process

处理字幕识别和翻译（统一 API）。

**功能说明：**
- 接收前端发送的字幕数据（包含精确时间戳）
- 记录原文字幕
- 根据配置自动进行翻译（如果启用）
- 返回 SSE 流式响应

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道唯一标识符 |
| text | string | body | ✅ | 字幕文本内容 |
| startTime | number | body | ✅ | 字幕开始时间（毫秒，绝对时间戳） |
| endTime | number | body | ✅ | 字幕结束时间（毫秒，绝对时间戳） |

**请求示例：**
```json
{
  "channelId": "ch1",
  "text": "Hello World",
  "startTime": 1709251200000,
  "endTime": 1709251203500
}
```

**响应类型：** `text/event-stream` (SSE)

**SSE 事件类型：**

#### 1. source_recorded - 原文字幕已记录

```
event: source_recorded
data: {"status":"source_recorded","text":"Hello World"}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| status | string | 状态标识 |
| text | string | 原文文本 |

#### 2. result - 特殊结果（当 doTranslate=false 时）

```
event: result
data: no_translation
```

#### 3. translation_chunk - 翻译流式片段

```
event: translation_chunk
data: 你
```

| 数据 | 类型 | 说明 |
|------|------|------|
| chunk | string | 翻译结果的一个片段 |

#### 4. complete - 翻译完成

```
event: complete
data: {"status":"completed","translation":"你好世界"}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| status | string | 状态标识 |
| translation | string | 完整翻译结果 |

#### 5. error - 错误信息

```
event: error
data: {"error":"错误描述信息"}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| error | string | 错误描述 |

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Missing required fields: channelId and text | 缺少必填字段 |
| 400 | Missing required fields: startTime and endTime | 缺少时间字段 |
| 400 | startTime must be less than endTime | 开始时间必须小于结束时间 |
| 400 | Subtitle recording not initialized for channel: ch1 | 字幕录制未初始化 |
| 400 | Stream start time not set for channel: ch1 | 直播开始时间未设置 |
| 500 | Failed to process subtitle | 处理字幕失败 |

---

### POST /api/subtitles/session/start

开始字幕录制会话。

**功能说明：**
- 初始化字幕录制环境
- 创建 SRT 文件写入器
- 准备接收字幕数据

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道唯一标识符 |

**请求示例：**
```json
{
  "channelId": "ch1"
}
```

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Session started`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | channelId is required | 缺少 channelId 参数 |
| 400 | Channel not found: ch1 | 频道不存在 |
| 500 | Failed to start session | 启动会话失败 |

---

### POST /api/subtitles/session/end

结束字幕录制会话。

**功能说明：**
- 关闭 SRT 文件写入器
- 清理缓存和状态记录

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道唯一标识符 |

**请求示例：**
```json
{
  "channelId": "ch1"
}
```

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Session ended`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | channelId is required | 缺少 channelId 参数 |
| 500 | Failed to end session | 结束会话失败 |

---

### GET /api/subtitles

获取字幕文件列表。

**功能说明：**
- 列出所有已录制的字幕文件
- 包含文件元数据（大小、修改时间等）

**请求参数：** 无

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 字幕文件列表

```json
[
  {
    "filename": "[2026-02-28-23-17-28]_source_TestChannel_https_example_com.srt",
    "path": "subtitles\\[2026-02-28-23-17-28]_source_TestChannel_https_example_com.srt",
    "size": 1024,
    "lastModified": "2026-02-28T15:17:28Z",
    "timestamp": "[2026-02-28-23-17-28]",
    "type": "source",
    "channelName": "TestChannel",
    "streamLink": "https_example_com"
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| filename | string | 文件名 |
| path | string | 文件完整路径 |
| size | number | 文件大小（字节） |
| lastModified | string | 最后修改时间（ISO 8601） |
| timestamp | string | 时间戳（文件名中的部分） |
| type | string | 字幕类型：`source`（原文）或 `translate`（翻译） |
| channelName | string | 频道名称 |
| streamLink | string | 直播流链接（已处理） |

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to list subtitles | 获取列表失败 |

---

### GET /api/subtitles/{filename}

获取字幕文件内容。

**功能说明：**
- 读取指定字幕文件的文本内容

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| filename | string | path | ✅ | 字幕文件名 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应类型：** `text/plain`
- **响应内容：** SRT 格式的字幕内容

```srt
1
00:00:00,960 --> 00:00:03,480
Hello World

2
00:00:04,000 --> 00:00:07,000
你好世界
```

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Invalid filename | 文件名不合法（包含 .. 或路径分隔符） |
| 404 | Not Found | 文件不存在 |
| 500 | Failed to get subtitle | 读取文件失败 |

---

### GET /api/subtitles/{filename}/download

下载字幕文件。

**功能说明：**
- 以附件形式下载字幕文件

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| filename | string | path | ✅ | 字幕文件名 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应类型：** `text/plain`
- **响应头：** `Content-Disposition: attachment; filename="filename.srt"`
- **响应内容：** 文件字节流

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Invalid filename | 文件名不合法 |
| 404 | Not Found | 文件不存在 |
| 500 | Failed to download subtitle | 下载失败 |

---

### DELETE /api/subtitles/{filename}

删除字幕文件。

**功能说明：**
- 删除指定的字幕文件

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| filename | string | path | ✅ | 字幕文件名 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Deleted: filename.srt`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Invalid filename | 文件名不合法 |
| 404 | Not Found | 文件不存在 |
| 500 | Failed to delete subtitle | 删除失败 |

---

### POST /api/subtitles/batch-delete

批量删除字幕文件。

**功能说明：**
- 一次性删除多个字幕文件

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| filenames | array | body | ✅ | 文件名列表 |

**请求示例：**
```json
{
  "filenames": [
    "[2026-02-28-23-17-28]_source_Test.srt",
    "[2026-02-28-23-17-28]_translate_Test.srt"
  ]
}
```

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Deleted 2 of 2 files`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | No filenames provided | 未提供文件名列表 |
| 500 | Failed to batch delete subtitles | 批量删除失败 |

---

### GET /api/subtitles/filter/{type}

按类型筛选字幕文件。

**功能说明：**
- 根据字幕类型筛选文件列表

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| type | string | path | ✅ | 字幕类型：`source` 或 `translate` |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 字幕文件列表（格式同 GET /api/subtitles）

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to filter subtitles | 筛选失败 |

---

### GET /api/subtitles/channel/{channelName}

按频道名称筛选字幕文件。

**功能说明：**
- 根据频道名称筛选文件列表
- 支持 URL 编码的频道名称

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelName | string | path | ✅ | 频道名称（可 URL 编码） |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 字幕文件列表（格式同 GET /api/subtitles）

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to get subtitles by channel | 获取失败 |

---

## 频道状态 API

### POST /api/channel/set

设置/更新频道信息。

**功能说明：**
- 创建或更新频道信息
- 保存到频道仓库

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道唯一标识符 |
| channelName | string | body | ❌ | 频道显示名称 |
| platform | string | body | ❌ | 平台类型：`youtube`、`twitch`、`bilibili` |
| streamUrl | string | body | ❌ | 直播流地址 |
| streamTitle | string | body | ❌ | 直播标题 |

**请求示例：**
```json
{
  "channelId": "ch1",
  "channelName": "Test Channel",
  "platform": "youtube",
  "streamUrl": "https://example.com/stream.m3u8",
  "streamTitle": "My Stream"
}
```

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 更新后的频道信息对象

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Channel Id is null or empty | 频道 ID 为空 |
| 500 | 更新失败 | 更新频道失败 |

---

### GET /api/channel/get/byid/{channelId}

根据频道 ID 获取频道信息。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | path | ✅ | 频道唯一标识符 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 频道信息对象

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 404 | Not Found | 频道不存在 |

---

### GET /api/channel/get/name

获取频道名称。

**功能说明：**
- 根据平台类型获取频道名称
- YouTube：通过 yt-dlp 查询
- Twitch：直接返回
- Bilibili：返回不支持提示

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道 ID |
| platform | string | body | ✅ | 平台类型 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 包含频道名称的频道信息对象

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Bilibili is not supported yet | Bilibili 平台暂不支持 |
| 500 | Failed to get YouTube channel name | 获取 YouTube 频道名称失败 |

---

### POST /api/channel/monitor/start

启动频道状态监控。

**功能说明：**
- 启动定期检测频道直播状态
- 自动配置字幕和 OBS 自动化处理
- 当检测到直播开始时自动设置开始时间
- 当检测到直播结束时自动停止录制

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道 ID |
| platform | string | body | ✅ | 平台类型 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Monitoring started`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to start monitoring | 启动监控失败 |

---

### POST /api/channel/monitor/stop

停止频道状态监控。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道 ID |
| platform | string | body | ✅ | 平台类型 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Monitoring stopped`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to stop monitoring | 停止监控失败 |

---

### GET /api/channel/monitor/status

获取当前频道状态。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | body | ✅ | 频道 ID |
| platform | string | body | ✅ | 平台类型 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 频道状态对象

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 404 | Not Found | 频道不存在 |
| 500 | Failed to get current status | 获取状态失败 |

---

### GET /api/channel/monitor/sse

实时推送频道状态（SSE）。

**功能说明：**
- 通过 Server-Sent Events 实时推送频道状态更新
- 长连接，持续推送
- 如果频道不存在，会自动创建新频道记录

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| channelId | string | query | ✅ | 频道 ID |
| platform | string | query | ✅ | 平台类型：`youtube`、`twitch`、`bilibili` |

**响应类型：** `text/event-stream`

**SSE 数据格式：**
```json
{
  "channelId": "ch1",
  "channelName": "Test Channel",
  "platform": "youtube",
  "isCheckingStream": false,
  "isNoStream": false,
  "isStreaming": true,
  "isUpcomingStream": false,
  "scheduledStreamTime": "2026-03-01T12:00:00Z",
  "streamUrl": "https://example.com/stream.m3u8",
  "streamTitle": "My Stream"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| channelId | string | 频道 ID |
| channelName | string | 频道名称 |
| platform | string | 平台类型 |
| isCheckingStream | boolean | 是否正在检查状态 |
| isNoStream | boolean | 是否无直播 |
| isStreaming | boolean | 是否正在直播 |
| isUpcomingStream | boolean | 是否有即将开始的直播 |
| scheduledStreamTime | string | 预定开始时间（ISO 8601） |
| streamUrl | string | 直播流地址 |
| streamTitle | string | 直播标题 |

**前端使用示例：**

```typescript
// 使用原生 EventSource API
const eventSource = new EventSource(
  `/api/channel/monitor/sse?channelId=${channel.channelId}&platform=${channel.platform}`
);

eventSource.onmessage = (event) => {
  const status = JSON.parse(event.data);
  console.log('Channel status:', status);
};

// 关闭连接
eventSource.close();
```

---

## 翻译服务 API

### GET /api/translation/models

获取可用翻译模型列表。

**功能说明：**
- 获取当前配置的翻译服务支持的模型列表

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：**
```json
{
  "options": [
    {
      "label": "GPT-4o",
      "value": "gpt-4o"
    },
    {
      "label": "GPT-3.5 Turbo",
      "value": "gpt-3.5-turbo"
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| options | array | 模型选项列表 |
| options[].label | string | 模型显示名称 |
| options[].value | string | 模型 ID |

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to get models | 获取模型列表失败 |

---

### POST /api/translation/translate/stream

流式翻译（SSE）。

**功能说明：**
- 提供流式翻译服务
- 实时返回翻译片段
- 可选是否记录字幕

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| text | string | body | ✅ | 待翻译文本 |
| from | string | body | ✅ | 源语言代码 |
| to | string | body | ✅ | 目标语言代码 |
| channelId | string | query | ❌ | 频道 ID（用于字幕记录） |
| recordSubtitle | boolean | query | ❌ | 是否记录字幕 |

**请求示例：**
```json
{
  "text": "Hello World",
  "from": "en",
  "to": "zh"
}
```

**响应类型：** `text/event-stream`

**SSE 事件类型：**

#### translation - 翻译片段

```
event: translation
data: 你
```

#### complete - 翻译完成

```
event: complete
data: {"status":"completed"}
```

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Translation failed | 翻译失败 |

---

### POST /api/translation/translate

单次翻译（不记录字幕）。

**功能说明：**
- 执行一次性翻译
- 返回完整翻译结果

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| text | string | body | ✅ | 待翻译文本 |
| from | string | body | ✅ | 源语言代码 |
| to | string | body | ✅ | 目标语言代码 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 翻译后的文本

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Translation failed | 翻译失败 |

---

### POST /api/translation/session/start

开始翻译会话（已废弃）。

> ⚠️ **已废弃**：请使用 `/api/subtitles/session/start`

---

### POST /api/translation/session/end

结束翻译会话（已废弃）。

> ⚠️ **已废弃**：请使用 `/api/subtitles/session/end`

---

## 设置管理 API

### GET /api/settings

获取所有系统设置。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 设置对象

```json
{
  "ChannelId": "ch1",
  "StatusCheckTool": "ytdlp",
  "YoutubeCookiesPath": "/path/to/cookies.txt",
  "YoutubeApiKey": "your-api-key",
  "obsWebsocket": {
    "host": "localhost",
    "port": 4455,
    "password": "your-password"
  },
  "doObsRecord": false,
  "doAutoRestartObsSource": true,
  "ObsScene": "restreamer",
  "ObsSource": "stream",
  "doRecognize": true,
  "doTranslate": true,
  "translationProducer": "ollama",
  "ollamaConfig": {
    "host": "localhost",
    "port": 11434,
    "model": "llama2"
  },
  "deeplAuthKey": "your-deepl-key",
  "openaiConfig": {
    "baseUrl": "https://api.openai.com/v1",
    "apiKey": "your-openai-key",
    "model": "gpt-3.5-turbo"
  },
  "sourceLang": "en",
  "TargetLang": "zh",
  "subtitleStyle": {
    "showChinese": true,
    "chineseColor": "#ffffff",
    "chineseSize": 48
  },
  "historyStyle": {
    "mainBgColor": "#000000",
    "mainBgOpacity": 0.6
  }
}
```

**异常情况：**

无（总是返回当前设置，如未配置则返回默认值）

---

### PUT /api/settings/set

更新设置（仅内存，不保存到文件）。

**请求参数：** 完整的设置对象（见 GET /api/settings）

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 更新后的设置对象

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Bad Request | 请求体为空或格式错误 |

---

### POST /api/settings/save

保存当前设置到文件。

**请求参数：** 完整的设置对象

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 保存后的设置对象

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | Bad Request | 请求体为空或格式错误 |

---

## OBS 控制 API

### PUT /api/obs/stream/start

开始推流。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `success`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to start stream | 启动推流失败 |

---

### PUT /api/obs/stream/stop

停止推流。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `success`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to stop stream | 停止推流失败 |

---

### PUT /api/obs/record/start

开始录制。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `success`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to start record | 启动录制失败 |

---

### PUT /api/obs/record/stop

停止录制。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `success`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to stop record | 停止录制失败 |

---

### PUT /api/obs/scene/set

切换 OBS 场景。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| name | string | query | ❌ | 场景名称（默认：restreamer） |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `success`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to set scene | 切换场景失败 |

---

### POST /api/obs/media-monitor/start

启动媒体源监测。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| sourceName | string | body | ❌ | 媒体源名称（默认使用设置中的 ObsSource） |
| interval | number | body | ❌ | 检测间隔（秒，默认：10） |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Media monitor started for source: stream`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | sourceName is required or not configured | 未提供且设置中未配置 |
| 500 | Failed to start media monitor | 启动监测失败 |

---

### POST /api/obs/media-monitor/stop

停止媒体源监测。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| sourceName | string | body | ❌ | 媒体源名称 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** `Media monitor stopped for source: stream`

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 400 | sourceName is required or not configured | 未提供且设置中未配置 |
| 500 | Failed to stop media monitor | 停止监测失败 |

---

### GET /api/obs/media-monitor/status

获取媒体监测状态。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| sourceName | string | query | ❌ | 媒体源名称 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：**
```json
{
  "sourceName": "stream",
  "isMonitoring": true,
  "isAutoRestartEnabled": true
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| sourceName | string | 媒体源名称 |
| isMonitoring | boolean | 是否正在监测 |
| isAutoRestartEnabled | boolean | 是否启用自动重启 |

**异常情况：**

| HTTP 状态码 | 错误信息 | 说明 |
|------------|---------|------|
| 500 | Failed to get media monitor status | 获取状态失败 |

---

## 字幕样式 API

### PUT /api/subtitles/save-style/subtitle

保存字幕样式配置。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| showChinese | boolean | body | ❌ | 是否显示中文字幕 |
| chineseColor | string | body | ❌ | 中文字幕颜色（十六进制） |
| chineseStrokeColor | string | body | ❌ | 中文字幕描边颜色 |
| chineseSize | number | body | ❌ | 中文字幕大小（像素） |
| showEnglishFinal | boolean | body | ❌ | 是否显示英文终稿 |
| englishFinalColor | string | body | ❌ | 英文终稿颜色 |
| englishFinalStrokeColor | string | body | ❌ | 英文终稿描边颜色 |
| englishFinalSize | number | body | ❌ | 英文终稿大小 |
| showEnglishRealtime | boolean | body | ❌ | 是否显示英文实时字幕 |
| englishRealtimeColor | string | body | ❌ | 英文实时字幕颜色 |
| englishRealtimeStrokeColor | string | body | ❌ | 英文实时字幕描边颜色 |
| englishRealtimeSize | number | body | ❌ | 英文实时字幕大小 |
| bgOpacity | number | body | ❌ | 背景透明度（0.0-1.0） |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 保存后的样式对象

---

### PUT /api/subtitles/save-style/history

保存历史记录样式配置。

**请求参数：**

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| mainBgColor | string | body | ❌ | 主背景颜色 |
| mainBgOpacity | number | body | ❌ | 主背景透明度 |
| cardBgColor | string | body | ❌ | 卡片背景颜色 |
| cardBgOpacity | number | body | ❌ | 卡片背景透明度 |
| englishColor | string | body | ❌ | 英文字幕颜色 |
| englishStrokeColor | string | body | ❌ | 英文字幕描边颜色 |
| englishSize | number | body | ❌ | 英文字幕大小 |
| chineseColor | string | body | ❌ | 中文字幕颜色 |
| chineseStrokeColor | string | body | ❌ | 中文字幕描边颜色 |
| chineseSize | number | body | ❌ | 中文字幕大小 |

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 保存后的样式对象

---

### GET /api/subtitles/get-style/subtitle

加载字幕样式配置。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 当前字幕样式对象

---

### GET /api/subtitles/get-style/history

加载历史记录样式配置。

**正常响应：**

- **HTTP 状态码：** 200 OK
- **响应内容：** 当前历史记录样式对象

---

## 附录

### 语言代码

| 代码 | 语言 |
|------|------|
| en | 英语 |
| zh | 中文 |
| ja | 日语 |
| ko | 韩语 |
| pt | 葡萄牙语 |
| es | 西班牙语 |
| fr | 法语 |
| de | 德语 |

### 平台类型

| 值 | 平台 |
|----|------|
| youtube | YouTube |
| twitch | Twitch |
| bilibili | 哔哩哔哩 |

### 字幕类型

| 值 | 说明 |
|----|------|
| source | 原文字幕 |
| translate | 翻译字幕 |
