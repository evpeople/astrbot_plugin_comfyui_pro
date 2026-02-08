# AstrBot 源码架构指南

本文档整理自插件开发过程中对 AstrBot 源码的探索。

## 项目结构

```
AstrBot/
├── astrbot/
│   ├── api/                    # API 接口层
│   │   ├── event/             # 事件相关
│   │   ├── message_components.py  # 消息组件
│   │   ├── platform/           # 平台相关
│   │   ├── provider/          # LLM 提供商
│   │   └── star/              # 插件基类
│   ├── core/                   # 核心实现
│   │   ├── agent/             # Agent 框架
│   │   ├── conversation_mgr.py # 对话管理器
│   │   ├── platform/          # 平台适配器
│   │   └── stages/            # 处理阶段
│   └── builtin_stars/         # 内置插件
└── ...
```

## 核心概念

### 1. 消息事件 (AstrMessageEvent)

消息事件是所有交互的基础，定义在 `astrbot/core/platform/astr_message_event.py`。

```python
class AstrMessageEvent(abc.ABC):
    def __init__(
        self,
        message_str: str,           # 纯文本消息
        message_obj: AstrBotMessage, # 完整消息对象
        platform_meta: PlatformMetadata,
        session_id: str,
    ):
```

**关键属性**：
- `message_str`: 纯文本消息内容
- `message_obj`: `AstrBotMessage` 对象，包含完整消息结构
- `platform_meta`: 平台元信息（平台类型、ID 等）
- `unified_msg_origin`: 统一消息来源，格式为 `platform_name:message_type:session_id`

**关键方法**：
- `get_sender_id()`: 获取发送者 ID
- `get_group_id()`: 获取群组 ID（私聊返回空字符串）
- `get_sender_name()`: 获取发送者名称
- `send(message)`: 发送消息
- `plain_result(text)`: 创建纯文本结果

### 2. Telegram 平台适配器

Telegram 专用事件类型 `TelegramPlatformEvent`（`astrbot/core/platform/sources/telegram/tg_event.py`）：

```python
class TelegramPlatformEvent(AstrMessageEvent):
    MAX_MESSAGE_LENGTH = 4096  # Telegram 消息限制

    async def send_streaming(self, generator, use_fallback=False):
        """流式发送消息（仅 Telegram 等支持编辑消息的平台）"""
```

**注意事项**：
- Telegram 使用 `message_obj.group_id` 获取群 ID
- Telegram 私聊时 `group_id` 为 `None`
- Telegram 使用 `message_obj.message_id` 获取消息 ID

### 3. 对话管理器 (ConversationManager)

负责管理会话与 LLM 的对话历史，定义在 `astrbot/core/conversation_mgr.py`。

**核心方法**：

```python
class ConversationManager:
    async def get_curr_conversation_id(unified_msg_origin: str) -> str | None:
        """获取当前对话 ID"""

    async def get_conversation(
        unified_msg_origin: str,
        conversation_id: str,
        create_if_not_exists: bool = False
    ) -> Conversation | None:
        """获取对话对象"""

    async def get_conversations(
        unified_msg_origin: str | None = None,
        platform_id: str | None = None
    ) -> list[Conversation]:
        """获取所有对话列表"""

    async def add_message_pair(
        cid: str,
        user_message: UserMessageSegment | dict,
        assistant_message: AssistantMessageSegment | dict
    ) -> None:
        """添加用户-助手消息对到历史"""
```

**Conversation 对象结构**：
```python
@dataclass
class Conversation:
    platform_id: str      # 平台 ID
    user_id: str         # 用户 ID（统一消息来源）
    cid: str             # 对话 ID（UUID 格式）
    history: str = ""    # 历史记录（JSON 格式字符串）
    title: str | None = ""
    persona_id: str | None = ""
    created_at: int = 0
    updated_at: int = 0
```

**历史记录格式**：
`history` 是 JSON 字符串，解析后为消息列表：
```json
[
    {"role": "user", "content": "用户消息"},
    {"role": "assistant", "content": "助手回复"}
]
```

### 4. LLM 调用

通过 `context.llm_generate()` 调用：

```python
provider_id = await self.context.get_current_chat_provider_id(umo)
llm_resp = await self.context.llm_generate(
    chat_provider_id=provider_id,
    prompt="你的提示词",
)
result = llm_resp.completion_text  # 获取返回文本
```

### 5. 消息组件

定义在 `astrbot/api/message_components.py`：

| 组件 | 说明 |
|------|------|
| `Plain` | 纯文本 |
| `Image` | 图片 |
| `At` | @用户 |
| `Reply` | 回复引用 |
| `File` | 文件 |
| `Record` | 语音 |

**示例**：
```python
from astrbot.api.message_components import Plain, Image, Reply

# 检查消息链中的组件
for comp in event.message_obj.message:
    if isinstance(comp, Plain):
        text = comp.text
    elif isinstance(comp, Image):
        path = await comp.convert_to_file_path()
    elif isinstance(comp, Reply):
        # Telegram 的回复消息
        reply_content = comp.message_str
```

## 流式输出注意事项

AstrBot 默认开启流式输出（`streaming_response: True`）。

**潜在问题**：
- 流式输出时，LLM 回复内容**可能不会立即写入数据库**
- 对话历史的保存有 `save_interval = 60` 秒间隔
- 插件在 LLM 回复完成后才能获取到历史记录

**解决方案**：
- 对于 `/draw` 等需要获取 LLM 回复的指令，优先检测 Reply 消息
- 在 Telegram 中，让用户回复 LLM 的消息再触发指令

## 插件开发要点

### 1. 指令注册

```python
@filter.command("指令名", aliases=["别名1", "别名2"])
async def my_command(self, event: AstrMessageEvent):
    yield event.plain_result("回复内容")
```

### 2. 权限检查

```python
def _check_access(self, event: AstrMessageEvent):
    """检查用户是否有权限使用"""
    user_id = str(event.get_sender_id())
    if user_id in self.admin_user_ids:
        return True, ""
    return False, "无权限"
```

### 3. 消息发送

```python
# 纯文本
yield event.plain_result("Hello")

# 图片
from astrbot.api.message_components import Image
image = Image.fromFileSystem("/path/to/image.jpg")
yield event.chain_result([image])

# 消息链
from astrbot.api.message_components import Plain, Image
chain = [Plain("这是"), Image.fromFileSystem("a.jpg")]
yield event.chain_result(chain)
```

## 常见问题

### 1. Telegram event 没有 `message` 属性

Telegram 事件类型是 `TelegramPlatformEvent`，没有 `message` 属性，应该用 `message_obj`：

```python
# 错误
event.message  # AttributeError

# 正确
event.message_obj.message  # 消息链列表
event.message_obj.message_str  # 纯文本
```

### 2. 对话历史为空

可能原因：
- 流式输出模式下历史未及时保存
- 当前对话 ID 与 LLM 写入的不是同一个

解决方案：
- 使用 `get_conversations()` 获取所有对话
- 优先检测 Reply 消息

### 3. 获取不到 LLM 回复

检查步骤：
1. 确认 `umo`（unified_msg_origin）格式正确
2. 使用 `get_curr_conversation_id()` 获取当前对话
3. 使用 `get_conversations()` 遍历所有对话
4. 解析 `history` JSON 字符串，查找 `role: assistant` 的消息

## 参考文件

| 文件 | 说明 |
|------|------|
| `astrbot/core/platform/astr_message_event.py` | 消息事件基类 |
| `astrbot/core/platform/sources/telegram/tg_event.py` | Telegram 适配器 |
| `astrbot/core/conversation_mgr.py` | 对话管理器 |
| `astrbot/api/message_components.py` | 消息组件 |
| `astrbot/api/event/__init__.py` | 事件 API |
