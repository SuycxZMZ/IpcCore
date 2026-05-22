# IpcCore Agents 工作约束

本文档供所有智能体在本仓库中快速浏览项目硬约束。完整需求和架构仍以 `req.md`、`arch.md` 为准，本文只做集中摘要，不替代原文档。

---

## 1. 项目边界

- 模块名统一为 `IpcCore`，中文描述为 `IpcCore 进程间通信核心库` 或 `IpcCore 通信核心库`。
- 第一版只做 Linux 调试版，采用 C/S 模型。
- 第一版底层通信限定为 Unix Domain Socket，socket 类型为 `AF_UNIX + SOCK_STREAM`。
- 第一版必须使用非阻塞 I/O 和 epoll 事件循环。
- 第一版不接入 Android，不依赖 Android NDK，不依赖 Android native log。
- 第一版不做 AHardwareBuffer、GraphicBuffer、fd passing、PubSubCore、ShmPubSubCore、Camera 业务、JNI、jar、TCP、UDP、RPC。
- 后续分层为：

```text
ShmPubSubCore
  ↓
PubSubCore
  ↓
IpcCore
```

---

## 2. 代码规模和设计原则

- 第一版核心代码控制在 5000 行以内，不包含 demo 和测试代码。
- 设计原则：大道至简、边界清晰、代码可控、便于替换日志系统。
- 第一版不做复杂线程池、不做无锁队列、不做复杂 Reactor 框架、不做复杂插件机制。
- 能用清晰类和明确状态机解决的问题，不引入复杂继承、复杂模板或隐式行为。

---

## 3. 命名约束

- 类名前缀统一为 `Ipc`，例如 `IpcServer`、`IpcClient`、`IpcSession`、`IpcEventLoop`、`IpcMessage`。
- 配置和结果类型使用 `IpcServerConfig`、`IpcClientConfig`、`IpcStatistics`、`IpcError`、`IpcResult`、`IpcConnectionState`。
- 文件名前缀统一为 `ipc_`，推荐目录为 `ipc_core/`。
- 命名空间统一为：

```cpp
namespace ipc_core
{
}
```

- 不再使用 `C/S Core`、`CS Core`、`CsCore`、`Cscore`、`CsServer`、`CsClient`、`CsSession` 等旧模块名或旧类名。
- “C/S 模型”只能作为通信模型描述保留，不能作为模块名。

---

## 4. C++ 标准和依赖

- IpcCore 第一版最高支持 C++17。
- 不允许使用 C++20 或更高版本特性。
- 不允许使用协程、concepts、C++20 ranges、`std::jthread`。
- 内部线程统一使用 `std::thread`。
- 同步原语优先使用 C++17 标准库能力：`std::mutex`、`std::condition_variable`、`std::atomic`、`std::lock_guard`、`std::unique_lock`。
- `std::recursive_mutex` 仅在确有必要时使用，使用处必须有中文注释说明原因。
- 第一版不引入 Boost、ASIO、libevent、libuv 等大型第三方库。
- 第一版只依赖 Linux 系统调用和 C++17 标准库。

---

## 5. I/O、epoll 和队列

- 所有 socket fd 必须设置为 non-blocking。
- accept、connect、send、recv 都不允许永久阻塞。
- 必须处理 `EAGAIN`、`EWOULDBLOCK`、`EINTR`、对端关闭和 socket error。
- 事件循环必须支持 Stop 唤醒，不得执行长时间阻塞逻辑。
- 每个 Session 必须有独立发送队列和接收缓存。
- 发送队列使用 `std::mutex + std::deque` 即可，不使用无锁队列。
- Send 只允许快速入队并立即返回，不允许在 Send 内 while 1 等待，不允许队列满后 sleep 无限重试。
- 队列满必须返回明确错误，例如 `IpcError::kQueueFull`。
- 正常消息收发必须支持半包累计和粘包拆分。

---

## 6. epoll 底层 POD 数据结构

- 涉及 epoll、socket fd、event fd、timer fd、唤醒 fd 等底层事件信息时，可以使用简单 C 风格 POD 数据结构。
- POD 数据结构仅用于底层事件循环内部，不向上层业务暴露。
- POD 字段必须语义明确，成员使用小写 + 下划线，例如 `int fd`、`uint32_t events`、`uint64_t session_id`、`void* user_data`。
- POD 结构体必须有中文注释，说明用途和字段含义。
- POD 不得持有复杂资源所有权。
- fd 的创建、关闭、所有权转移必须由明确的 C++ 类或函数负责。
- `epoll_event.data.ptr` 如果保存指针，必须保证指针生命周期长于 epoll 监听周期。
- Session 关闭时，必须先从 epoll 删除 fd，再释放 Session 对象。
- 如使用 `void* user_data`，必须在注释中说明真实类型、生命周期和安全约束。

---

## 7. 资源所有权

- IpcCore 虽然底层使用 Linux C API，但资源管理必须使用现代 C++ 思路。
- fd、线程、Session、发送队列、接收缓存都必须有明确所有权。
- `IpcServer` 拥有监听 fd 和服务端事件循环线程。
- `IpcClient` 拥有客户端 socket fd 和客户端事件循环线程。
- `IpcSession` 拥有连接 fd、发送队列、接收缓存和 Session 状态。
- 服务端可以通过 `std::unordered_map<uint64_t, std::shared_ptr<IpcSession>>` 管理多个 Session。
- 跨线程共享 Session 可以使用 `std::shared_ptr` 管理生命周期。
- 事件回调和避免循环引用场景可以使用 `std::weak_ptr`，使用前必须 `lock()`。
- 不允许通过裸指针长期持有跨线程对象所有权；裸指针只能作为非拥有引用使用，并必须注释说明。
- 不允许形成无法释放的 `std::shared_ptr` 循环引用。
- Stop、Close、析构路径必须断开回调、关闭 fd、清理队列、释放 Session 引用。
- fd 建议使用小型 RAII 封装，或者至少保证每个 fd 只有一个明确关闭点。

---

## 8. 线程创建和线程命名

- IpcCore 内部线程统一使用 `std::thread` 创建。
- 不使用 `pthread_create` 直接创建线程。
- 不允许 detach 长期线程。
- Stop 时必须通知线程退出并 join。
- 线程函数必须捕获异常，避免异常逃逸导致进程直接 terminate。
- 每个长期运行线程必须有明确线程名。
- 线程启动后使用 pthread 接口设置线程名；Linux 下使用 `pthread_setname_np`。
- 线程名用于 top、perf、gdb、logcat 等工具追踪，应简短、稳定、可读，并考虑 Linux 线程名长度限制。
- 建议线程名：`ipc_server_io`、`ipc_client_io`、`ipc_callback`。
- 多实例可追加短 id，例如 `ipc_srv_io_1`、`ipc_cli_io_1`。
- 线程命名失败不影响主流程，但必须打印 warning 日志。

---

## 9. Stop / Close / 析构

- Stop 必须可重复调用。
- Stop 不允许永久阻塞。
- Stop 必须唤醒 event loop。
- Stop 必须关闭 socket fd 和所有 Session。
- Stop 必须清空发送队列和接收缓存。
- Stop 必须等待线程退出。
- Stop 后不得再触发新的消息回调。
- 析构函数不允许抛异常；析构时如果尚未 Stop，应尽最大努力调用 Stop。
- Session 关闭流程必须避免重复 close fd。

---

## 10. 日志约束

- IpcCore 内部只允许使用统一日志宏。
- 日志宏统一为 `IPC_LOGE`、`IPC_LOGW`、`IPC_LOGI`、`IPC_LOGD`。
- 默认日志 tag 建议为 `IPC_LOG_TAG "IpcCore"`。
- 第一版 Linux 调试时日志打印到控制台。
- 后续 Android 可切换到 Android native log，真实车机项目可切换到项目已有日志系统。
- 不允许在业务代码中直接写 `printf`、`std::cout`、`__android_log_print` 或项目私有日志接口。
- 高频日志必须限频。
- 正常每条消息收发不允许默认打印 info 日志。

---

## 11. 注释和接口

- 每个函数、类、结构体、枚举类必须有中文注释。
- 每个公开接口必须说明用途、参数含义、返回值含义、线程上下文。
- 涉及 epoll、非阻塞 I/O、半包粘包、发送队列、自动重连、心跳超时、Session 关闭、Stop 退出等核心逻辑，必须有关键中文注释。
- 注释不能只重复代码表面含义，必须解释设计意图、状态变化、资源所有权或异常处理原因。
- 回调函数必须注明运行在哪个线程。
- Stop / 析构 / Close 相关函数必须注明资源释放顺序。
- Send 相关函数必须注明是否阻塞、是否只是入队、队列满时如何返回。
- 所有 public 方法必须返回明确错误码或结果对象，不允许只用 bool 表达复杂失败原因。
- 上层不应直接依赖 errno，内部 errno 必须转换为 IpcCore 错误码。

---

## 12. 修改文档的要求

- 修改需求或架构时，优先同步更新 `req.md` 和 `arch.md`。
- 不要删除 `req.md`、`arch.md` 中已有的完整约束内容。
- 若本文与 `req.md`、`arch.md` 不一致，应以 `req.md`、`arch.md` 为准，并同步修正本文。

---

## 13. 提交风格

- 提交标题必须使用 `[中文类别] 动词短语`。
- 标题方括号 `[]` 内必须是中文。
- 不允许使用 `[docs]`、`[metrics]`、`[web_preview]`、`[chore]`、`[ARCH-008]` 这类英文、拼音或纯编号类别。
- 提交正文必须使用多条 `- ` 列表说明修改事实、边界和影响。
- 提交信息风格必须对齐项目既有中文提交，例如 `[文档] 提交需求文档以及架构文档`。
