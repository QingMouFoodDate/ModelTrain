# ModelTrain · 轻眸识刻算法计算核心

> **组织归属**：`QingMouFoodDate` GitHub 组织  
> **仓库性质**：**大创核心算法中枢**，专注于食品包装日期**小目标微距检测、点阵字符识别、喷码双向纠错矩阵与 GB 7718 语义决策树**的研发与轻量化导出  

---

## 一、 核心定位与技术突破主线

立足大创立项申报书“找得到、读得准、判得清”的核心研究路线，本仓库承担算法全生命周期的研发攻坚，直击工业与生活实拍中“小尺度（<2%）、反光褶皱、点阵断笔、日期混淆”四大痛点：

### 1. 核心技术突破主线 (申报书与立项答辩重点)
- **突破一 · 小目标微距检测定位 (找得到)**：
  - 针对微小打码区域，结合浅层高分辨率特征（P2 检测头）与自适应 Anchor 机制，配合 CLAHE 去反光增强算子，在复杂图文背景中精准定位日期打码框；
- **突破二 · 微型工业字符序列识别 (读得准)**：
  - 针对工业点阵喷墨、无墨凹凸钢印、激光刻印，采用轻量序列识别网络进行字符转录；
- **突破三 · 喷码双向纠错矩阵 (核心创新算法)**：
  - 针对点阵断针与粘连混淆（如 `O/D ↔ 0`, `I/l/T ↔ 1`, `S/3 ↔ 5`, `B/8 ↔ 8`），基于字符拓扑空洞与汉明距离构建加权 Levenshtein 动态规划对齐，大幅提升低置信度字符的纠错率；
- **突破四 · GB 7718 日期语义决策树 (判得清)**：
  - 严格遵循《预包装食品标签通则》(GB 7718)，支持“单一到期日毫秒级比对时钟”与“生产日期 + 直读保质期时长动态演算”双模式，实现结构化输出与临期安全状态推演；
- **突破五 · 模型轻量化与跨平台导出**：
  - 将 PyTorch 模型无损导出为标准 ONNX 格式，供后端 Web 演示前台与端侧环境加载。

### 2. 职责解耦与开发建议 (Design Principles & Scope)
- **环境优先保障原生迭代**：推荐优先在本地或实验室 GPU 原生 Python 环境中高效调试与实验，降低前期环境配置门槛；如后续有跨平台复现或环境隔离需求，可配合提供可选的 Dockerfile 辅助脚本；
- **专注模型研发与核心指标**：本仓库聚焦于数据集管道、网络优化、消融实验（支撑结题报告、软著与论文）及轻量化 ONNX 资产交付；完整的业务持久化与在线交互界面由 TrainPlatform 与 WebClient 协同承接。

---

## 二、 推荐算法实验架构 (参考科研与工业标准)

为保证科研实验可复现（消融实验、论文撰写与答辩展示）及工程交付标准，**推荐采用超参配置与网络结构解耦的标准结构**：

```text
ModelTrain/ (推荐目录结构参考)
├── README.md                      # 本说明文档
├── requirements.txt               # 算法核心依赖 (PyTorch, Torchvision, Albumentations, ONNX, PyYAML)
├── .gitignore                     # 忽略权重文件 (*.pth, runs/, __pycache__/)
├── configs/                       # 【实验配置中心】网络结构与训练超参数通过 YAML 集中管理，便于实验复现
│   ├── det_date_p2.yaml           # 小目标检测模型超参 (引入 P2 微小目标特征层)
│   └── rec_crnn_tiny.yaml         # 局部微型字符 OCR 识别网络超参
├── datasets/                      # 【数据集封装】标准 PyTorch Dataset 与自适应数据增强
│   ├── date_dataset.py            # 读取同级暂存库 ../data/raw/manifest.jsonl
│   └── transforms.py              # CLAHE 去反光增强、微距尺度缩放与仿射形变
├── models/                        # 【神经网络定义】
│   ├── detectors/                 # 检测网络结构 (Backbone, FPN Neck, P2 微小目标检测头)
│   ├── recognizers/               # 识别网络结构 (CRNN / SVTR / CTC Loss)
│   ├── postprocess/               # 【后处理与决策核心】
│   │   ├── corrector.py           # 喷码双向纠错矩阵 (加权 Levenshtein DP)
│   │   └── decision_tree.py       # GB 7718 日期语义决策树 (生产日期+保质期推演)
├── tools/                         # 【标准入口脚本】算法日常工作入口
│   ├── train.py                   # 启动训练：python tools/train.py --config configs/det_date_p2.yaml
│   ├── eval.py                    # 启动评测：在验证集上计算 mAP50 与 OCR 字符准度
│   └── export_onnx.py             # 导出交付：将 PyTorch 模型转为合规 ONNX 模型
└── deploy/                        # 【交付前精度验证】
    └── verify_onnx.py             # 检验 ONNX 与 PyTorch 原模型推断结果的一致性 (误差 < 1e-4)
```

---

## 三、 共享数据集挂载与质检门禁

为避免高容量图片污染代码 Git 历史，数据集物理存放在**同级独立暂存库**：
- **数据源路径**：`../data/raw/`（当前包含经过单张原图无损审阅的 **197 张纯正黄金基准样本** 及 `manifest.jsonl`）；
- **数据质量宪法**：全员遵循 [`ProjectPRD/standards/数据采集审核与质量控制规范.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/数据采集审核与质量控制规范.md)（4 项生活环境容差与 6 大一票否决底线）；
- **门禁验证**：训练前可运行暂存库验证脚本：
  ```bash
  python ../data/scripts/verify_dataset_quality.py
  ```

---

## 四、 算法交付规约 (唯一跨仓契约)

算法模型最终交付给 `TrainPlatform` 时，**必须且只能严格遵守** [`ProjectPRD/contracts/model_contracts.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/model_contracts.md)：

1. **检测模型规格**：输入尺寸固定为 `[1, 3, 640, 640]`，输出固定格式的 Anchor/BBox 阵列；
2. **OCR 识别规格**：输出标准字符概率分布，且**必须附带标准的 `dict.txt` 字符字典表**；
3. **交付产物**：打包为包含 `qingmou_det.onnx`、`qingmou_rec.onnx`、`dict.txt` 与 `metadata.yaml`（记录 mAP 和耗时）的 Release Bundle。

---

## 五、 算法探索空间与技术自主权

**算法框架与网络架构不设限制，鼓励团队大胆探索与消融对比**：
- 目标检测可在 **YOLOv8 / YOLOv11 / YOLOX / RT-DETR** 间自由对比选型，重点对比 P2 浅层微小目标特征层的增益；
- OCR 识别可在 **CRNN / SVTR-Tiny** 等轻量架构中探索；
- 丰富的实验对比与消融分析将在大创结题答辩、软件著作权申报和论文发表中提供核心支撑材料。

---

## 六、 阶段完成定义 (Definition of Done, DoD)

- [ ] **阶段 1 (数据跑通)**：`datasets/date_dataset.py` 能够成功读取 `../data/raw/` 并解析 197 张样本与标注；
- [ ] **阶段 2 (Baseline 建立)**：完成第一版小目标检测模型训练，验证集 mAP50 达到基线水平；
- [ ] **阶段 3 (ONNX 交付)**：通过 `tools/export_onnx.py` 导出 ONNX 模型，并由 `verify_onnx.py` 验证精度对齐。
