# 器灵 AI Skills

帮助 AI 编程助手为现有工程接入器灵软件授权和开放 API。每个 Skill 是可读取的工作说明与必要参考资料，按 Agent Skills 的 `SKILL.md` 结构组织。

[官网](https://toolsoul.daniu7.cn) · [在线开发文档](https://867fas1f7s.apifox.cn/) · [Python SDK](https://github.com/qilingAI/qiling-sdk-python)

## 当前 Skill

| Skill | 用途 |
|---|---|
| [qiling-sdk-integrate](skills/qiling-sdk-integrate/SKILL.md) | 软件授权、套餐与设备校验，或 API Key 浏览器授权、文本/图片/视频模型、任务和余额查询 |

适用于 Python 桌面工具、Web 和其他语言软件。Python 使用官方 SDK，其他语言按在线 HTTP 文档接入。不要求把原工程重写，也不自动上传、上架、付款或部署。

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

从项目配置或平台提供的接入信息确认实际 API 根地址（`base_url`）。官网链接用于访问网站，不代表 SDK 的 API 地址；Skill 不预设生产域名，也不要在根地址后附加 `/api/v1`。

软件授权接入：在器灵开发者后台保存软件和套餐后，到“我的软件 → 开发调试”生成测试 App Key 和测试授权码。App Key 使用平台返回值，不自行从软件 ID 或 slug 推导。授权码不要提交到仓库或发送到公共对话；把它放在本地安全输入/存储中。

开放接口接入：使用用户 API Key，已有 Key 从安全存储读取，或通过浏览器确认后一次性领取。SDK 0.3.0 提供 `QilingOpenClient` 和 `QilingAuthorizationClient`。软件传正式 App Key 校验授权与套餐 AI 开关，独立工具直接调用可省略。账号/SDK token 不能替代 API Key。流程、接口和 Python 示例见 [开放接口参考](skills/qiling-sdk-integrate/references/open-api.md)。

实际接口和字段以 [在线文档](https://867fas1f7s.apifox.cn/) 为准，AI 可读取 [llms.txt](https://867fas1f7s.apifox.cn/llms.txt)。附带参考摘要用于快速开始和离线场景，不宣称替代平台的最新版本。

测试授权只做登录、设备、权益与测试额度调试，不能调用真实 AI 或付款。Skill 不包含 SDK 二进制、用户凭据、后台源码或生产配置。

模型必须从动态文档选择并显式传 `model` 和业务 `request_id`，不内置模型清单或价格。调用按用户余额计费，与软件普通次数额度分开；网络结果不明时查询原任务，不自动重发生成请求。旧 `ai_chat/ai_image/ai_translate` 已移除，文本和图片改用 API Key 客户端。

## 维护

修改 Skill 时保持它与当前平台 API 一致，核对参考链接与安装来源。使用模拟服务验证未授权拒绝、设备 ID、会话刷新和退出，不为验证创建真实订单或消耗正式权益。

许可证为 MIT，见 [LICENSE](LICENSE)。
