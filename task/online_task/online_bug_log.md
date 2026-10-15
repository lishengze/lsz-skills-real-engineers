# 线上问题分析日志（online_bug_log）

> 每次分析完线上问题，都必须把「分析时间、问题、结论」追加到本文件末尾（不删除、不覆盖，保持连续可追溯）。

---

## 问题记录

### 2026-10-14 10:15 — UniCloud Workspace 云桌面客户端黑屏

**问题**：workspace-vdi（UniCloud 云桌面客户端，版本 E2004H07）打开后一直黑屏，无法看到远程桌面内容。

**影响范围**：用户 lsz 的 Ubuntu NUC（Intel TigerLake UHD 核显），GNOME 桌面。

**分析过程**：
1. 定位日志：
   - `/home/lsz/文档/Workspace/log/Workspace.log`（主日志，5017 行）
   - `/userdata/UniCloud/SpaceAgent/log/SpaceAgent.20261014.log`（代理日志）
   - `/userdata/UniCloud/Workspace/RUN/debug.log`（CEF/Chromium 日志）
2. 日志关键点：
   - `09:24:53 launch vm fail 9899`：首次启动 VM 失败（瞬时，客户端自动重试）。
   - `09:25:00 desktop is already connected success now`：**桌面连接实际成功**，VM 为 Windows 10 LTSC，running 状态。
   - `09:27:06 Got goaway`：gRPC 收到 GOAWAY，服务器断开连接。
   - debug.log 大量 `InitializeSandbox() called with multiple threads in process gpu-process`（CEF GPU 进程错误）+ 10-08 出现 `X connection error received`。
   - 客户端含 `libspice-client-glib-2.0.so`，协议 `vdp`，`spicePreferProto:2`。
3. 环境排查：
   - `XDG_SESSION_TYPE=wayland`，`WAYLAND_DISPLAY=wayland-0` —— **系统运行在 Wayland 会话**。
   - 客户端显示配置走 `XWAYLAND0/1`（Wayland 下的 XWayland 兼容层）。
   - 有 Xorg 会话可用（`/usr/share/xsessions/ubuntu-xorg.desktop`）。
   - GDM 配置了**自动登录**（`AutomaticLogin=lsz`），走 Wayland。

**结论（根因）**：
- 黑屏根因是 **Wayland 会话下，UniCloud Workspace 客户端（VDP/SPICE 协议，依赖 X11+OpenGL 渲染）通过 XWayland 兼容层渲染失败**，导致桌面连接成功但画面不显示。
- 佐证：连接成功（非网络/认证问题）、CEF GPU 进程错误、X 连接错误、客户端走 XWayland。
- 9899 为首次启动 VM 瞬时失败，非黑屏根因。

**解决方案**：
1. **首选**：切换登录会话到 Xorg：
   - 因配置了自动登录（无登录界面可选），需改 `/etc/gdm3/custom.conf`，在 `[daemon]` 下加 `WaylandEnable=false`（禁用 Wayland），或设置 `DefaultSession=ubuntu-xorg.desktop`，然后重启图形会话/系统。
   - 切换到 Xorg 后，SPICE/X11 渲染正常，黑屏应解决。
2. **替代**：若客户端支持，关闭硬件加速/切换软件渲染模式。
3. 验证：切换 Xorg 后重连桌面。

**处理状态**：已定位根因，解决方案已给出；**待用户确认是否执行 Xorg 会话切换**（修改 GDM 配置 + 重启图形会话为高风险操作，未擅自执行）。

---

### 2026-10-14 10:40 — UniCloud Workspace 黑屏（续：真正根因确认 + 已解决）

**问题**：切换 Xorg 会话后仍黑屏，继续深入排查。

**进一步分析**（关键证据）：
1. 窗口树显示：`Vdp` 主窗口在 `+0+0`（2560x1440），但渲染子窗口 `StreamWin` 在 `-32000,-32000`（Qt 离屏隐藏位置）。用 xwd 截图分析：Vdp 窗口 `mean=0, unique_colors=1`（**全黑**）。
2. VdSession 进程 CPU 仅 3.3%（无解码渲染），但 fd=127 连接已接收 **23MB 数据**（`lastrcv:6`，视频流在持续传输）。
3. **VdSession stderr 关键错误**（GStreamer 解码管线崩溃）：
   ```
   gst_element_unlink: assertion 'GST_IS_ELEMENT (src)' failed
   gst_object_ref: assertion 'object != NULL' failed
   gst_bin_remove: assertion 'GST_IS_ELEMENT (element)' failed
   ```
4. **环境检查**：系统只装了 `gstreamer1.0-plugins-good`，**缺少 bad/ugly/libav 视频解码插件**；GStreamer 中无 `avdec_h264`/`vaapih264dec` 解码器。
5. 附带发现：当前 shell 的 `LD_LIBRARY_PATH` 被 VS Code DevChat 扩展注入（`~/.vscode/extensions/gtht.devchat-*/.../libstdc++.so.6`，旧版无 GLIBCXX_3.4.29），导致所有依赖 libstdc++ 的库（VAAPI 驱动、GStreamer 插件）加载失败。**客户端从桌面启动时无此干扰**（用户正常启动路径）。

**最终根因**：
- 黑屏真正根因是 **系统缺少 GStreamer 视频解码插件**（gstreamer1.0-plugins-bad/ugly/libav），客户端（VdSession）用 GStreamer 解码远程桌面 H.264 视频流，但无法创建解码元素 → 解码管线构建失败（gst_* 断言崩溃）→ 视频流无法解码 → 黑屏。
- **Wayland 不是根因**（切换 Xorg 后仍黑屏）。早期"Wayland 渲染失败"结论不成立，已修正。
- StreamWin 在 -32000 是 Qt 离屏渲染窗口的隐藏位置，属正常设计，非根因。

**解决方案（已执行并验证成功）**：
1. 安装 GStreamer 视频解码插件：`sudo apt install gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-vaapi vainfo`。
2. 删除损坏的 GStreamer 注册表缓存：`rm -f ~/.cache/gstreamer-1.0/registry.*`，强制重新扫描（否则 avdec 解码器不注册）。
3. 用干净环境重启客户端：`env -u LD_LIBRARY_PATH DISPLAY=:0 /userdata/UniCloud/Workspace/run.sh`（避免 DevChat 注入的旧 libstdc++ 干扰）。
4. **验证**：VdSession stderr 中 GStreamer-CRITICAL 错误消失；Vdp 窗口截图 `mean=0→0.95, unique_colors=1→106830`，**远程桌面画面正常显示**。

**处理状态**：✅ **已解决**。根因 = GStreamer 视频解码插件缺失；已安装插件 + 重建注册表 + 干净环境重启客户端，画面恢复正常。截图存于 `/home/lsz/桌面/workspace_vdi_fixed.png`。

**遗留**：
- 客户端当前以 `IsMultiScreen=false`（单屏）运行（此前排查时临时修改，备份 `vdisettings.ini.bak.20261014`）。如需恢复双屏，可改回 `IsMultiScreen=true` 后重启客户端验证。
- 建议客户端通过桌面快捷方式正常启动（避免 DevChat 的 LD_LIBRARY_PATH 干扰）；若从终端启动需 `env -u LD_LIBRARY_PATH`。
### 2026-10-15 09:00 — UniCloud Workspace 云桌面黑屏（22.04 捆绑 GStreamer 解码链缺 libva/fftw3）

**问题**：workspace-vdi（E2004H07）在 Ubuntu 22.04.5 上启动后黑屏，远程桌面画面不显示。VdSession 进程在跑、连接成功，但画面黑屏。

**影响范围**：用户 lsz 的 Ubuntu 22.04.5 主机。

**分析过程**：
1. 系统 22.04.5 + Workspace E2004H07，VdSession 进程运行（PID 12788），连接日志 `desktop is already connected success now`。
2. 解码层：`gst-inspect-1.0 avdec_h264` 报"没有那样的组件或插件"。
3. 关键：`start.sh` 设 `GST_PLUGIN_PATH=RUN/gstreamer-1.0`（**捆绑 GStreamer 1.14.5**），VdSession 用捆绑 GStreamer，非系统。捆绑目录有 `libgstlibav.so`（avdec_h264 提供者）、`libgstvaapi.so`。
4. 用 VdSession 完整 LD_LIBRARY_PATH 复现 `ctypes.CDLL('gstreamer-1.0/libgstlibav.so', RTLD_NOW)` 报 **`libva.so.2: cannot open shared object file`**。
5. 深入：捆绑 `libgstlibav.so`/`libgstvaapi.so` 依赖 `libva.so.2`（核心 libva），但捆绑 RUN 目录**没有** libva.so.2（只有 libva-x11/drm/wayland 变体），系统 22.04 **未安装 libva**。
6. 捆绑 `libva-x11.so.2.100.0` 需要 `va_fool_postp`（数据符号 `DO *UND*`），任何捆绑库都不定义它，只能由核心 libva.so.2 提供。
7. 装 libva 后又暴露 `libfftw3.so.3`（双精度 FFTW）缺失，系统只有单精度 `libfftw3-single3`。

**结论**：
- **根因**：VdSession 用捆绑 GStreamer 1.14，其解码插件 `libgstlibav.so`（avdec_h264）依赖 `libva.so.2` + `libfftw3.so.3`，而捆绑目录缺核心 libva.so.2、系统 22.04 也未装 libva（且只装单精度 fftw3）→ 插件加载失败 → avdec_h264 元素创建失败 → 黑屏。
- **解决方案**（参考知识库 §8，多一步装系统 libva）：
  1. 装系统 libva 全套：`sudo apt install libva2 libva-x11-2 libva-drm2 libva-wayland2`（提供 libva.so.2 + 系统配套 libva-x11）
  2. 禁用捆绑 libva-x11（`mv libva-x11.so* .bak`），用系统配套（无 va_fool_postp 需求）
  3. 装 `libfftw3-double3`（提供 libfftw3.so.3）
  4. 删 GStreamer 注册表缓存：`rm -f ~/.cache/gstreamer-1.0/registry.*`
  5. 干净环境重启：`env -u LD_LIBRARY_PATH DISPLAY=:0 run.sh`
- **验证**：`ctypes.CDLL('libgstlibav.so')` 加载 OK；捆绑 GStreamer 下 `gst_element_factory_make('avdec_h264')` 返回 OK；Workspace 重启后稳定运行，日志无 GStreamer 解码错误。
- **处理状态**：✅ 修复已就位，待用户重新登录连接实测确认画面正常。
