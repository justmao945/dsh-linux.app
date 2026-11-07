# dsh-web

Linux 桌面壳：后台跑一个 `dsh web` 服务进程，前台只有一个 chromium 无边框 app 窗口，
看起来、用起来都像一个本地 app。关窗口即停服务；服务崩了窗口跟着退出；重复启动不会
双开（装了 `wlrctl` 时还会把已有窗口拉到前台）。

## 用法

```sh
bin/install   # 装到 ~/.local/bin + 应用启动器（无需 root）
dsh-web       # 或从启动器（Super+Space）点 "DSH Web"
```

额外参数原样转发给 `dsh web`，例如 `dsh-web --patch ./extra.yml`。

## 结构

| 文件 | 作用 |
| --- | --- |
| `bin/dsh-web` | 包装脚本：定位 dsh CLI → `dsh web --no-open --port 0`（setsid 独立进程组）→ 轮询 stdout 里的 URL → `chromium --app=<URL>` 开唯一窗口 → 任意一侧退出就收掉另一侧 |
| `bin/install` | 用户级安装（二进制、`.desktop`、图标） |
| `pkgbuild/PKGBUILD` | 系统级打包（`makepkg -fsi`，装到 `/usr`） |
| `pkgbuild/dsh-web.desktop` | 启动器条目（`StartupWMClass=dsh-web` 对应窗口 class） |
| `pkgbuild/dsh-web.svg` | 图标 |

## 设计取舍

- **chromium `--app` 而非 Tauri/Electron/Qt WebEngine**：Omarchy 已自带 chromium；
  qt6-webengine 约 500MB 依赖，Tauri 需要引入 cargo-tauri 与 Rust 构建链。壳的本质就是
  "一个指向本地 URL 的窗口"，`--app` 模式零浏览器 UI、独立 `user-data-dir`、
  profile 目录名决定 Wayland app id，任务栏/启动器里就是独立应用。
- **`--port 0`** 由 OS 挑空闲端口，从 stdout 解析 `dsh web: http://…/?token=…`，永不端口冲突。
- **单实例**：`$XDG_RUNTIME_DIR/dsh-web.lock` 上的 `flock`；抢不到锁就尝试聚焦已有窗口再退出。
  聚焦优先用 `wlrctl toplevel focus`（`omarchy-pkg-add wlrctl`，可选依赖）；hyprctl 只作
  兜底——Hyprland 0.56.x 的 Lua 配置下 `hyprctl dispatch` 字符串派发有上游 bug
  （hyprwm/Hyprland discussion #16224），聚焦可能静默失效，但不影响"不双开"。
- **窗口识别**：chromium 在 Wayland 下的 app id 由 URL 主机名 + profile 目录名拼成
  `chrome-127.0.0.1__-dsh-web`（`--profile-directory=dsh-web`，`--class` 只对 X11 生效），
  `.desktop` 的 `StartupWMClass` 与之对齐。
- **进程组清理**：dsh web 用 `setsid` 起在独立进程组，TERM 先、KILL 兜底，不留孤儿 node 进程。
- **运行时状态**：日志在 `~/.local/state/dsh-web/web.log`，chromium 独立 profile 在
  `~/.local/state/dsh-web/chromium/`（与日常浏览器互不干扰，且带会话 token，不与他人共享）。
