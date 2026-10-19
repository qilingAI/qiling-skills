# 开放接口、API Key 与模型

核对日期：2026-10-19。来源为 [公开文档索引](https://867fas1f7s.apifox.cn/llms.txt)。从索引读取当前接口页，不猜发布 ID。API 根地址由接入配置指定，官网不自动作为 API 地址。

## 选择凭证

- `QilingAiClient` 使用授权码登录取得 SDK token，负责软件权益、设备、普通次数和续费。
- 官方 OpenAI SDK 使用用户 API Key，模型接口采用标准 Bearer 鉴权；器灵 `QilingOpenClient` 0.5.4 同步 `/v1` 模型协议，工具/账户接口仍使用 x-api-key。SDK/账号 JWT 不能代替 API Key。
- 普通模型调用只需要 API Key、Base URL 和 Model，不要求 App Key 或授权码。已登记软件需要来源检查时传 x-qiling-app-key，服务端检查软件启用、上架状态与 AI 套餐配置；不要求 Key 账号拥有软件授权。App Key 使用平台登记值。

Key 是通用用户凭证，可以访问启用的模型和工具，并消耗用户调用余额。多个应用可能共用同一默认 Key，重置/停用会影响所有接入应用。App Key 用于软件能力检查，不是拿到完整 Key 后不可移除的隔离边界；不要为绕过拒绝响应删除软件来源头。

软件登录账号与 AI 授权账号独立：A 的软件 SDK 会话继续校验使用权，B 在浏览器确认后提供 B 的 API Key，AI 费用、余额和记录归 B。切换 AI Key 不切换或覆盖软件会话。不要自行要求两者同账号，也不因 AI Key 有效而绕过软件权益。测试 App Key 不支持正式 AI，正式软件无可用 AI 配置或已关闭时仍拒绝。

软件展示快捷授权入口前，从最新登录/校验响应读取 product.ai_quick_authorization_enabled；false 隐藏，字段缺失才默认 true。开关按原软件授权所属站点设置，切换账号或重新校验后更新，不能用 AI 付费账号站点推导。它不影响已授权 Key 或模型权限，不需要另查接口，详细默认值见 [integration.md](integration.md#ai-快捷授权入口)。

## 浏览器快速授权

1. `POST /api/v1/auth/api-key/requests`，软件提交 app_key，可选 site_code 取软件原授权/校验响应 site.code；Skills 可省略并提供 client_name。无需账号或 SDK Bearer。
2. 返回 `device_code`、`verification_url`、`expires_in`、`interval`。只展示或打开 verification_url，让用户自己登录并确认。device_code 仅应用持有，不放进浏览器、URL 或日志，无需本地回调服务。
3. `POST /api/v1/auth/api-key/token`，JSON 请求体为 `{"device_code": "应用持有的轮询凭证"}`。按 interval（至少 3 秒）查询，最长 expires_in（默认 600 秒）。pending 等待；denied、consumed 或过期停止；HTTP 410 过期，429 放慢轮询。
4. authorized 仅首次返回 `api_key`、`app_key`、`base_url` 和模型/文档 URL。响应丢失需重新授权，不自动重复领取。API Key 存系统凭据库或受保护存储；不提交 Git，不放公共前端配置。不要根据返回 URL 自动更换凭证请求的目标站点。

Python SDK 0.5.4 起可保留分站来源：

```python
from qiling_ai_sdk import QilingAuthorizationClient

authorization = QilingAuthorizationClient(
    api_url,
    app_key=registered_app_key,
    site_code=license_session.get("site", {}).get("code", "platform"),
)
client = authorization.connect()
model_docs = client.model_docs()
```

site_code 是软件来源，不能改用浏览器付款账号的站点、任意域名或软件 slug。服务端为有效分站使用数据库登记的 host（含已登记自定义域名）生成 verification_url；客户端只打开返回地址，不拼接或硬编码主站。空字符串、省略或 platform 使用主站；未知/停用分站 HTTP 404，域名配置无效 HTTP 409，提示联系分站管理员，不自动回退。不要提交任意 URL、Host 或 redirect_uri。

来源只影响确认页品牌和新注册用户的归属，不修改已登录 B 的站点、Key 或扣费账户，也不改变 A 的软件授权。connect 继续使用原配置的统一 API 地址，不能将分站确认页域名用作 Base URL；site_code 不是鉴权或收费凭证。已有软件将 site.code 与原授权上下文保存，重登/切换软件会话时重新读取。

`connect` 返回 API Key 客户端；`authorize` 返回原始结果，供宿主保存密钥，两者都不会持久化。GUI 在后台任务执行，取消或切换用户后丢弃旧结果；需要事件循环自行驱动时用 start/poll。无自动浏览器时用 `open_browser=False, on_authorization_url=show_link`。API Key 没有 SDK refresh_token 流程；401 检查密钥状态或重新授权。

## OpenAI 标准调用

已有 OpenAI SDK 程序保留原调用方式，只替换三个配置：用户 API Key、已确认的平台模型 Base URL（含 `/v1`）、平台模型名。不改为器灵私有请求/响应封装。

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["QILING_API_KEY"],
    base_url=os.environ["QILING_BASE_URL"],  # 含 /v1，不是 /api/v1/open。
    max_retries=0,
)
result = client.chat.completions.create(
    model=os.environ["QILING_MODEL"],
    messages=[{"role": "user", "content": user_text}],
)
text = result.choices[0].message.content
```

不需要安装器灵 SDK，也不强制先查询目录或增加 App Key、request_id。需要选模型时通过 `client.models.list()` 读取 `data[].id`，能力和用户价格位于每个模型 `qiling.config`。单模型查询使用 `client.models.retrieve(model_id)`；完整动态文档仍是平台根地址下的 `GET /api/v1/open/model-docs`，返回 items/request_schema/limits/pricing。以文档和模型限制判断支持参数，不固定模型清单、价格或尺寸，不自动执行返回的示例。

流式沿用 `client.chat.completions.create(..., stream=True, stream_options={"include_usage": True})`，迭代 chunk.choices 的 delta.content；最终用量块可能 choices 为空，需先检查。结束或取消时关闭 stream，不将 SSE 当成一次 JSON，不将中途断开视为成功或自动重发。普通文本支持 max_tokens 或 max_completion_tokens 二选一，采样与思考参数按当前模型文档使用。

图片沿用 `client.images.generate`，普通结果 data[] 中读取 url/b64_json。图片编辑调用官方 `images.edit(image=文件或文件列表, model=..., prompt=...)`，发送 multipart 的 image/image[]，不将文件路径作为远程 URL。图片 SSE 使用 stream=True，经官方 SDK 或器灵 client.openai() 桥接读取并关闭；普通 image 方法不解码 SSE。参考图、图层扩展用 extra_body，字段按公开 request_schema。

当前开放 Chat Completions（含 SSE、函数工具/并行工具、工具结果续接、结构化输出、图文/音视频内容块）、图片生成/编辑及图片 SSE。具体能力需模型后台配置及供应商支持，不因接口有字段就假设每个模型都支持。Responses 尚未开放。视频仍是器灵 JSON 异步扩展。工具与多模态细节见 [model-capabilities.md](model-capabilities.md)。

已登记软件需要来源检查时可用官方客户端 default_headers 设置 x-qiling-app-key；普通直接调用不加。已有工程原先提供该来源头时保留，不为绕过拒绝而删除。

## 器灵客户端与路径

`QilingOpenClient` 0.5.4 接受平台根地址或含 `/v1` 的模型 Base URL；普通 chat/image 返回标准 JSON，不再返回旧 data 包装。平台调用 id 位于 result["qiling"]["id"]，费用在 qiling.billing；文本是 choices[0].message.content，图片为 data[]。流式使用官方 OpenAI SDK，器灵客户端的 chat 不读取 SSE。

| 功能 | 当前完整路径 | 用法 |
|---|---|---|
| 模型目录/单模型 | GET `/v1/models`、`/v1/models/{model_id}` | 官方 models.list/retrieve；器灵 models/model |
| 文本 | POST `/v1/chat/completions` | 官方 chat.completions.create；器灵 chat |
| 图片 | POST `/v1/images/generations` | 官方 images.generate；器灵 image 为普通 JSON |
| 图片编辑 | POST `/v1/images/edits` | 官方 images.edit，multipart 文件；支持模型开放的 SSE |
| 视频扩展 | POST `/v1/videos` | 器灵 video，request_id 可选，支持 content 与便捷素材字段 |
| 费用预估 | POST `/v1/models/estimate` | 器灵 estimate(kind, request) |
| 原模型任务/取消视频 | GET/DELETE `/v1/model-calls/{call_id}` | 器灵 model_call/cancel_video |
| 动态模型文档 | GET `/api/v1/open/model-docs` | 器灵 model_docs |
| 余额/统计/记录 | GET `/api/v1/open/balance`、`/api/v1/open/usage`、`/api/v1/open/logs` | balance/usage/logs |
| 工具目录 | GET `/api/v1/open/endpoints` | endpoints |
| 工具异步任务 | POST `/api/v1/open/tasks`，GET `/api/v1/open/tasks/{task_id}` | submit_task/task |
| 去 AI | POST `/api/v1/open/remove-ai` | remove_ai(text)，再 task(task_id) |

旧 `/api/v1/open` 模型路径已移除，不再提供回退请求。账户、工具和快速授权保留原路径。工具提交只有 endpoint/input/options，不加模型字段；模型视频使用独立 videos 路径。

文本/图片仍需 model 及原协议的 messages 或 prompt；请求号为可选 request_id 或 Idempotency-Key。普通官方调用无需自定义字段，省略时平台生成。需要可靠业务重试时，事先保存请求号，官方 SDK 在 extra_headers 中传固定 Idempotency-Key；器灵客户端用 idempotency_key 或 request_id。默认关闭生成请求自动重试，未知结果先查询原平台调用；不同内容不能共用请求号。开放视频 request_id 也可省略或使用 Idempotency-Key，返回顶层 id/status/billing 和 data.video_url；请求尾帧且上游返回时读取 data.last_frame_url。SDK Bearer 视频仍必填 request_id。

### 去 AI 正文与换行

去 AI 的段落边界会影响上游改文效果。`remove_ai(text)` 的 `text`，以及 `submit_task("remove_ai", input=...)` 的 `input`，都应原样传递用户正文，保留真实换行、空行、缩进和首尾空白。粘贴的多行文章保存为多行 UTF-8 文件，不用空格拼接段落、空白正则或 `strip()` 清洗正文。文件可用 `Path(path).read_bytes().decode("utf-8-sig")` 读取，保留原始换行格式。

JSON 编码会负责表示换行，服务端解码后仍是真实换行。不要先把换行替换成反斜杠加字母 n 的文本，或手动编码两次。长文按接口字符上限分段时，按顺序拼接各段应与输入正文逐字符一致；提交前核对实际换行数量，原文有多行而提取结果为一行时先修复输入。接口返回的改文也保留段落结构。测试应检查真实 HTTP JSON 解码后的正文与原文完全一致。

## 计费、任务与迁移

- 模型共享用户调用余额，与普通软件次数额度独立；不在模型调用后追加 consume_quota，不套用工具按次优惠。余额不包含推广可提现收益，logs/usage 限当前 API Key。
- estimate 接收 kind=text/image/video 和对应完整请求，只做最大费用估算，不调用上游或冻结余额。生成提交时只要求余额大于 0，不预扣、冻结或占用余额；完成后按可信真实用量和请求价格快照结算，可使余额为负。余额小于等于 0 时停止新调用，充值先抵欠款。不要用估价做客户端预冻结，也不把 estimated_fee 当最终扣费。用户报价已含倍率，不再乘一次；费用保留十进制字符串，计算时用 Decimal。
- 视频仅 succeeded 为完成，queued/running/cancel_requested 查询同一平台 id；unknown/review 或用量缺失等待核对。文本/图片扩展费用位于 qiling.billing，视频位于顶层 billing；fee=null 不表示免费。流式错误或未完成不能假定结算成功。
- 模型错误是标准 error 对象，包含 message/type/param/code，器灵 SDK 在 QilingAiError.detail 保留该对象；HTTP 400 参数不支持、401 Key 无效、402 余额不足、403 权限限制、409 冲突或待核对。网络结果不明不自动重发，保存可用的 qiling.id 或错误中 call_id/request_id；取消接受不代表最终取消或退款。
- 旧 QilingAiClient.ai_chat/ai_image/ai_translate 已移除，文本和图片迁移到官方 OpenAI SDK 或 QilingOpenClient 0.5.4 的 chat/image；翻译可使用当前文本模型。SDK Bearer 仍保留 ai_models、ai_estimate、ai_video、ai_task、ai_cancel_video、ai_bills；SDK Bearer 视频请求 model 和 request_id 必填，app_key 来自已校验的软件授权；支持与开放视频相同的新素材和音频选项。

默认以模拟 API 测试鉴权分离、授权终止、稳定请求号和原任务查询。测试授权不能调用真实模型；仅为接入或查文档不自动发起付费生成、充值或购买。
