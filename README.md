# 器灵 AI Skills

帮助 AI 编程助手为现有工程接入器灵软件授权和开放 API。每个 Skill 是可读取的工作说明与必要参考资料，按 Agent Skills 的 `SKILL.md` 结构组织。

[官网](https://toolsoul.daniu7.cn) · [在线开发文档](https://867fas1f7s.apifox.cn/) · [Python SDK](https://github.com/qilingAI/qiling-sdk-python)

## 当前 Skill

| Skill | 用途 |
|---|---|
| [qiling-sdk-integrate](skills/qiling-sdk-integrate/SKILL.md) | 软件授权、套餐与设备校验，或 API Key 浏览器授权、文本/图片/视频模型、任务和余额查询 |

适用于 Python 桌面工具、Web 和其他语言软件。模型接口采用 OpenAI 格式，已有 OpenAI SDK 程序替换 API Key、Base URL、Model 即可；软件授权和平台功能使用器灵 SDK。不要求重写工程，也不自动上传、上架、付款或部署。

## 使用

支持 Skills CLI 的编程助手，可以安装指定 Skill：

```sh
npx skills add qilingAI/qiling-skills --skill qiling-sdk-integrate
```

安装时选择需要的助手与项目范围；本命令不是上传、付款或部署授权。也可以手动安装：克隆仓库，将 `skills/qiling-sdk-integrate` 整个文件夹放入你的编程助手支持的 Skills 目录。支持显式调用的助手可以这样请求：

```text
使用 $qiling-sdk-integrate 为这个工程接入器灵 SDK。
先做测试授权登录和套餐权益校验，复用现有界面与配置。
```

没有 Skill 自动加载功能时，将 [SKILL.md](skills/qiling-sdk-integrate/SKILL.md) 提供给助手，并让它按其中引用读取接入说明。

开放接口接入也可这样请求：

```text
使用 $qiling-sdk-integrate 为这个工程接入器灵开放 API。
复用现有 API Key 安全存储；没有 Key 时提供浏览器授权。
按动态模型文档实现调用和任务查询，先用模拟请求验证。
```

## 准备配置

从项目配置或平台提供的接入信息确认地址。OpenAI SDK 的 Base URL 为平台根地址加 `/v1`；软件授权和快速授权使用平台根地址。官网链接不自动作为 API 地址，不预设生产域名，也不重复拼接前缀。

软件授权接入：在器灵开发者后台保存软件和套餐后，到“我的软件 → 开发调试”生成测试 App Key 和测试授权码。App Key 使用平台返回值，不自行从软件 ID 或 slug 推导。授权码不要提交到仓库或发送到公共对话；把它放在本地安全输入/存储中。

开放接口接入：使用用户 API Key，已有 Key 从安全存储读取，需要时通过浏览器确认后一次性领取。器灵 SDK 0.5.3 支持分站授权来源；已有官方 OpenAI SDK 可直接沿用，无需增加 App Key 或器灵请求号。已登记软件需要来源检查时可传正式 App Key。账号/SDK token 不能替代 API Key。流程、接口和 Python 示例见 [开放接口参考](skills/qiling-sdk-integrate/references/open-api.md)。

实际接口和字段以 [在线文档](https://867fas1f7s.apifox.cn/) 为准，AI 可读取 [llms.txt](https://867fas1f7s.apifox.cn/llms.txt)。附带参考摘要用于快速开始和离线场景，不宣称替代平台的最新版本。

测试授权只做登录、设备、权益与测试额度调试，不能调用真实 AI 或付款。Skill 不包含 SDK 二进制、用户凭据、后台源码或生产配置。

模型使用 `/v1/models` 返回的 `data[].id`；开放文本、图片和视频请求号均可选；需要业务重试时保存并复用原号。文本读取 `choices[0].message.content`，图片读取 `data[]`；流式沿用官方 SDK。调用按用户余额计费，与软件普通次数额度分开；结果不明时不自动重发。旧 `/api/v1/open` 模型路径已移除，工具和账户接口保持原路径。

新增能力包括函数工具、并行工具、工具结果续接、JSON 结构化输出、多模态理解、图片编辑/流式，以及视频参考素材、音频与首尾帧；按模型配置和上游支持启用。调用细节见 [模型能力参考](skills/qiling-sdk-integrate/references/model-capabilities.md)。普通调用可用 Python 3.9+；器灵 SDK 的官方 OpenAI 桥接可选依赖需 Python 3.10+。

新增思考模式能力声明：读取 supports_thinking/thinking_modes 和动态请求 Schema 后提供开关；不传 thinking 仍可能采用上游默认思考。软件登录与 AI 付费账号可以不同，AI 费用及记录归 API Key 账号。当前模型调用不预扣或冻结余额，按可信实际用量结算，余额可为负；充值先抵欠款。

分站软件浏览器授权时传原软件会话 site.code 作为 site_code，只打开服务端 verification_url。确认页品牌可来自分站，API 地址和 AI 付费账号独立；404/409 不自动改走主站。省略或 platform 保持主站流程，不预设分站域名。

## 维护

修改 Skill 时保持它与当前平台 API 一致，核对参考链接与安装来源。使用模拟服务验证未授权拒绝、设备 ID、会话刷新和退出，不为验证创建真实订单或消耗正式权益。

许可证为 MIT，见 [LICENSE](LICENSE)。
