# 视频多维度质量分检视台 (Video Multi-Dimension Quality QC Dashboard)

> A browser-based, dependency-free dashboard for inspecting video quality scores along multiple axes: **brightness / motion / saturation + Video Type / Video Content classification**, with per-author drill-down and CSV import/export.

## 功能特性

- **完全在浏览器运行**：CSV 上传后用 PapaParse 解析，无需后端，可纯静态托管在 GitHub Pages。
- **多维度阈值实时调节**：
  - `brightness_metrics.score` 1-10，默认 `[4, 8]`
  - `motion_metrics.score` 1-5，默认 `[4, 5]`
  - `saturation_metrics.score` 1-5，默认 `[2, 4]`
- **分类筛选**：Video Type 5 子维度（Camera Perspective / Primary Subject / Interaction Mode / Motion Intensity / Capture Method）+ Video Content 多选。
- **作者筛选**：基于 `requirement_info.author` 的 second-pass 下拉。
- **采样方式**：随机采样 OR channel 分层采样（每个 channel 最多 10 条）。
- **统计面板**：
  - 日期范围内总样本数 / 满足阈值 / 不满足阈值
  - 全局 brightness / motion / saturation 档位分布柱状图（基于 ECharts）
  - 内嵌 score 评分标准说明（参考算子源代码）
- **视频卡片**：5 列网格 + 懒加载（IntersectionObserver）+ 质量分徽章 + meta JSON 折叠 + 文件名复制。
- **CSV 导出**：导出当前阈值下全部匹配样本（不受采样上限影响），文件名含日期 + 阈值参数便于追溯。

## 数据要求

CSV 必须包含以下列（来自视频质量分算子打标输出）：

- `raw_id`: 视频唯一 ID
- `project_name`: 项目名（后缀必须是 `_yyyymmddhh` 形式，作为日期下拉选项）
- `video_url`: 完整签名链接
- `dur`: 视频时长（秒）
- `result`: JSON 字符串，包含：

```json
{
  "requirement_info":     { "channel": "...", "page_url": "...", "author": "..." },
  "classfication_result": { "Video Type": { ... 5 子维度 ... }, "Video Content": [...] },
  "brightness_metrics":   { "score": <1-10>, "metrics": {...}, "reasons": [...], "error": "" },
  "motion_metrics":       { "score": <1-5>,  "decision": "...", ...},
  "saturation_metrics":   { "score": <1-5>,  "decision": "...", ...}
}
```

## 评分标准（与 `视频质量分.py` 对齐）

### Brightness 1-10

| 分 | 语义 | 决策 |
|----|------|------|
| 1  | 纯黑 / 极暗 | drop |
| 2  | 很暗（低调风格） | drop |
| 3  | 偏暗 | borderline |
| 4-7 | 略偏暗 / 正常偏暗 / 正常偏亮 / 明亮 | keep |
| 8  | 高调 | borderline |
| 9-10 | 接近过曝 / 严重过曝 | drop |

### Motion 1-5

| 分 | 语义 | 决策 |
|----|------|------|
| 1 | 完全静态 (Frozen) | drop |
| 2 | 切镜型 (PPT/Cut-like) | drop |
| 3 | 弱运动 (UI/字幕/抖动) | drop |
| 4 | 正常运动 (Normal) | keep |
| 5 | 强运动 (Strong) | keep |

### Saturation 1-5

| 分 | 语义 | 决策 |
|----|------|------|
| 1 | 接近灰度 | drop |
| 2 | 偏灰寡淡 | borderline |
| 3 | 正常自然 | keep |
| 4 | 鲜艳自然（最优） | keep |
| 5 | 过饱和滤镜 | borderline |

## 使用步骤

1. 打开 [GitHub Pages 链接](#部署)
2. 点击右上角 **📁 上传 CSV** 按钮，选择质量分打标结果 CSV
3. 等待解析完成（百万行级 CSV 通常 < 30s）
4. 调整：
   - 顶部 **日期** 下拉（按 `project_name` 后缀的 `yyyymmddhh`）
   - **采样方式**（随机 / channel 分层）
   - **最大显示数**（50 / 100 / 200 / 500）
   - 中部分类筛选 + 质量分阈值滑块
   - 底部 **作者** 下拉（基于已过滤数据）
5. 视频卡片实时刷新，每张卡片显示三档 score + meta JSON + 文件名
6. 点击 **⬇ 导出当前结果** 把当前阈值下全部匹配样本导出为 CSV

## 部署

### 本地预览
```bash
cd /path/to/repo
python3 -m http.server 8000
# 浏览器打开 http://localhost:8000
```

### GitHub Pages
1. Push 到 GitHub 仓库
2. Settings → Pages → Source: `main` 分支根目录
3. 几分钟后访问 `https://jiangenhua.github.io/<repo-name>/`

## 技术栈

- **HTML5 + Vanilla JS**（无构建步骤）
- [Tailwind CSS 2.2.19](https://tailwindcss.com/)（CDN）
- [PapaParse 5.4.1](https://www.papaparse.com/)（CSV 解析，支持 chunk streaming）
- [ECharts 5.5.0](https://echarts.apache.org/)（档位分布柱状图）

## 与上游算子链路的关系

```
原始视频 (iceberg/cos)
    │
    ├─→ 世界模型分类算子_gemma4_抽帧.py （GPU/Gemma-4 26B）
    │    ↳ 写出 result.classfication_result + result.requirement_info
    │
    └─→ 视频质量分.py（CPU 单 stage）
         ↳ 写出 result.brightness_metrics + motion_metrics + saturation_metrics
              │
              ▼
         ===== 本仪表盘消费 =====
              │
              ▼
         上传 CSV → 浏览器解析 → 多维度阈值筛选 → 视频抽样可视化 → 导出 keep 集合
```

## 浏览器兼容性

- Chrome / Edge / Safari (推荐 Chrome)
- 视频元素需要支持 `<video controls>`（所有现代浏览器都支持）
- 大文件 CSV 解析依赖 `FileReader` API（>200MB 文件建议关闭其它内存密集 Tab）

## License

MIT
