# UniCloud Workspace 云桌面黑屏问题修复操作文档

> 本文档记录 uniCloud Workspace（workspace-vdi）云桌面客户端在 Linux 下黑屏问题的完整排障与修复过程，供在其他环境遇到类似问题时参考复用。
>
> 适用环境：Linux（Ubuntu 22.04，GNOME 桌面，Intel 核显）
> 客户端版本：UniCloud Workspace E2004H07
> 排障日期：2026-10-14

---

## 1. 问题现象

- workspace-vdi 云桌面客户端打开后**一直黑屏**，无法看到远程桌面内容。
- 客户端能正常登录、连接远程 VM（日志显示连接成功），但画面不显示。
- 切换 Wayland/Xorg 会话后**仍黑屏**（说明不是会话类型问题）。

## 2. 关键环境信息

| 项 | 值 |
|----|-----|
| 操作系统 | Ubuntu 22.04.4 LTS |
| 桌面/会话 | GNOME（Xorg） |
| 内核/GPU | Intel TigerLake-U HD Graphics（iGPU） |
| 云桌面协议 | VDP / SPICE |
| 视频解码 | GStreamer（客户端内置） |
| 客户端安装路径 | `/userdata/UniCloud/Workspace/` |
| 客户端日志 | `/home/<用户>/文档/Workspace/log/Workspace.log` |
| 代理日志 | `/userdata/UniCloud/SpaceAgent/log/SpaceAgent.*.log` |

## 3. 排障流程（逐层排除）

> 核心思路：**按「连接 → 渲染 → 进程 → 解码 → 环境」逐层排查**，先看日志/实测，不凭猜测。

### 3.1 连接层（排除网络/认证问题）
```bash
# 看主日志
tail -50 /home/<用户>/文档/Workspace/log/Workspace.log
```
**关键判断**：出现 `desktop is already connected success now`、`connectVmResult true`、VM 状态 `running` → **连接成功**，排除网络/认证问题。

### 3.2 渲染层（看窗口和画面）
```bash
# 看窗口树，找渲染窗口位置
xwininfo -root -tree | grep -iE 'Vdp|StreamWin'
# 截图 Vdp 主窗口并分析是否全黑
xwd -id <Vdp窗口ID> -out /tmp/vdp.xwd
convert /tmp/vdp.xwd /tmp/vdp.png
convert /tmp/vdp.png -format "mean=%[fx:mean] unique_colors=%k" info:
```
**关键判断**：
- 渲染子窗口 `StreamWin` 在 `-32000,-32000` 是 **Qt 离屏渲染窗口的隐藏位置，属正常设计**，勿误判为 bug。
- Vdp 主窗口 `mean=0, unique_colors=1` → **全黑**，确认黑屏。

### 3.3 进程层（看是否在解码渲染）
```bash
ps -o pid,pcpu,rss,cmd -p $(pgrep -x VdSession)
# 看视频流是否在传输
ss -tinp | grep $(pgrep -x VdSession)
```
**关键判断**：
- VdSession CPU 很低（如 3%）→ **没在解码/渲染**。
- 但某个连接 `bytes_received` 很大（如 23MB）、`lastrcv` 很小（几 ms）→ **视频流在持续传输**。
- 结论：**收到了视频流，但没解码出来** → 问题在解码层。

### 3.4 解码层（看 GStreamer 报错）
```bash
# 查看 VdSession 的 stderr（启动时重定向到的日志）
cat /tmp/ws_start*.log
```
**关键判断**：出现 **GStreamer 管线崩溃**：
```
GStreamer-CRITICAL: gst_element_unlink: assertion 'GST_IS_ELEMENT (src)' failed
GStreamer-CRITICAL: gst_object_ref: assertion 'object != NULL' failed
GStreamer-CRITICAL: gst_bin_remove: assertion 'GST_IS_ELEMENT (element)' failed
```
→ 客户端用 GStreamer 解码视频，但**解码元素创建失败**。

### 3.5 环境层（查 GStreamer 解码器）
```bash
gst-inspect-1.0 --version
# 查是否有 H.264/H.265 解码器
gst-inspect-1.0 | grep -iE 'avdec_h264|vaapih264dec'
# 查已装的 gstreamer 插件包
dpkg -l | grep -iE 'gstreamer.*(bad|ugly|good|libav)'
# 查 VAAPI 硬件解码（Intel）
vainfo
```
**关键判断**：系统只装了 `gstreamer1.0-plugins-good`，**缺 bad/ugly/libav**；`gst-inspect-1.0` 无 `avdec_h264`/`vaapih264dec` → **缺少视频解码插件**。

## 4. 根因

**系统缺少 GStreamer 视频解码插件**（`gstreamer1.0-plugins-bad/ugly/libav`），客户端（VdSession）用 GStreamer 解码远程桌面的 H.264 视频流，但**无法创建解码元素** → 解码管线构建失败（gst_* 断言崩溃）→ 视频流无法解码 → **黑屏**。

> 补充：早期误判为"Wayland 会话渲染问题"，切换 Xorg 后仍黑屏，才转向解码层排查。**Wayland 不是根因**。

## 5. 解决方案（三步，需按序执行）

### 步骤 1：安装 GStreamer 视频解码插件
```bash
sudo apt install gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-vaapi vainfo
```

### 步骤 2：重建 GStreamer 插件注册表
> 装了插件但 `gst-inspect-1.0` 仍找不到解码器时，需删除损坏的注册表缓存强制重扫。
```bash
rm -f ~/.cache/gstreamer-1.0/registry.*
# 验证解码器已注册
gst-inspect-1.0 avdec_h264     # 应显示 Factory Details
gst-inspect-1.0 vaapih264dec   # 应显示 Factory Details
```

### 步骤 3：用干净环境重启客户端
> 若从终端启动，需清空被注入的 `LD_LIBRARY_PATH`（见 §7 常见坑），否则依赖 libstdc++ 的库加载失败。
```bash
# 先终止旧进程
kill $(pgrep -x Workspace) $(pgrep -x VdSession)
# 用干净环境启动
env -u LD_LIBRARY_PATH DISPLAY=:0 /userdata/UniCloud/Workspace/run.sh
```
> 从桌面快捷方式正常启动时无 `LD_LIBRARY_PATH` 干扰，可省略 `env -u LD_LIBRARY_PATH`。

## 6. 验证结果

```bash
# 1. VdSession stderr 中 GStreamer-CRITICAL 错误应消失
grep -iE 'GStreamer|gst_|CRITICAL' /tmp/ws_start*.log   # 应无严重错误

# 2. Vdp 窗口画面恢复（不再全黑）
xwd -id <Vdp窗口ID> -out /tmp/vdp2.xwd
convert /tmp/vdp2.xwd /tmp/vdp2.png
convert /tmp/vdp2.png -format "mean=%[fx:mean] unique_colors=%k" info:
# 修复前: mean=0, unique_colors=1（全黑）
# 修复后: mean≈0.95, unique_colors≈106830（画面正常）
```
**实测结果**：VdSession 的 GStreamer 严重错误消失；Vdp 窗口 `mean 0→0.95`、`颜色 1→106830`，**远程桌面正常显示**。

## 7. 常见坑与注意事项

1. **Wayland 不是默认根因**：云桌面黑屏要按「连接→渲染→进程→解码→环境」逐层排除，别过早切会话。
2. **装了插件但 gst-inspect 找不到**：删除 `~/.cache/gstreamer-1.0/registry.*` 强制重扫（注册表缓存可能损坏/过期）。
3. **DevChat 等扩展注入 LD_LIBRARY_PATH**：会指向旧版 `libstdc++.so.6`，导致 `GLIBCXX_3.4.29 not found`，使 VAAPI 驱动、GStreamer 插件等加载失败。
   ```bash
   echo $LD_LIBRARY_PATH   # 排查是否被注入
   env -u LD_LIBRARY_PATH <命令>   # 规避
   ```
4. **StreamWin 在 -32000,-32000**：是 Qt 离屏渲染窗口的隐藏位置，**属正常设计**，不是 bug。
5. **多屏模式**：若客户端开多屏（`IsMultiScreen=true`）且渲染异常，可临时改单屏（`IsMultiScreen=false`）测试，改前备份 `vdisettings.ini`。
6. **高分辨率/双屏（2560x1440）**：会加大渲染负担，排障时可先降分辨率/关副屏。

## 8. 其他环境适配说明

- **日志路径**：客户端日志在 `/home/<用户>/文档/Workspace/log/`，不同版本/主机路径可能不同，用 `find / -iname 'Workspace.log'` 定位。
- **安装路径**：客户端在 `/userdata/UniCloud/Workspace/`，启动脚本为 `run.sh` → `RUN/start.sh`。
- **GStreamer 插件包名**：Ubuntu 为 `gstreamer1.0-plugins-bad/ugly/libav`；Debian 类似；其他发行版用对应包管理器安装等价插件。
- **硬件解码**：Intel 核显用 VAAPI（`gstreamer1.0-vaapi` + `vainfo`）；NVIDIA 需对应的 VAAPI/VDPAU 驱动。
- **若以上无效**：可能需更新客户端版本，或联系厂商（紫光云/华为）确认该版本的已知问题。

## 9. 参考链接

- 排障原始记录：`task/online_task/online_bug_log.md`（本项目内）
- 知识库经验：`study/knowledge_base_system/10_系统排障与经验.md`（本项目内）
- 客户端官方文档：UniCloud Workspace 用户手册（厂商提供）