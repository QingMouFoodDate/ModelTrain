# ModelTrain · 轻眸识刻算法计算核心

> **组织归属**：`QingMouFoodDate` GitHub 组织  
> **仓库性质**：专注于食品包装生产日期与保质期**小目标定位检测、微型字符 OCR 识别网络训练与轻量化导出**的算法计算核心工程  

---

## 🔬 一、 核心定位与职责边界

针对食品包装上日期常存在的**“小尺度、反光褶皱、点阵喷码断笔、背面营养成分表大量无关印刷文字干扰”**等工业痛点，本仓库承担算法模型从“数据读取 ➔ 模型训练 ➔ 指标评测 ➔ 轻量化交付”的全生命周期研发。

### 1. 本仓职责范围 (In Scope)
- **小目标日期检测模型研发**：在整张包装图中精准快速框选生产日期与到期日所在微小打码区域；
- **微型工业字符 OCR 识别**：针对点阵喷墨、无墨钢印、激光刻印进行高精度序列字符识别；
- **图像自适应增强流水线**：研发针对点阵断笔、金属曲面高光、透视形变的自适应预处理算子；
- **模型轻量化与跨平台导出**：将 PyTorch 模型无损导出为标准 ONNX 格式，供后端与端侧加载。

### 2. 避免过度设计与边界禁止 (Out of Scope)
- ❌ **严禁写死 Docker 容器强依赖**：算法同学可以在单机或实验室 GPU 电脑上直接使用 Python 原生环境开发调试；
- ❌ **严禁侵入 Web 或业务持久层**：不要写 Web 界面代码，不要直接连接业务数据库。算法仓库专注于指标（mAP50、字符精确度）与模型资产交付。

---

## 🏗️ 二、 推荐算法实验架构 (参考科研与工业标准)

为保证科研实验可复现（消融实验、论文撰写与答辩展示）及工程交付标准，**强烈推荐采用超参配置与网络结构解耦的标准结构**：

```text
ModelTrain/ (推荐目录结构参考)
├── README.md                      # 本说明文档
├── requirements.txt               # 算法核心依赖 (PyTorch, Torchvision, Albumentations, ONNX, PyYAML)
├── .gitignore                     # 忽略权重文件 (*.pth, runs/, __pycache__/)
├── configs/                       # 【实验配置中心】所有超参通过 YAML 驱动，严禁代码硬编码！
│   ├── det_date_nano.yaml         # 小目标检测模型超参 (learning_rate, epochs, input_size, batch_size)
│   └── rec_crnn_tiny.yaml         # 局部微型字符 OCR 识别网络超参
├── datasets/                      # 【数据集封装】标准 PyTorch Dataset 与自适应数据增强
│   ├── date_dataset.py            # 读取同级暂存库 ../data/raw/manifest.jsonl
│   └── transforms.py              # CLAHE 去反光增强、微距尺度缩放与仿射形变
├── models/                        # 【神经网络定义】
│   ├── detectors/                 # 检测网络结构 (Backbone, FPN Neck, P2 微小目标检测头)
│   └── recognizers/               # 识别网络结构 (CRNN / SVTR / CTC Loss)
├── tools/                         # 【标准入口脚本】算法同学的日常工作入口
│   ├── train.py                   # 启动训练：python tools/train.py --config configs/det_date_nano.yaml
│   ├── eval.py                    # 启动评测：在验证集上计算 mAP50 与 OCR 字符准度
│   └── export_onnx.py             # 导出交付：将 PyTorch 模型转为合规 ONNX 模型
└── deploy/                        # 【交付前精度验证】
    └── verify_onnx.py             # 检验 ONNX 与 PyTorch 原模型推断结果的一致性 (误差 < 1e-4)
```

---

## 📂 三、 共享数据集挂载与质检门禁

为避免高容量图片污染代码 Git 历史，数据集物理存放在**同级独立暂存库**：
- **数据源路径**：`../data/raw/`（当前包含经过单张原图无损审阅的 **197 张纯正黄金基准样本** 及 `manifest.jsonl`）；
- **数据质量宪法**：全员遵循 [`ProjectPRD/standards/数据采集审核与质量控制规范.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/数据采集审核与质量控制规范.md)（4 项生活环境容差与 6 大一票否决底线）；
- **门禁验证**：训练前可运行暂存库验证脚本：
  ```bash
  python ../data/scripts/verify_dataset_quality.py
  ```

---

## 🤝 四、 算法交付规约 (唯一跨仓契约)

算法模型最终交付给 `TrainPlatform` 时，**必须且只能严格遵守** [`ProjectPRD/contracts/model_contracts.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/model_contracts.md)：

1. **检测模型规格**：输入尺寸固定为 `[1, 3, 640, 640]`，输出固定格式的 Anchor/BBox 阵列；
2. **OCR 识别规格**：输出标准字符概率分布，且**必须附带标准的 `dict.txt` 字符字典表**；
3. **交付产物**：打包为包含 `qingmou_det.onnx`、`qingmou_rec.onnx`、`dict.txt` 与 `metadata.yaml`（记录 mAP 和耗时）的 Release Bundle。

---

## 🚀 五、 算法组探索空间 (技术自主权)

**算法框架与网络架构不设限制，鼓励算法同学大胆创新**：
- 目标检测可在 **YOLOv8 / YOLOv11 / YOLOX / RT-DETR** 间自由对比选型，鼓励探索 P2 浅层微小目标特征层；
- OCR 识别可在 **CRNN / SVTR-Tiny** 等轻量架构中探索；
- 丰富的实验对比与消融分析将在答辩和论文中成为极具含金量的加分项！

---

## 🏁 六、 阶段完成定义 (Definition of Done, DoD)

- [ ] **阶段 1 (数据跑通)**：`datasets/date_dataset.py` 能够成功读取 `../data/raw/` 并解析 197 张样本与标注；
- [ ] **阶段 2 (Baseline 建立)**：完成第一版小目标检测模型训练，验证集 mAP50 达到基线水平；
- [ ] **阶段 3 (ONNX 交付)**：通过 `tools/export_onnx.py` 导出 ONNX 模型，并由 `verify_onnx.py` 验证精度对齐。
