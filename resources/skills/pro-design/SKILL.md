---
name: pro-design
description: 用 ChatGPT 网页会话或 OpenAI 兼容生图 API 生成设计图与前端代码/HTML demo，产物保存在本地设计面板；只在用户明确要求设计、出图或 demo 时使用。
argument-hint: <设计目标>
display-name: GPT-Image Design
short-description: Generate images and frontend demos through a web session or image API
---

# GPT-Image Design

只在用户明确说「出几张设计图 / 做个前端 demo / 让 ChatGPT 设计」或同意你的提议时启动。纯后端实现、需要跑代码验证的任务不要用。

web 通道与研究插件共用同一个**本机 Chrome profile 和单账号**；一轮只在一个会话里提交，不做多账号并行，也不合并多份答复。

web 模型默认 `prefer-pro`，优先使用可用 Pro，没有才沿用当前模型；禁用、锁定和升级入口不算可用 Pro。显式 `auto` 保留当前选择，指定其他模型仍须核验。图片能力和额度以账号提供为准；旧配置需改为 `prefer-pro` 后新建主题。API 通道仍使用独立的 `imageModel`。

## 启动：先给设计卡

设计卡必须包含：

1. 目标与目标用户；
2. **成功判据**，例如「至少 1 张完整首屏图」「一个可打开的单文件 HTML demo」「与现有品牌/代码不冲突」；
3. 材料（现有设计、品牌、代码路径）与 brief；
4. 轮次上限、本轮方向/图片数（默认 3）与产物位置；
5. 当前通道（web / api）与模型；API 通道要写明「只出图，不出 HTML demo」。

调用 `gpt_image_design`：`action=start, confirmed=true, title, question, brief, criteria[], materials[], rounds, designs, store`。

`store` 必须与设计卡一致（`project` 或 `local`）；核对工具返回的 `store` 与 `path`。local 与 project 均使用插件的 `read_round`、`read_report`、`write_report` 整理交付稿，无需宿主通用文件工具访问 local 目录。保留 DSH 工具权限检查，不修改宿主权限或隐式迁移已有主题。

`designs` 是**本轮方向/图片数**（1–6，默认取设置里的 3）；`rounds` 是追问轮数。两者都写在卡片上让用户确认。

brief 里写清尺寸/比例、风格、必须出现的文案、必须区分的点、是否要 HTML demo。不要只说「设计好看点」。

## 通道差异

- **web**：一轮一次提交，图片与 HTML/代码块一起回来。
- **api**：每个方向一次生图请求（`designs` 次），只出图片；需要 demo 时由你基于图片和 brief 另行写 HTML，并说明不是模型产出。
- api 通道未配置自动裁决时默认只跑 1 轮；用户要求多轮才显式传 `rounds`，并提醒按 API 计费。
- key 只读设置或 `GPT_IMAGE_API_KEY`，不要在会话里回显；`action=probe` 可检查 base URL、key 与模型名。

## 运行期间

发出去后不逐轮汇报。生成图片通常需要 1–3 分钟；卡住时看 `stall_reason`，`unknown_submission` 必须人工核对。api 通道的 `config` 表示 key/base URL/模型名或请求参数被拒，修设置后 `resume`；DSH 重启会丢掉进行中的 API 请求，这一轮不会自动重发。

`stalled` 时读取 `status.last_round`。只有 `submissionState=not_sent` 才能确认发送前停止；任务已获授权且原因已解决或仅为临时导航超时时，可 `resume` 原轮一次，不新建主题。仍失败就报告现场，不循环重试；提交不明时不能凭错误文案自行重发。

## 产物与裁决

- 图片由传输层抓取或下载，保存到 `<主题>/artifacts/round-NN/image-01.png`；
- web 通道的前端代码块保存为 `code-01.html`/`.tsx` 等，HTML 可以直接在设计面板的 iframe 里预览；
- api 通道没有代码块；需要 demo 时由你另写，并在交付说明里标明来源；
- 面板「Image Design」按项目列出主题，详情是画廊 + demo 预览 + 设计决策；概览显示「取回 X/Y 张、HTML 是否取回、视觉待人工确认」，取回数量不等于验收通过。

答复/结果回来后由宿主裁决：

1. 逐张检查图片是否满足判据（文案、比例、方向）；
2. 有 HTML demo 时打开核实能渲染，记录依赖/外链；
3. 选方向、写理由，未采纳项保留驳回依据；
4. 交付 `design.md`，引用实际产物路径，不把模型原文当交付物。

先用 `status` 与 `read_round(topic_id, round)` 核对版本、轮次、原答复及裁决；打开实际图片/HTML 做视觉与交互验收。用 `read_report(topic_id)` 读取旧稿与 `report.revision`，再调用 `write_report(topic_id, markdown, generation, through_round, expected_revision, confirmed=true)`。没有旧稿时传 `expected_revision=""`；用户已授权当前任务与交付时无需重复确认。插件固定写 `design.md` 并自动添加覆盖范围文件头，正文不要包含文件头；旧 `report.md` 仍兼容读取。跨项目、运行中、过时轮次或旧 revision 会被拒绝；冲突时重新读取并整合。工具权限被拒绝时说明原因，不改用外部脚本代写。文件存在不代表视觉验收通过。

自动裁决失败时主题停在 `reviewing`（`review_from=answered`），图片与 HTML 保留：接受、继续、重新生成，或修好裁决模型后重试。文本裁决看不到图片，不得用它替“方向可区分 / 文案可读”这类视觉判据盖章。

## 追问

`gpt_image_design(action="continue", topic_id=..., brief="只改配色，保留版式", materials=[...], confirmed=true)`。

用户选定一张已有图片时，使用面板的“以这张继续改”，或 `gpt_image_design(action="continue", topic_id=..., base_artifact="artifacts/round-01/image-02.png", brief="保留布局，只换背景色", confirmed=true)`。底图必须是本主题已记录的 PNG/JPEG/WebP；这条路径只附所选一张图、只追加一轮一个结果，不与 `materials` 或多个 `designs` 合用。记录保留底图关联，旧图不覆盖。`imageEdit=off` 或 API 不支持 edits 会停止，不改用无底图生成；底图确实发送不等于视觉符合要求。
web 通道在同一会话里只上传新增材料，会话失效会自动回退整份清单；api 通道每轮重新发提示词，参考图按当轮材料重新附加。

整版不满意用 `regenerate`（保留历史、开新版本），`note` 写不满意点，可附 `materials=[...]`（相对主题项目根目录）。终态主题确认后可 `delete`，目录与面板记录一并删除，不可恢复。

## 边界

- 不自动 push、不改仓库代码；demo 只是产物，实施另行授权。
- 图片和代码会写入项目记录；公开仓库注意版权与素材授权。
- web 通道的页面改版、账号风险与 ToS 风险随任务说明；api 通道按服务商条款与 API 计费。
