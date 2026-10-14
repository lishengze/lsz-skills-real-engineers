# daily 复盘输入（2026-10-14 ~ 2026-10-14）

> 由 review_prepare.sh 自动生成，供 agent 执行复盘。
> 统计信息：
- 覆盖日志文件：1 个
- 任务条目：6
- 经验教训条目：6

## 一、任务清单

### 来源：20261014.md
  3:## 任务：将模型自我提升复盘体系应用到 lsz-skills-real-engineers 仓库，并建立完整知识库
  34:## 任务：排查并解决 UniCloud Workspace 云桌面黑屏问题
  68:## 任务：排查并解决 UniCloud Workspace 在 Ubuntu 24.04 上登录即崩溃、无法显示的问题
  101:## 任务：Workspace 24.04 修复补充——libgnutls 重命名不够，需 LD_PRELOAD shim 补齐 H3C 定制符号
  132:## 任务：Workspace 24.04 登录后仍崩溃——恢复 quic/ 捆绑 glib 2.56 + libglib_h3c shim 补齐 glib 2.58+ 符号
  156:## 任务：Workspace 24.04 黑屏——捆绑 GStreamer 1.14 解码链（旧 FFmpeg + libva-x11 符号缺失 + 注册表缓存）

## 二、经验教训汇总（反思素材）

### 来源：20261014.md
  1. **复盘体系跨项目复用只需 3 个核心件**：task_rule.md（规范）+ review_prepare.sh（脚本）+ work_log 系统（数据源），但真正"反思"依赖 task_prompt.md §7 和知识库沉淀目标，缺了复盘会退化成"只出报告不反哺"。
  2. **review_prepare.sh 与 work_log 标题格式强耦合**：脚本靠 `## 任务`/`### 经验教训`/`### 结论`/`待办` 固定标题聚合，work_log 必须沿用 task_rule.md §11.1 模板，否则抓不到内容。
  3. **复制文档要修正路径引用**：从 otc_src 拷贝的 memory/self_evolution.md 含 otc_src 特有知识库文件引用（10_技能与工具、11_工程纪律），应用到新项目须更新为当前知识库结构，否则指针失效。
  4. **知识库按"仓库是什么→技能架构→各分桶→文档发布→工具链→复盘→工程纪律"组织**，比按时间组织更利于检索复用。
  
  1. **云桌面黑屏要按"连接→渲染→进程→解码→环境"逐层排除**，先看日志/实测，不凭猜测。Wayland 不是默认根因。
  2. **云桌面客户端用 GStreamer 解码，缺解码插件是 Linux 高频黑屏原因**。安装 gstreamer1.0-plugins-bad/ugly/libav 解决。
  3. **装了插件但 gst-inspect 找不到** → 删除 `~/.cache/gstreamer-1.0/registry.*` 强制重扫（注册表缓存损坏）。
  4. **DevChat 注入 LD_LIBRARY_PATH**（旧 libstdc++）导致 `GLIBCXX_3.4.29 not found`，用 `env -u LD_LIBRARY_PATH` 规避；排查先 `echo $LD_LIBRARY_PATH`。
  5. **StreamWin 在 -32000,-32000 是 Qt 离屏渲染窗口的隐藏位置，属正常设计**，勿误判为 bug。
  6. **dpkg Installed-Size 单位是 KB**，异常大包用 `du -sk` 实测；单位填错会虚高 1024 倍。
  
  1. **Workspace 在 24.04 崩溃≠22.04 黑屏**：先看崩溃转储/日志判定是崩溃还是黑屏，再对症下药，不要照搬 22.04 的 GStreamer 修复。
  2. **捆绑旧库与系统库版本符号冲突**是升级系统后高频崩溃源：`ldd <so>` 出现 `version 'XXX' not found`，说明运行时 LD_LIBRARY_PATH 优先加载了捆绑旧库。用 `readelf -V` 对比捆绑库与系统库的版本符号。
  3. **排查崩溃优先看 minidump 异常码**：SIGSEGV(0xb)@0x0 = 空指针解引用，结合协议日志定位到具体连接线程。
  4. **修复方式**：重命名捆绑的冲突库（备份 .bak）强制用系统库，比改 LD_LIBRARY_PATH 顺序更精准、副作用小。
  5. **跨版本适配**：同一客户端在 22.04 正常、24.04 崩溃，优先检查捆绑的第三方库（openssl/gnutls/glib）与新版系统库的符号兼容。
  
  1. **"undefined symbol: xxx, version V" 与 "version 'V' not found" 是两种相反问题**：
     - `version 'V' not found` = 加载了太旧的库，缺新符号（→ 用更新的库）
     - `undefined symbol: xxx, version V` = 加载了太新的库，缺定制旧符号（→ 需补符号）
  2. **同一 soname 两个库各缺对方符号**（捆绑旧库 vs 系统新库）时，不能只选一个；用 LD_PRELOAD shim 补齐缺失符号，`.symver` 指定版本 + 转发到另一库实现。
  3. **链接错误不是崩溃**：进程无 minidump 直接退出，stderr 报 symbol lookup error。捕获 VdSession stderr 是关键（start.sh 后台输出 `/tmp/ws_restart*.log`）。
  4. **H3C 定制函数**（如 `gnutls_add_extension_string`）只存在于捆绑库，系统库不可能有，跨版本适配必须保留定制符号。
  
  1. **恢复捆绑库前确认"谁依赖它"**：quic/ glib 2.56 是 H3C QUIC 库编译依赖，禁用后它们用系统 glib 2.80 崩溃。
  2. **崩溃在系统库 = 有库误用系统库**；崩溃在捆绑库 = 捆绑库自身问题。
  3. **nm 合并导出符号一次性找全缺失符号**，避免反复启动逐个报错。
  
  1. **先确认用哪套 GStreamer**（GST_PLUGIN_PATH），再决定装系统插件还是修捆绑链。
  2. **`undefined symbol` 是数据对象**（objdump -T 显示 DO *UND*）= 缺全局变量/回调，禁用不配套捆绑库比补符号干净。
  3. **GStreamer 注册表缓存会缓存加载失败状态**，修复底层后必须删 `~/.cache/gstreamer-1.0/registry.*`。
  4. **`ctypes.CDLL(..., RTLD_NOW)` 能直接复现插件加载错误**；用 `env LD_LIBRARY_PATH=...` 而非改 os.environ。
  

## 三、待验证假设 / 遗留问题（若有）

  - 暂无。
  ---
  
  - 暂无。
  ---
  
  - 需用户重新登录连接，确认桌面能正常显示（修复已就位，等待实测）。
  ---
  
  - 需用户重新登录连接，确认桌面能正常显示（shim 已就位，等待实测）。
  ---
  
  - 需用户登录连接，确认桌面显示（等待实测）。
  
  ---
  
  - 需用户登录连接，确认桌面正常显示（等待实测，已解决）。
  ---

