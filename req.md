# IpcCore 进程间通信核心库需求文档

版本：v0.3  
阶段：第一版 Linux 调试版  
范围：仅 IpcCore，不包含 PubSubCore、ShmPubSubCore、Android handle 传递和 Camera 业务
设计原则：大道至简、边界清晰、代码可控、便于替换日志系统

---

## 1. 背景

当前底层通信、发布订阅和共享内存 Buffer 管理耦合较深，异常定位时难以判断问题来自 socket、订阅关系、共享内存状态还是 Camera 业务。后续要重构 PubSubCore 和 ShmPubSubCore，必须先沉淀一个稳定、精简、可测试的 IpcCore 进程间通信核心库。

整体分层目标如下：

```text
ShmPubSubCore
  ↓
PubSubCore
  ↓
IpcCore
```

R1. 第一阶段只聚焦 IpcCore。

R2. IpcCore 只负责连接、会话、消息、队列、重连、心跳、超时、错误、日志和资源释放。

R3. IpcCore 不理解 topic、publish、subscribe、frame、buffer_index、GraphicBuffer、AHardwareBuffer、共享内存状态位、Camera 通道和 JNI 回调。

---

## 2. 重构目标

R4. IpcCore 第一版采用 C/S 模型，在 Linux 环境下实现一个精简、稳定、可测试的本机进程间通信核心库。

R5. IpcCore 后续作为 PubSubCore 和 ShmPubSubCore 的底座。

R6. 第一版优先保证主路径清晰、异常路径可靠、退出路径可控。

R7. 第一版不追求框架化、大而全、插件化。

R8. 第一版核心代码规模目标控制在 5000 行以内，不包含 demo 和测试代码。

---

## 3. 第一版范围

### 3.1 第一版目标

R9. 使用 Unix Domain Socket。

R10. socket domain 使用 `AF_UNIX`。

R11. socket type 使用 `SOCK_STREAM`。

R12. 所有 socket fd 使用非阻塞 I/O。

R13. 使用 epoll 事件循环。

R14. 支持 Server / Client / Session。

R15. 支持多客户端连接。

R16. 支持发送队列。

R17. 支持发送队列上限。

R18. 队列满时 Send 必须返回明确错误。

R19. 支持慢客户端断开。

R20. Client 默认开启自动重连。

R21. 支持心跳与超时。

R22. 支持半包和粘包处理。

R23. 支持 Stop / DeInit / 析构安全退出。

R24. 支持统一错误码或结果对象。

R25. 支持统一日志宏。

### 3.2 第一版非目标

R26. 第一版不做 Android 接入。

R27. 第一版不做 Android AHardwareBuffer 传递。

R28. 第一版不做 GraphicBuffer 传递。

R29. 第一版不做 fd passing。

R30. 第一版不做 PubSubCore。

R31. 第一版不做 ShmPubSubCore。

R32. 第一版不做 Camera 业务接入。

R33. 第一版不做 JNI。

R34. 第一版不做 jar。

R35. 第一版不做跨设备 TCP。

R36. 第一版不做 UDP。

R37. 第一版不做 RPC 框架。

R38. 第一版不做复杂线程池。

R39. 第一版不做无锁队列。

R40. 第一版不做消息持久化和历史消息回放。

R41. Android AHardwareBuffer / GraphicBuffer / fd passing 只能作为后续扩展预留，不能写成第一版需求。

---

## 4. 平台范围

R42. 第一版只要求 Linux 环境调试通过。

R43. 第一版不依赖 Android NDK。

R44. 第一版不依赖 Android native log。

R45. 第一版不依赖 AHardwareBuffer。

R46. 第一版不依赖 GraphicBuffer。

R47. 后续 Android 版本可以在第一版基础上补充 Android native log、Android socket 权限处理、SELinux 策略适配、AHardwareBuffer handle 传递和 ShmPubSubCore 集成。

---

## 5. 功能需求

### 5.1 Server 需求

R48. Server 负责监听本地 socket path。

R49. Server 负责 accept Client 连接。

R50. Server 负责为每条连接创建 Session。

R51. Server 负责管理多个 Session。

R52. Server 支持主动关闭指定 Session。

R53. Server Stop 时必须关闭所有 Session。

R54. 单个 Session 异常不得导致整个 Server 崩溃。

### 5.2 Client 需求

R55. Client 负责主动连接 Server。

R56. Client 默认开启自动重连。

R57. Client 自动重连可以通过配置关闭。

R58. Client 自动重连需要支持重连间隔配置。

R59. Client 自动重连只负责重新建立 socket 连接。

R60. Client 自动重连不负责订阅恢复、共享内存恢复或 Camera 业务状态恢复。

### 5.3 Session 需求

R61. Session 表示一条已建立连接。

R62. Session 负责该连接上的发送和接收。

R63. Session 拥有独立发送队列。

R64. Session 拥有独立接收缓存。

R65. Session 拥有独立状态。

R66. Session 拥有独立统计信息。

R67. Session 关闭时必须清空发送队列和接收缓存。

### 5.4 消息协议需求

R68. IpcCore 必须定义统一消息头。

R69. 协议头由 IpcCore 自动填充。

R70. 上层不传入 magic、version 和 header_size。

R71. 接收端必须校验 magic、version、header_size 和 payload_size。

R72. IpcCore 必须基于 payload_size 组包。

R73. 不允许假设一次 recv 得到完整消息。

R74. 不允许假设一次 send 完成完整消息。

R75. 接收缓存必须支持半包累计。

R76. 接收缓存必须支持粘包拆分。

R77. 非法协议头应触发 ProtocolError。

R78. 非法 payload_size 应触发 ProtocolError。

R79. ProtocolError 只关闭对应 Session。

R80. 第一版 payload 只承载小消息，不承载图像数据本体或共享内存大块数据。

R81. 后续 ShmPubSubCore 的 frame metadata 可以作为 payload 承载，但不是第一版重点。

### 5.5 发送队列需求

R82. 每个 Session 必须有独立发送队列。

R83. 发送队列使用有锁队列即可，推荐 `std::mutex + std::deque`。

R84. 发送队列必须支持最大消息数限制。

R85. 发送队列必须支持最大 pending bytes 限制。

R86. 发送队列入队失败时必须返回明确错误。

R87. Send 只负责快速入队并立即返回。

R88. Send 不允许长时间执行 socket send。

R89. Send 不允许 while 1 等待。

R90. Send 不允许队列满后 sleep 重试。

R91. Send 队列满时返回 QueueFull。

R92. Send 在 Session 未连接时返回 Disconnected。

R93. Send 在 Core 未启动时返回 NotStarted。

### 5.6 自动重连需求

R94. Client 默认开启自动重连。

R95. 自动重连可以关闭。

R96. 自动重连间隔可配置。

R97. 默认重连间隔建议为 500 ms。

R98. 可支持最大重连间隔。

R99. 可支持简单退避策略。

R100. 第一版不做复杂指数退避。

R101. 第一版不做复杂熔断。

R102. 重连成功后只上报 OnConnected。

R103. 后续 PubSubCore 根据 OnConnected 恢复订阅关系。

### 5.7 心跳与超时需求

R104. IpcCore 支持心跳。

R105. 第一版心跳默认开启。

R106. 心跳间隔可配置。

R107. 心跳超时时间可配置。

R108. 心跳消息属于 IpcCore 内部消息。

R109. 心跳消息和心跳 ack 不上抛给上层。

R110. 超过 heartbeat_timeout_ms 未收到对端有效消息，判定连接超时。

R111. 连接超时后主动关闭 Session。

R112. 连接超时后上报 disconnected。

R113. Client 连接超时后进入自动重连。

### 5.8 Stop / DeInit / 析构需求

R114. Server Start 和 Client Start 可以失败，并必须返回明确错误码。

R115. 重复 Start 应返回 AlreadyStarted。

R116. Server Stop 和 Client Stop 必须可重复调用。

R117. Stop 不允许永久阻塞。

R118. Stop 必须唤醒 event loop。

R119. Stop 必须关闭 socket fd。

R120. Stop 必须关闭所有 Session。

R121. Stop 必须清空发送队列。

R122. Stop 必须等待线程退出。

R123. Stop 后不得再触发新的消息回调。

R124. 析构函数不允许抛异常。

R125. 析构时如果尚未 Stop，应尽最大努力调用 Stop。

R126. 所有 fd 必须有明确所有权。

R127. 所有线程必须有明确退出路径。

---

## 6. 非功能需求

### 6.1 代码规模

R128. 第一版核心代码控制在 5000 行以内，不包含 demo 和测试代码。

R129. 第一版不追求复杂抽象层、复杂继承、复杂模板、复杂线程池、复杂 reactor 框架、复杂无锁结构和复杂插件机制。

### 6.2 性能目标

R130. Linux 第一版需要通过 6000 次/s 小 payload 消息压力测试。

R131. 6000 次/s 是小消息通知压力，不是图像数据本体传输。

R132. 压测中不应出现发送队列无限增长。

R133. 压测中不应出现 fd 泄露。

R134. 压测中不应出现线程泄露。

R135. 压测中不应出现 Stop 卡死。

R136. 压测中不应出现 Client 重连后资源泄露。

### 6.3 可靠性

R137. 所有 socket fd 必须设置为 non-blocking。

R138. accept、connect、send、recv 都不允许永久阻塞。

R139. 必须处理 `EAGAIN`、`EWOULDBLOCK` 和 `EINTR`。

R140. 必须处理对端主动关闭。

R141. 必须处理 socket error。

R142. 事件循环必须支持 Stop 唤醒。

R143. 事件循环中不得执行长时间阻塞逻辑。

R144. IpcCore 调用上层回调时不得持有全局锁、发送队列锁或 Session 管理锁。

R145. 上层回调不得长时间阻塞。

### 6.4 可维护性

R146. 第一版最高支持 C++17，不允许使用 C++20 或更高版本特性，不引入大型第三方库。

R147. 对外接口不允许只返回 bool，应返回 IpcResult 或 IpcError。

R148. 上层不应直接依赖 errno，内部 errno 必须转换为 IpcCore 错误码。

R149. IpcCore 错误码最多暴露给 PubSubCore 和 ShmPubSubCore，不直接暴露给 Camera 业务层。

### 6.5 可观测性

R150. IpcCore 需要提供基础统计信息。

R151. 统计信息至少覆盖 active_session_count、total_accept_count、total_connect_count、total_disconnect_count、total_send_message_count、total_recv_message_count、total_send_bytes、total_recv_bytes、total_queue_full_count、total_protocol_error_count 和 total_timeout_count。

R152. 每个 Session 建议记录 session_id、state、pending_message_count、pending_bytes、send_message_count、recv_message_count、queue_full_count、last_send_time、last_recv_time 和 last_error。

---

## 7. 日志系统需求

R153. 第一版 Linux 调试时日志打印到控制台。

R154. 后续 Android 可切换到 Android native log。

R155. 真实车机项目可切换到项目已有日志系统。

R156. IpcCore 内部只允许使用统一日志宏。

R157. IpcCore 内部不允许直接写 `printf`、`std::cout`、`__android_log_print` 或项目私有日志接口。

R158. 日志替换应尽量通过几行宏定义完成。

R159. 第一版保留 `IPC_LOGE`、`IPC_LOGW`、`IPC_LOGI`、`IPC_LOGD` 四类日志宏。

R159-1. 默认日志 tag 建议为 `IPC_LOG_TAG "IpcCore"`。

R160. 关键生命周期和异常必须打印日志，包括 server start、server stop、client start、client stop、listen failed、connect failed、accept success、session disconnected、send failed、recv failed、queue full、send stalled、protocol error 和 heartbeat timeout。

R161. 高频日志必须限频。

R162. 正常每条消息收发不允许默认打印 info 日志。

R163. debug 日志应支持通过编译宏关闭。

---

## 8. 代码实现硬性规范

### 8.1 注释要求

R164. 每个函数必须有中文注释。

R165. 每个类必须有中文注释。

R166. 每个结构体必须有中文注释。

R167. 每个枚举类必须有中文注释。

R168. 每个公开接口必须说明用途、参数含义、返回值含义、线程上下文。

R169. 涉及 epoll、非阻塞 I/O、半包粘包、发送队列、自动重连、心跳超时、Session 关闭、Stop 退出等核心逻辑的位置，必须有关键中文注释。

R170. 注释不能只重复代码表面含义，必须解释设计意图、状态变化、资源所有权或异常处理原因。

R171. 对于容易误用的接口，必须在注释中说明调用约束。

R172. 对于回调函数，必须注明回调运行在哪个线程。

R173. 对于 Stop / 析构 / Close 相关函数，必须注明资源释放顺序。

R174. 对于 Send 相关函数，必须注明是否阻塞、是否只是入队、队列满时如何返回。

注释示例风格：

```cpp
/**
 * @brief 启动 IpcCore 服务端并开始监听本地 Unix Domain Socket。
 *
 * 该函数会创建监听 socket，设置非阻塞模式，并启动 epoll 事件循环线程。
 * 函数成功返回后，服务端可以接受客户端连接。
 *
 * @param config 服务端启动配置，包含 socket 路径、最大连接数、队列大小等参数。
 * @return IpcResult 启动结果，成功返回 IpcError::kOk，失败返回具体错误码。
 *
 * @note 该函数不能重复调用，重复调用应返回 IpcError::kAlreadyStarted。
 * @note 该函数失败时必须释放已经创建的 fd 资源。
 */
IpcResult Start(const IpcServerConfig& config);
```

### 8.2 命名规范

R175. 后续 C++ 实现必须遵守以下命名规范。

| 类型 | 命名风格 | 示例 |
| --- | --- | --- |
| 类名 | 大驼峰 PascalCase | `class CameraSource`, `class FrameBroker` |
| 结构体 | 大驼峰 PascalCase | `struct FrameHandle`, `struct CameraConfig` |
| 联合体 | 大驼峰 PascalCase | `union DataBuffer` |
| 枚举类 | 大驼峰 PascalCase | `enum class PixelFormat` |
| 枚举值 | 小驼峰 + k 前缀 | `kNV12`, `kRGB888` |
| 函数/方法 | 大驼峰 PascalCase | `void StartStreaming()`, `bool Publish()` |
| 成员变量 | 小写 + 下划线后缀 | `int device_fd_`, `bool is_running_` |
| 局部变量 | 小写 + 下划线 | `int frame_count`, `void* buffer_ptr` |
| 全局变量 | g_ + 前缀 + 下划线后缀 | `int g_system_instance_count_` |
| 静态变量 | s_ + 前缀 + 下划线后缀 | `int s_instance_count_` |
| 常量 | k 前缀 + 大驼峰 | `const int kMaxBufferCount = 4` |
| 宏定义 | 全大写 + 下划线 | `#define MAX_SIZE 100` |
| 命名空间 | 全小写 + 下划线 | `namespace ipc_core` |
| 文件名 | 全小写 + 下划线 | `camera_source.h`, `frame_broker.cpp` |
| 类型别名 | 大驼峰 PascalCase | `using FrameCallback = ...` |
| 模板参数 | 大驼峰 PascalCase | `template <typename T>` |

### 8.3 代码格式规范

R176. 缩进统一为 4 个空格，不使用 Tab。

R177. 大括号独立成行，使用 Allman 风格。

R178. 控制语句与圆括号之间保留空格，例如 `if (condition)`。

R179. 单行代码建议不超过 100 列。

R180. 头文件使用 `#pragma once`。

R181. 文件名使用全小写 + 下划线。

R182. 不使用拼音命名。

R183. 不使用含义不清的缩写。

R184. 名称长度建议不超过 32 个字符。

R185. 避免使用保留字和关键字。

R186. 避免使用容易混淆的字符，如 `l`、`1`、`I`、`O`、`0`。

R187. 避免使用下划线开头和结尾，成员变量后缀下划线除外。

### 8.4 C++ 实现约束

R188. 第一版核心代码控制在 5000 行以内，不包含 demo 和测试代码。

R189. IpcCore 第一版最高支持 C++17。

R189-1. 不允许使用 C++20 或更高版本特性。

R189-2. 不允许使用协程。

R189-3. 不允许使用 concepts。

R189-4. 不允许使用 C++20 ranges。

R189-5. 不允许使用 `std::jthread`。

R189-6. IpcCore 内部线程统一使用 `std::thread`。

R189-7. 同步原语优先使用 C++17 标准库能力，例如 `std::mutex`、`std::condition_variable`、`std::atomic`、`std::lock_guard` 和 `std::unique_lock`。

R189-8. `std::recursive_mutex` 仅在确有必要时使用，使用位置必须有中文注释说明原因。

R189-9. 第一版不引入 Boost、ASIO、libevent、libuv 等大型第三方库。

R189-10. 第一版只依赖 Linux 系统调用和 C++17 标准库。

R190. 发送队列使用 `std::mutex + std::deque` 即可，不使用无锁队列。

R191. 第一版不设计复杂线程池。

R192. 所有 fd 必须有明确所有权。

R193. 所有线程必须有明确退出路径。

R194. 所有 public 方法必须返回明确错误码或结果对象，不允许只用 bool 表达复杂失败原因。

R195. 不允许在 Send 内部 while 1 等待。

R196. 不允许队列满后 sleep 无限重试。

R197. 不允许事件循环线程长时间阻塞。

R198. 不允许在持锁状态下调用上层回调。

R199. 不允许析构函数抛异常。

R200. Stop 必须可重复调用。

R201. Stop 不允许永久阻塞。

R202. Stop 后不得再触发新的消息回调。

R203. 高频日志必须限频。

R204. 正常每条消息收发不允许默认打印 info 日志。

### 8.5 Linux 底层 epoll 数据结构约束

R204-1. 涉及 Linux 底层 epoll、socket fd、event fd、timer fd、唤醒 fd 等底层事件信息时，可以使用简单 C 风格 POD 数据结构。

R204-2. POD 数据结构仅用于底层事件循环内部，不向上层业务暴露。

R204-3. POD 数据结构字段必须语义明确，不能使用含糊命名。

R204-4. POD 结构体成员按照项目规范使用小写 + 下划线，例如 `int fd`、`uint32_t events`、`uint64_t session_id`、`void* user_data`。

R204-5. POD 结构体必须有中文注释，说明用途和字段含义。

R204-6. POD 数据结构不得持有复杂资源所有权。

R204-7. fd 的生命周期不得由裸 POD 结构体隐式管理。

R204-8. fd 的创建、关闭、所有权转移必须由明确的 C++ 类或函数负责。

R204-9. epoll 回调或事件分发中不得长期保存悬空指针。

R204-10. 如使用 `void* user_data`，必须在注释中说明其真实类型、生命周期和安全约束。

### 8.6 资源管理与现代 C++ 约束

R204-11. IpcCore 虽然底层使用 Linux C API，但资源管理必须使用现代 C++ 思路。

R204-12. fd、线程、Session、发送队列、接收缓存都必须有明确所有权。

R204-13. 对于跨线程共享的 Session 对象，可以使用 `std::shared_ptr` 管理生命周期。

R204-14. 对于避免循环引用或事件回调中弱引用 Session 的场景，可以使用 `std::weak_ptr`。

R204-15. 不允许通过裸指针长期持有跨线程对象所有权。

R204-16. 裸指针只能作为非拥有引用使用。

R204-17. 如果使用裸指针，必须通过注释说明该指针不拥有资源。

R204-18. 不允许在多个对象之间形成无法释放的 `std::shared_ptr` 循环引用。

R204-19. 回调中捕获对象时，应优先考虑 `std::weak_ptr`，避免对象已销毁后继续访问。

R204-20. Stop、Close、析构路径中必须断开回调、关闭 fd、清理队列、释放 Session 引用。

### 8.7 线程创建与线程命名要求

R204-21. IpcCore 内部线程统一使用 `std::thread` 创建。

R204-22. 不使用 `pthread_create` 直接创建线程。

R204-23. 线程启动后，需要使用 pthread 接口设置线程名。

R204-24. Linux 下使用 `pthread_setname_np`。

R204-25. 线程命名用于车机问题定位、top、perf、gdb、logcat 等工具追踪。

R204-26. 每个长期运行线程必须有明确线程名。

R204-27. 线程名应简短、稳定、可读。

R204-28. 线程名长度需要考虑 Linux 线程名长度限制。

R204-29. 线程退出必须可控。

R204-30. Stop 时必须通知线程退出并 join。

R204-31. 不允许 detach 长期线程。

R204-32. 线程函数必须捕获异常，避免异常逃逸导致进程直接 terminate。

R204-33. 线程入口必须有中文注释说明线程职责。

R204-34. 建议线程名包括 `ipc_server_io`、`ipc_client_io`、`ipc_callback`。

R204-35. 如果后续存在多个实例，可以追加短 id，例如 `ipc_srv_io_1`、`ipc_cli_io_1`。

---

## 9. 验收标准

### 9.1 基础功能验收

R205. Server Start / Stop 正常。

R206. Client Start / Stop 正常。

R207. Client 默认连接 Server。

R208. Server accept Client。

R209. Client Send 到 Server。

R210. Server Send 到 Client。

R211. Server Broadcast 到多个 Client。

R212. Client 主动断开后 Server 感知 disconnected。

R213. Server 主动关闭 Session 后 Client 感知 disconnected。

R214. Server Stop 后 Client 感知 disconnected。

### 9.2 异常场景验收

R215. Client 进程异常退出后 Server 不崩溃。

R216. Server 进程异常退出后 Client 进入自动重连。

R217. Server 重启后 Client 自动重连成功。

R218. 发送队列满返回 QueueFull。

R219. 客户端不读导致发送阻塞时，Server 主动断开该 Session。

R220. 非法 magic 触发 ProtocolError。

R221. 非法 version 触发 ProtocolError。

R222. 非法 payload_size 触发 ProtocolError。

R223. 半包可正确组包。

R224. 粘包可正确拆包。

R225. Stop 不死锁。

R226. 析构不崩溃。

### 9.3 压力测试验收

R227. 1 个 Server + 1 个 Client 持续 6000 msg/s 小消息收发通过。

R228. 1 个 Server + 10 个 Client 广播小消息通过。

R229. Client 反复断开重连 1000 次通过。

R230. Server 反复 Start / Stop 1000 次通过。

R231. 持续运行 24 小时无 fd 泄露。

R232. 持续运行 24 小时无线程泄露。

R233. 持续运行 24 小时无内存持续增长。

---

## 10. 待确认项

TODO-1. 是否允许使用 `std::function` 作为回调。

TODO-2. 是否需要完全避免动态内存分配。

TODO-3. 默认 socket path 命名规则。

TODO-4. 默认 max_pending_message_count。

TODO-5. 默认 max_pending_bytes。

TODO-6. 默认 heartbeat_interval_ms。

TODO-7. 默认 heartbeat_timeout_ms。

TODO-8. 默认 send_stall_timeout_ms。

TODO-9. 第一版是否需要 CMake demo 工程。

TODO-10. 第一版是否需要同时提供 server demo 和 client demo。

TODO-11. 第一版是否需要提供单元测试，还是先用 demo 压测验证。

TODO-12. 是否需要为 Stop 设置默认最大等待时间。

TODO-13. 是否需要为每个 Session 设置最大接收缓存上限。
