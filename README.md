# tiu

**tie 生态独立自研 UI 框架** / *The tie-language in-house UI framework*

tiu 是 tie 生态的 UI 框架（定位 2026-09-11 定）：**独立、高性能、跨平台、不依赖 trm**；
可与 trm 同用。三层架构：渲染引擎（Engine）/ 绘制 API 库（Drawing API）/ UI 控件（UI
widgets）；层间只走数据（绘制表 / 脏矩形 / 事件轴独立），不穿越代码引用。

*EN: tiu is the tie ecosystem's UI framework — independent, high-performance,
cross-platform, NOT depending on trm; usable alone or with trm. Three layers:
render engine / drawing API library / UI widgets; layers communicate via data
only (draw-list / dirty rect / event axis), never code references.*

## 设计 / Design

- [tiu-render-engine.md](docs/designs/tiu-render-engine.md)（渲染引擎七层架构，ROAD p.9.4.1）
- [tiu-drawing-api.md](docs/designs/tiu-drawing-api.md)（绘制 API 库对象模型与双模式）
- [tiu-event-system.md](docs/designs/tiu-event-system.md)（事件轴：输入 / 命中 / 焦点）

## 实施 / Implementation Plans

- [2026-09-12-tiu-api-impl.md](docs/plans/2026-09-12-tiu-api-impl.md)（API 库实施计划，契约冻结）
- [2026-09-12-tiu-render-impl.md](docs/plans/2026-09-12-tiu-render-impl.md)（渲染引擎实施计划，后端顺序）

## 构建 / Build

tiu 全部以 **tie 语言**实现，用自举 tiec 编译（要求本机克隆
[tie-lang/tiec](https://github.com/tie-lang/tiec) 为 tiu 的兄弟目录，用
`tiec/compiler/tiec.exe` 自举编译）：

```powershell
..\tiec\compiler\tiec.exe --no-cache api\src\xxx.tie
..\tiec\compiler\tiec.exe --no-cache engine\src\xxx.tie
```

探针（验收门禁）同样以 tie 编写、tiec 编译运行：`tests\probe_*.tie` 全部 PASS 即模块闭环。

## 内容 / Contents

- `api/` 绘制 API 库（图元 / Path / Paint / Layer 对象模型、IR 编码器、Canvas 会话与双模式）
- `engine/` 渲染引擎（绘制表 IR loader、软件统一内核与渲染语义、帧缓冲导出）
- `ui/` 控件框架（骨架 / 声明式构建复用 / 约束布局 / 主题 token / 差分桥 / 命中对接 / 绘制桥）
- `host/` 平台壳（窗口 / 消息泵 / 上屏 / 事件；tie 绑定的最小平台层，见下）
- `examples/` 示例应用（可运行的窗口程序，见下）
- `docs/` 设计与实施文档（从 tie-main 收拢；tie-main 侧保留原件）
- `tests/` 探针（gold IR / gold 图 / 双模式等价 / 端到端可见闭环 / 交互闭环）

## 示例 / Examples

- [`examples/counter.tie`](examples/counter.tie) —— **计数器应用**：声明式界面 + 绝对定位 +
  主题切换 + 按钮三态（悬停 / 按下）+ 状态驱动重绘。启动后先自演示一遍（合成点击），
  之后可手动交互，按 `ESC` 或关闭窗口退出。

一键构建并运行（tiu 仓根；需先按上节准备好工具链）：

```powershell
..\tiec\compiler\tiec.exe examples\build.tie -o build\example_build.exe
build\example_build.exe
```

EN: `examples/counter.tie` is a runnable counter app wiring all layers together
(declarative build, absolute layout, theme switching, three-state buttons,
state-driven redraw). It self-demos with synthetic clicks on start, then takes
real input; ESC or the close button quits.

## 平台壳 / Host layer

三层（UI / API / engine）之外是最小平台壳 `host/`：**窗口 / 消息泵 / 上屏 / 事件队列**。
C 壳（`host/win32/tiu_host.cpp`）收编自 ext/gfx 的 Win32 嵌入层（p.6.8.9–6.8.10），
收编时逐字节核验一致。

**为什么窗口壳必须有 C 端**：tie 无法把函数值作为 extern 形参传递
（实测 `error[E00041]`：extern 形参仅接受标量 / string / ptr / slice），故 C 回调
不可用、WndProc 无法用 tie 写。窗口状态机因此收敛在 C 端；tie 侧只做绑定与编排。
**演进方向**：编译器支持 C 回调后，本 C 壳整体退役、平台层转纯 tie。

EN: Beyond the three layers sits the minimal platform host (window / message pump /
present / event queue). The C shell is absorbed from ext/gfx's Win32 embedding layer;
it exists because tie cannot pass a function value as an extern parameter, so a
WndProc callback is impossible in pure tie. The shell retires once the compiler gains
C-callback support.

### 构建 host 层 / Build the host layer

```powershell
..\tiec\compiler\tiec.exe host\build.tie -o build\host_build.exe
build\host_build.exe
```

构建驱动（tie 写）链路：`clang-cl` 编 C 壳 → `tiec --emit-ir` 编探针 →
`clang -c` → `clang -fuse-ld=link` 链接（`+ ..\trm-lite\trm_lite.a` 表运行时、
系统库 user32/gdi32/shell32）。产物落在 `build/`（不入库）。

## License

本仓库按 **Tie Public License v2.0（TPL 2.0）** 授权发布（全文见 [LICENSE](LICENSE)）：
你可自由使用、修改并分发本软件源码，包括用于商业产品，仅需保留版权声明并附本许可证。

EN: This repository is released under the **Tie Public License v2.0 (TPL 2.0)**
(full text in [LICENSE](LICENSE)): you may freely use, modify, and redistribute
the source code, including in commercial products, provided you retain the
copyright notice and a copy of the license.
