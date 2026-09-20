# 单张纸直线裁切试排工具（Sheet Layout）

面向小型印刷店的**单张纸试排 / 批量排产 / 多纸型成本排产**网页：输入纸张尺寸、刀缝宽度和多种成品（宽 × 高 × 数量），
自动生成**断头台式（guillotine）直线裁切布局**——既可试排一张纸，也可设定用纸上限逐张排完整批需求，
还可在 2～8 种候选纸张之间按**单位已排成品面积成本**逐轮择优、混合用纸完成整批生产。

## 裁切规则

- 每一刀都沿直线**切透当前矩形区域，将其一分为二**；
- 成品允许旋转 90°（图中以 `⟳90°` 标注，放置尺寸与规格互换）；
- **刀缝按实际宽度占位，切割不侵占任何成品**（刀缝带与成品只共边、不重叠）；
- 排不下的件数按成品规格单独列出；
- 输出 SVG：成品块（名称、规格、旋转状态、件号）、带序号的裁切中心线（序号即下刀顺序）、
  余料区，以及已排面积占比、刀缝面积、余料面积、总刀数等统计。

排样采用“大件优先 + 最佳适配（BSSF 变体）+ 较短余料优先切分”的贪心断头台启发式算法，
属快速试排工具，不保证全局最优。

## 批量排产

在「排产模式」中选择**批量排产**并设置**最多用纸张数**后，服务端按同一套断头台规则**逐张排样**：
每排完一张纸，就从剩余需求中扣除该张已排数量，再排下一张，直到满足任一终止条件：

| 终止原因 `stopReason` | 含义 |
| --- | --- |
| `completed` | 全部成品排完 |
| `max-sheets` | 已用完设置的最多纸张数，仍有剩余 |
| `no-progress` | 某张纸一件成品都排不进，**立即止损**（该空白纸不计入结果，绝不输出空白纸、不会死循环） |

接口返回每张纸各自完整的布局（`sheets`）、剩余件数（`remaining`），
以及汇总（`summary`）：用纸数、完成率、总刀数、已排/总件数。
前端可点击「第 N 张」标签逐张查看布局与下刀顺序，并显示终止原因横幅。

## 多纸型成本排产

在「排产模式」中选择**多纸型成本排产**后，录入 **2～8 种候选纸张**：
名称（可留空，自动命名为「纸型 N」）、宽、高、**单价（元/张，必须 > 0）**、
**可用张数（0～500 的整数，0 表示该纸型暂无库存）**。

服务端每轮对所有「仍有库存 **且** 能放入至少一件余件」的纸型调用现有单张排样，
按以下顺序选出下一张纸：

1. **单位已排成品面积成本最低**：`单价 / 本张已排成品面积`（元/mm²，接口以 元/m² 返回 `costPerM2`）；
2. 同分时**本张已排件数多**者优先；
3. 再同分按**录入顺序**靠前者优先。

选中后扣减该纸型库存 1 张、按该张布局扣减剩余需求，进入下一轮，直到满足任一终止条件：

| 终止原因 `stopReason` | 含义 |
| --- | --- |
| `completed` | 全部成品排完 |
| `stock-exhausted` | 所有纸型库存均已耗尽，仍有剩余 |
| `no-progress` | 仍有库存的纸型**全部**排不进任何剩余成品，立即止损（不扣库存、不生成空白纸） |

> **空白纸绝不计入结果**：某纸型本轮一件都排不进时不参与选择、不扣库存；
> 只有确实排入成品的纸才会出现在 `sheets` 中。循环上界为各纸型库存之和（≤ 8×500），
> 每张至少排入一件，算法必然终止。

接口 `POST /api/layout/cost` 返回：

- `stopReason`：终止原因（上表三种之一）；
- `sheets`：**逐张**结果，每张含轮次 `round`、纸型（`paperId`/`paperName`/`paperWidth`/`paperHeight`）、
  单价 `price`、本张布局 `layout`（结构同 `/api/layout`），以及选择指标
  `piecesPlaced`、`placedArea`（mm²）、`costPerM2`（元/m²），
  `selection.ranked` 列出本轮**全部可行候选**的同名指标与 `selected` 标记；
- `usage`：**各纸型用量**——库存、已用张数、剩余库存、已排面积、小计成本；
- `summary`：总轮次/用纸数、用纸种类 `paperKindsUsed`、已排/总件数、完成率、总刀数、
  **总成本 `totalCost`（元）**、已排成品总面积（mm² 与 m²）、综合单位成本 `effectiveCostPerM2`；
- `remaining`：仍未排入的成品（含原始需求 `total`、已排 `placed`、剩余 `quantity`）。

前端提供：纸型录入表格、每张纸上的**混合用纸标签**（颜色与纸型对应）、
**各纸型用量与成本汇总表**、每轮候选纸型的指标对比表，以及终止原因横幅与剩余件数表。

## 单条命令启动（需要 Docker + Docker Compose）

```bash
docker compose up -d --build
```

启动后浏览器访问： **http://localhost:3000**

停止：

```bash
docker compose down
```

镜像构建时会通过 `npm ci` 自动备齐全部依赖（仅 Express），**无需任何外部服务或数据库**。

### 必要配置

| 配置 | 方式 | 默认值 | 说明 |
| --- | --- | --- | --- |
| 宿主机访问端口 | 环境变量 `PORT` | `3000` | 如 `PORT=8080 docker compose up -d --build`，则访问 http://localhost:8080 |

容器内固定监听 3000 端口，不挂载卷、不依赖外部存储。

## 本地直接运行（不用 Docker）

要求 Node.js ≥ 18。

```bash
npm install
npm start          # 默认 http://localhost:3000，可用 PORT=8080 npm start 改端口
```

## 使用说明

1. 填写纸张宽、高（mm）和刀缝宽度（mm，可为 0），也可从“常用纸度”下拉选择 A3/A2/A1、正度/大度全张；
2. 选择排产模式：
   - **单张试排**（默认）；
   - **批量排产**（需设置最多用纸张数，1–500）；
   - **多纸型成本排产**（在纸型表格中录入 2～8 种候选纸张的名称、宽高、单价和可用张数）；
3. 在成品清单中逐行填写名称（可留空）、宽、高、数量，可随时增删行；
4. 点击「生成裁切方案 / 生成排产方案 / 生成成本排产方案」：
   - 左侧/下方 SVG 显示纸面布局；红色虚线带圆形序号即裁切中心线，按序号顺序逐刀切割；
   - 批量模式下点击「第 N 张」标签切换查看每张纸的布局，顶部横幅显示终止原因与汇总；
   - 成本模式下标签显示每张纸的轮次与纸型名称（颜色与混合用纸标签一致），
     页面展示成本汇总、各纸型用量表与本轮候选指标对比表；
   - 「下刀顺序」表列出每一刀的方向（竖切/横切）、被切区域和切线长度；
   - 「排不下的成品 / 未排完的成品」表列出每种规格的已排/需求数量和剩余件数。

## HTTP 接口

- `GET /api/health` — 健康检查，返回 `{"ok":true}`
- `POST /api/layout` — 单张排样
- `POST /api/layout/batch` — 批量排产（在单张入参基础上增加 `maxSheets` 字段）
- `POST /api/layout/cost` — 多纸型成本排产（入参为 `kerf`、`papers`、`products`）

单张请求示例：

```json
{
  "paperW": 889,
  "paperH": 1194,
  "kerf": 3,
  "products": [
    { "name": "封面", "width": 210, "height": 285, "quantity": 12 },
    { "name": "名片", "width": 90, "height": 54, "quantity": 80 }
  ]
}
```

返回包含 `paper`、`stats`（含 `areaRatio` 已排面积占比 0–1）、`items`（成品摆放坐标与旋转状态）、
`cuts`（刀序、方向、被切区域、刀缝带、切线长）、`frees`（余料矩形）、`unplaced`（排不下的件数）。
非法输入返回 HTTP 400 与错误信息。

批量请求示例：

```json
{
  "paperW": 889,
  "paperH": 1194,
  "kerf": 3,
  "maxSheets": 10,
  "products": [
    { "name": "封面", "width": 210, "height": 285, "quantity": 40 },
    { "name": "名片", "width": 90, "height": 54, "quantity": 300 }
  ]
}
```

返回包含 `paper`、`stopReason`（`completed` / `max-sheets` / `no-progress`）、
`summary`（`sheetsUsed` 用纸数、`piecesPlaced`/`piecesTotal`、`completionRate` 完成率 0–1、`cutsTotal` 总刀数）、
`sheets`（每张纸各自完整的单张排样结果，结构同 `/api/layout` 返回）、
`remaining`（仍未排入的成品，结构同 `unplaced`）。

多纸型成本排产请求示例：

```json
{
  "kerf": 3,
  "papers": [
    { "name": "大度全张", "width": 889, "height": 1194, "price": 3.5, "stock": 10 },
    { "name": "A3", "width": 297, "height": 420, "price": 0.4, "stock": 50 }
  ],
  "products": [
    { "name": "封面", "width": 210, "height": 285, "quantity": 40 },
    { "name": "名片", "width": 90, "height": 54, "quantity": 300 }
  ]
}
```

返回包含 `stopReason`（`completed` / `stock-exhausted` / `no-progress`）、
`sheets`（逐张：`round`、纸型信息、`price`、`piecesPlaced`、`placedArea`、`costPerM2`、
`selection.ranked` 本轮全部可行候选指标、`layout` 完整单张布局）、
`usage`（各纸型 `stock`/`used`/`remainingStock`/`placedArea`/`cost`）、
`summary`（`sheetsUsed`、`paperKindsUsed`、件数与完成率、`cutsTotal`、
`totalCost` 总成本、`totalPlacedAreaM2`、`effectiveCostPerM2` 综合单位成本）、
`remaining`（剩余成品）。非法输入（纸型少于 2 种/多于 8 种、单价 ≤ 0、张数越界等）返回 HTTP 400。

## 项目结构

```
.
├── server.js            # Express 入口：静态页面 + /api/layout + /api/layout/batch + /api/layout/cost
├── src/packer.js        # 断头台二分排样核心算法（单张 layout + 固定纸型批量 layoutBatch）
├── src/cost.js          # 多纸型成本排产（逐轮试排候选、单位面积成本择优、库存与成本核算）
├── public/
│   ├── index.html       # 单页界面（单张/批量/多纸型成本三种模式）
│   ├── style.css
│   └── app.js           # 表单交互 + SVG 渲染 + 逐张查看 + 成本汇总（原生 JS）
├── test/smoke.test.js   # 单张/固定纸型批量算法不变量测试（node:test，无额外依赖）
├── test/cost.test.js    # 多纸型成本排产测试：择优规则、同分顺序、库存、终止原因、校验
├── Dockerfile
└── docker-compose.yml
```

## 测试

```bash
npm test
```

覆盖：成品不重叠且在纸面内、刀缝不侵占成品、旋转放置、二分区域几何一致、
面积守恒（成品 + 刀缝 + 余料 = 纸面）、未排件确实放不进任何余料区、非法输入校验，
以及批量排产的逐张扣减、上限停止、无进展止损（不产生空白纸）、空需求与参数校验等。

多纸型成本排产另覆盖：单位面积成本低者优先（与单价高低无关）、
同分时已排件数多者优先、再同分按录入顺序、逐轮库存/需求扣减、
库存耗尽终止、全部无进展止损且不扣库存、零库存纸型不可选、
每轮入选者指标确为最小、用量与总成本汇总守恒、空需求、以及
纸型数量（2～8）、单价（>0）、张数（0～500 整数）等参数校验。
