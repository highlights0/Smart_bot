# API 与软件架构

## 1. 第一阶段目标

先实现稳定语音交互，不部署本地大模型、不训练模型、不自建 ASR/TTS 推理服务。

推荐复用：
- 固件：https://github.com/78/xiaozhi-esp32
- LCDWiki ES3C35P 板型：https://github.com/78/xiaozhi-esp32/tree/main/main/boards/lcdwiki-es3c35p
- 服务端：https://github.com/xinnan-tech/xiaozhi-esp32-server

ESP32 设备按小智协议连接服务端；服务端再调用外部 ASR、LLM、TTS API。不要把普通 OpenAI Chat Completions 地址直接当作小智设备 WebSocket 地址。

## 2. 设备板型

必须选择 `lcdwiki-es3c35p`，对应目录包含 QSPI ST77922 屏幕、触摸、ES8311 音频和 ESP32-S3 GPIO 初始化。

不可误选 Waveshare 的 3.5、3.5B 或 3.49 板型。虽然主控都可能是 ESP32-S3，但显示控制器和引脚不同。

## 3. 横屏人偶脸适配

硬件屏幕原生 320×480 竖屏；参考外观需要横向 480×320。

当前上游 LCDWiki 板型配置使用：
- `DISPLAY_WIDTH=320`
- `DISPLAY_HEIGHT=480`
- `DISPLAY_SWAP_XY=false`
- LVGL `sw_rotate=0`

因此横屏适配是项目自身的固件开发任务，不是开箱即有的功能。

建议实现顺序：
1. 先用上游固件确认原生竖屏显示、触摸和音频工作。
2. 新增独立的 companion display rotation 配置，启用 LVGL 软件旋转或等效的渲染旋转方案。
3. 对触摸输入做相同方向的坐标变换，保证点击位置和显示一致。
4. 测试 30–60 FPS 表情动画、刷新撕裂、内存占用和音频并发。
5. 验证通过后再把开发板横向装入头部。

不要直接修改上游板型的固定引脚和 `DISPLAY_SWAP_XY` 断言；通过独立配置/组件实现，便于后续同步上游。

## 4. 逻辑链路

设备麦克风 → 小智设备通信协议 → 服务端音频处理/ASR → LLM → TTS → 音频回传设备 → 扬声器播放。

屏幕表情由固件本地状态机驱动：
- IDLE：待机
- LISTENING：聆听
- THINKING：思考
- SPEAKING：说话
- HAPPY / SAD / CURIOUS：情绪状态

第一版优先根据设备已有的聆听、思考、播放事件切换表情，不要求大模型返回复杂 JSON。

## 5. API 选择

第一版可选云端 API：
- LLM：阿里云百炼 OpenAI 兼容接口（例如 Qwen 系列）。
- ASR：服务端已适配的云端 ASR 模块。
- TTS：先用低门槛方案验证链路；稳定后再评估流式 TTS。

具体模型名称、免费额度、价格和接口参数可能变化，实施时以服务商控制台和最新官方文档为准。

## 6. 服务端部署

初期只部署小智服务端的 Server 模块，使用 Docker 或源码部署均可。将默认本地 ASR 改为云端 ASR，避免下载和运行本地语音模型。

要求：
- 配置文件和数据目录持久化。
- API Key 使用环境变量或服务端私有配置。
- 不提交真实密钥、设备凭证或个人录音。
- 公网访问时使用 HTTPS/WSS，并限制管理接口访问。
- 记录 API 超时、限流和异常日志，但避免记录完整敏感音频和密钥。

## 7. 记忆

第一版先使用服务端会话上下文/现有记忆模块。后续再增加项目自己的长期记忆服务，至少包含：
- 用户主动要求记住的信息。
- 对话偏好和角色设定。
- 重要事件摘要。
- 记忆查看、修改和删除能力。

## 8. 固件改造边界

优先保持上游音频、网络、OTA 和板级驱动不变。新增功能集中在独立组件：
- companion_ui：界面布局。
- companion_face：眼睛、嘴部和表情动画。
- companion_state：交互状态机。
- companion_display_rotation：横屏渲染和触摸坐标映射。
- companion_servo：后续舵机抽象接口。

API Key 不得进入 ESP32 固件。设备端只保存设备身份和连接服务端所需的最小凭证。
