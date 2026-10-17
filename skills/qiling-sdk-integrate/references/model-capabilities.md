# 模型新能力

核对日期：2026-10-17。接入前通过 [Apifox 索引](https://867fas1f7s.apifox.cn/llms.txt) 及已确认平台根地址下的 `/api/v1/open/model-docs` 读取当前 Schema。字段开放不代表每个模型均支持；以所选模型的后台能力开关、参数范围及供应商支持为准。

## 思考模式

刷新模型目录与动态文档缓存。模型 data[].qiling.config 的 supports_thinking、thinking_modes 明确支持与可选模式，支持时 capabilities 含 thinking；request_schema.properties.thinking 描述实际可传值。后台显式关闭优先，不靠别名或旧缓存判断支持；未知模型不默认宣称可切换。

仅在能力声明允许时提供 enabled/disabled 选择，并持久化用户设置。没有用户选择且模型支持 disabled 时可显式关闭；不传 thinking 使用供应商默认，不能将省略等同于关闭。器灵 chat 传 thinking={type: disabled}；官方 OpenAI SDK 用 extra_body={thinking: {type: disabled}}，最终 HTTP JSON 是顶层 thinking。reasoning_effort 不能代替开关。

保留 reasoning_content 与最终 content 的区别：普通返回、流式 delta、原任务查询、工具续接和幂等重放都不能丢弃思考字段；不把思考内容当最终答案，不默认写入普通日志。

## 工具与结构化输出

沿用官方 `chat.completions.create`，或器灵 `chat(..., **options)`：tools 是 type=function 的定义，function 含 name/description/parameters，可带 strict；tool_choice 用 none/auto/required 或指定 function，parallel_tool_calls 控制并行调用。平台不执行工具。

返回 assistant.tool_calls 时 content 可以为空，不能将其视为文本回答失败。宿主解析函数名称与 JSON 参数，校验权限和参数后调用已登记工具；模型给出的函数调用不自动授予外部操作权限。保留完整 assistant 消息及供应商要求的 reasoning_content，再逐项以 role=tool、匹配的 tool_call_id 回传结果；继续调用同一模型，处理最终正文或下一轮工具调用。设置合理轮数及取消机制，每轮分别计费。

结构化输出传 response_format={type: json_object}，或 type=json_schema 并提供 json_schema 的 name/schema/strict。按 Schema 验证模型输出；未开启能力时处理标准 error，不静默降级成普通文本。

## 多模态消息

messages 保持 OpenAI 消息结构，不将所有 content 强制转字符串。user.content 可为文本与 image_url、input_audio、video_url 等内容块；具体音视频字段与媒体范围从当前文档读取。assistant 保留 tool_calls 和 reasoning_content，tool 消息保留 tool_call_id。

HTTPS 与 Data URI 是否可用、文件大小和数量限制按模型文档。不要仅根据模型别名推断视觉、音频或视频理解能力；不要自动搭建媒体托管或公开上传用户素材。

## 图片编辑与流式

使用官方 SDK，或 Python 3.10+ 的器灵可选桥接（从已核实的 Git tag 安装 `[openai]`）：

```python
from qiling_ai_sdk import QilingOpenClient

gateway = QilingOpenClient(configured_base_url, securely_loaded_api_key)
with gateway.openai() as client, open(user_image_path, "rb") as source:
    result = client.images.edit(
        model=selected_image_model,
        prompt=user_edit_instruction,
        image=source,
    )
```

`POST /v1/images/edits` 为 multipart/form-data，上传 image 或 image[] 文件，二者选一。字段包括 model、prompt、n、size、response_format、stream 及可选 request_id；不自行设置 multipart boundary，不传本地文件路径作为远程 URL。

图片生成/编辑启用 stream=True 时使用官方 images.generate/edit 的事件迭代，并在 finally 中关闭 stream；只在 completed 事件采用最终图片，不能把 partial_image 当作完整结果。普通 `QilingOpenClient.image` 仅返回 JSON，流式改用官方 SDK 或 gateway.openai()。模型关闭图片输入、编辑或流式能力时会拒绝请求，不能靠更换字段绕过。

## 视频素材与结果

`POST /v1/videos` 使用器灵 JSON 异步协议；SDK 0.5.2 的 `video` 支持：

- content：官方素材块 text/image_url/video_url/audio_url；图片 role 为 first_frame/last_frame/reference_image，参考音频使用对应 reference_audio 角色。
- 便捷参数：prompt、reference_images、reference_video、reference_audios、first_frame、last_frame；与 content 二选一，不混用。首尾帧与全模态参考不能混用，last_frame 需要 first_frame。
- 输出参数：duration、resolution、ratio、generate_audio、return_last_frame、reference_seconds。ratio 支持的范围按当前 Schema；generate_audio 需模型支持。便捷音频参考需结合参考图或视频。

```python
task = gateway.video(
    model=selected_video_model,
    content=[
        {"type": "text", "text": user_prompt},
        {"type": "image_url", "image_url": {"url": approved_image_url}, "role": "reference_image"},
    ],
    ratio="9:16", generate_audio=True, return_last_frame=True,
    idempotency_key=saved_business_request_id,
)
current = gateway.model_call(task["id"])
```

视频参考素材支持文档允许的 HTTPS/Data URI，实际数量、时长和分辨率由模型限制。开放视频 request_id 可省略或用 Idempotency-Key；业务重试复用原号，不生成新任务。SDK Bearer 的 `ai_video` 仍要求 app_key/request_id，SDK 从已有授权补入 app_key，接受相同的新素材参数。

成功读取 data.video_url；请求 return_last_frame 且上游返回时读取 data.last_frame_url。模型任务查询/取消仍用 model_call/cancel_video，SDK Bearer 则用 ai_task/ai_cancel_video。reference_seconds 仅用于费用估算，不预扣或占用余额，最终按可信实际用量结算。断流、用量缺失、unknown/review 只查询或核对原任务，不新建生成请求。

软件使用权与 AI 付费可以归不同账号，Key 账号的余额/日志与软件会话分开保存。提交模型请求只检查余额大于 0；按实际费用结算可出现负余额，此时停止新请求，充值先抵欠款。估价不占用余额，用量缺失或结果未知等待核对，不当作免费成功。

验证使用模拟请求：思考 enabled/disabled 及省略语义、模型能力元数据保留、软件与 AI 账号凭证独立、负余额与估价透传，工具结果与 reasoning_content 完整回传、消息内容块未改写、JSON Schema 选项透传、multipart 字节不变、图片事件正确读取、视频素材互斥及可选值保留。仅为接入不自动执行真实工具或付费生成。
