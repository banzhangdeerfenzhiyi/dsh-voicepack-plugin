# dsh-voicepack-plugin — 微信语音包

> 独立仓库：<https://github.com/banzhangdeerfenzhiyi/dsh-voicepack-plugin>
> 属于 DSH 微信人格工具链；不替换已有的 `dsh-wechatagent-voice-plugin`（它的 `send_voice` 继续用默认音色）。

给 DeepSeek Harness 里的微信人格机器人加一套**语音包**：15 个 Edge 神经音色
（9 女声 / **6 男声**，含东北话、陕西话、粤语、台湾腔）＋ 可选 RVC 音色层
（大肥鱼=男声、派蒙、影），用指定音色说出真实微信语音条。

与已有 `dsh-wechatagent-voice-plugin` 的关系：那个插件的 `send_voice` 用**默认音色**
说话；本插件负责「换音色」——用户说“用男声说 / 换个声音 / 用云希的声音念 /
用大肥鱼的声音说”时，调用本插件的 `send_voice_pack`。两个插件共用同一条
Edge TTS → SILK → 本机 hook `/SendVoiceMsg` 通道。

## 目录

```
dsh-voicepack-plugin/
├── index.mjs            # 插件入口：注册 voice_pack_list / send_voice_pack，注入规则与策略
├── src/
│   ├── voices.mjs       # 音色目录（Edge 女声/男声/方言 + RVC 预设）与解析
│   ├── speech.mjs       # Edge TTS → 可选 RVC → PCM → SILK
│   ├── text.mjs         # 把回复文本清理成适合朗读的内容并分段
│   └── wechat.mjs       # 会话 → 微信聊天 → wxid → 本机 hook 发语音条
├── python/rvc_convert.py # Applio RVC 转换（路径全部走环境变量）
├── tools/
│   ├── install.mjs      # 装进 DSH profile（写 cordis.patch.yml，可 --remove）
│   ├── list-voices.mjs  # 命令行预览音色目录
│   └── test-voice.mjs   # 不走 DSH 直接合成一条语音（可 --send）
├── test/unit.test.mjs   # 纯本地测试（目录解析/配置/RVC 降级）
└── package.json
```

## 环境要求

- Windows + DSH 桌面版；WeChatAgent 本机 hook（默认 `http://127.0.0.1:30001`）及其工作区
  （包含 `state/accounts` 与 `config/wechatagent.json`）。
- Node ≥ 22.13。
- 可选：RVC 环境（`applio/core.py` + `venv312` + 音色模型）——没有时只用 Edge 音色，不影响使用。

## 安装

```bash
npm install
node tools/install.mjs        # 自动探测 WeChatAgent 工作区/RVC 目录并写入 profile patch
# 重启 DeepSeek Harness
node tools/list-voices.mjs    # 命令行预览音色
node tools/test-voice.mjs --text "你好呀" --voice 云希
```

参数：`--target <dir>` 自定义安装目录、`--workspace <dir>`、`--rvc-root <dir>`、
`--remove` 卸载、`--profile <name>`。

## 工具

### `voice_pack_list`

列出音色目录，可按 `gender`（female/male/all）、`locale`、`tag` 过滤。
返回内容包含每个音色的中文名、id、性别和标签，以及 RVC 预设。

### `send_voice_pack`

| 参数 | 说明 |
|---|---|
| `text` | 要念出来的口语内容（建议 ≤120 字，会自动清理 markdown 并分段） |
| `voice` | 可选 Edge 音色：`云希`/`晓伊`/`晓晓`/`东北话`/`粤语男声`…或完整 id |
| `rvc` | 可选 RVC 音色层：`大肥鱼`（男声）、`派蒙`（女声）、`影` |
| `rate` / `pitch` | 可选语速/音高，如 `+10%` / `+40Hz` |
| `maxChars` | 可选每条语音字数上限 |

发到**当前 DSH 会话绑定的微信聊天**（与 `send_voice` 相同映射）；RVC 模型缺失时
自动退回纯 Edge 音色而不是报错。

## 音色一览（全部在 Edge 朗读服务实测可用）

- 女声：晓晓、晓伊（默认）、晓萱
- **男声**：云希（年轻常用）、云健（播音）、云扬（新闻磁性）、云夏（少年）
- 方言：晓北（东北话）、晓妮（陕西话）
- 粤语：曉曼、曉佳、雲龍（男）；台湾腔：曉臻、曉雨、雲哲（男）
- RVC 音色：派蒙（女声，pitch -5）、大肥鱼（男声）、影（女声）

> 注：晓辰/晓涵/云野/云枫/云皓/云泽等音色只在 Azure 语音服务提供，
> Edge 朗读接口会拒绝，所以没有列进来；需要它们可以接 Azure TTS。

## 插件生态与配合使用

这套工具链拆成 4 个独立仓库，按「基础设施 → 语音能力 → 点歌 / 换音色」分层：

| 仓库 | 作用 | 什么时候需要 |
|---|---|---|
| [wechat-agent-bundle](https://github.com/banzhangdeerfenzhiyi/wechat-agent-bundle) | WeChatAgent 安装包（0.4.1，含已授权匹配微信 4.1.10.27 客户端）+ DSH 接入插件 + Hook 部署脚本/教程 | 微信侧接入 DSH 的基础设施（语音条、人格聊天） |
| [dsh-wechat-deploy-plugin](https://github.com/banzhangdeerfenzhiyi/dsh-wechat-deploy-plugin) | 在聊天里检查 / 准备 / 启动 / 重启微信 Hook | Hook 没就绪、版本不匹配、语音发不出去时先跑它 |
| [dsh-voicepack-plugin](https://github.com/banzhangdeerfenzhiyi/dsh-voicepack-plugin) | 15 个 Edge 音色 + 3 个 RVC 音色，用指定音色说微信语音条 | 想换音色 / 男声说话 |
| [dsh-qqmusic-plugin](https://github.com/banzhangdeerfenzhiyi/dsh-qqmusic-plugin) | QQ 音乐点歌 + AI 翻唱（默认 30 秒语音条） | 想听「她」唱歌 |
| **本仓库 dsh-voicepack-plugin** | 指定音色说微信语音条 | 想换音色 / 男声说话 |

### 配合使用示例（同一台机器）

1. 先按 [wechat-agent-bundle](https://github.com/banzhangdeerfenzhiyi/wechat-agent-bundle) 装好 WeChatAgent 与 DSH 接入插件，登录微信。
2. 用 [dsh-wechat-deploy-plugin](https://github.com/banzhangdeerfenzhiyi/dsh-wechat-deploy-plugin) 的 `wechat_deploy_status` 确认 Hook `ready` —— 语音条通道就绪。
3. 在微信里：
   - 「唱首稻香」→ 本插件渲染翻唱并发出语音条；
   - 「用男声说晚安」→ [dsh-voicepack-plugin](https://github.com/banzhangdeerfenzhiyi/dsh-voicepack-plugin) 用指定音色发声；
   - 「检查一下微信部署」→ [dsh-wechat-deploy-plugin](https://github.com/banzhangdeerfenzhiyi/dsh-wechat-deploy-plugin) 体检 Hook。
4. 三个插件共用同一条「Edge TTS / RVC → SILK → 本机 hook」语音通道，**互不替换**：
   原有 `dsh-wechatagent-voice-plugin` 的 `send_voice` 继续用默认音色说话，本插件负责「唱歌」，
   语音包负责「换音色」。

## 排错

| 现象 | 处理 |
|---|---|
| 提示找不到 WeChatAgent 工作区 | 配置 `workspace` 或设置 `WECHATAGENT_WORKSPACE` |
| 语音没发出去 | 看 `~/.dsh/voicepack/voicepack.log`；确认 WeChatAgent hook 在运行、当前会话绑定微信 |
| RVC 音色没生效 | 看日志里的 `rvc=` 字段；检查 `rvcRoot`、模型文件是否存在（缺失会自动退回 Edge 音色） |
| Edge TTS 失败 | 需要联网访问 Edge 朗读服务；稍后重试 |

## 注意

- 音色由微软 Edge 朗读服务提供，免费但非官方接口；RVC 音色模型是第三方资产，请使用你有权使用的音色。
- 语音内容仅供个人娱乐，不要用于冒充、诈骗或侵权。

