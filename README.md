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
- `engine/` 渲染引擎（绘制表 IR loader、软件统一内核与渲染语义、Frame Graph）
- `docs/` 设计与实施文档（从 tie-main 收拢；tie-main 侧保留原件）
- `tests/` 探针（gold IR / gold 图 / 双模式等价）

## License

本仓库按 **Tie Public License v2.0（TPL 2.0）** 授权发布（全文见 [LICENSE](LICENSE)）：
你可自由使用、修改并分发本软件源码，包括用于商业产品，仅需保留版权声明并附本许可证。

EN: This repository is released under the **Tie Public License v2.0 (TPL 2.0)**
(full text in [LICENSE](LICENSE)): you may freely use, modify, and redistribute
the source code, including in commercial products, provided you retain the
copyright notice and a copy of the license.
