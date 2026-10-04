# ModelTrain · 轻眸识刻算法研发与评测中心

> 面向食品包装日期区域定位、受限字符识别、实验评测和模型交付的离线研发仓库

ModelTrain 负责数据适配、模型训练、消融实验、统一评测和标准模型包导出。文档中的模型结构和实验方法均为研发候选或交付约束，具体采用方案以可复现实验结果为准。它不负责客户端 API、在线识别会话、日期语义决策或生产环境的模型调度。

## 当前状态

当前仓库主要维护离线训练、评测和模型交付设计，尚无可由本仓库 README
单独证明的可复现训练结果或发布模型。实际能力以训练代码、实验记录、
评测报告和模型产物为准。

---

## 一、研发定位与职责边界

### 1. 负责内容

- 日期区域和包装文字的视觉检测与识别算法；
- 点阵、激光和钢印等小目标字符的局部识别；
- 数据读取、标注格式适配和质量检查；
- 训练配置、实验记录和消融评测；
- FP32、FP16 或 INT8 等模型导出方案评估；
- 符合 `MODEL.md` 的模型交付包生成。

### 2. 不负责内容

- 不直接实现 Web、移动端或小程序 API；
- 不维护 `RecognitionSession` 的在线状态；
- 不在模型中实现最终日期期限决策；
- 不直接向客户端提供生产推理接口；
- 不将人工纠偏样本未经审核自动加入训练集；
- 不在公开代码仓库中提交原始图片或其他敏感数据。

模型交付后，由 Inference Runtime 负责加载、校验和调用；TrainPlatform 负责管理模型版本引用、编排推理请求，并完成日期语义推导与标示状态决策。模型本身只输出约定的机器观察值。

---

## 二、研究主线与处理流程

```mermaid
flowchart TD
    A["受控数据清单与标注"] --> B["数据质量检查"]
    B --> C["候选检测与识别模型"]
    C --> D["局部裁剪、校正与增强"]
    D --> E["受限字符解码与后处理"]
    E --> F["统一评测与消融实验"]
    F --> G["模型导出与模型卡片"]
    G --> H["标准模型交付包"]
    H --> I["Inference Runtime 加载与推理"]
    G -.版本与评测信息.-> J["TrainPlatform 模型元数据"]
```

研究重点包括：

### 1. 日期区域定位

研究适合极小目标和复杂背景的检测结构，关注日期区域面积小、曲面反光、点阵断笔和背景文字干扰等条件下的召回率与误检率。P2 高分辨率检测头等方法属于实验候选，最终以统一评测结果为准。

### 2. 局部图像处理

根据检测框生成局部输入，并评估旋转校正、透视变换、局部对比度增强和形态学处理对不同包装材质的影响。预处理不应掩盖原始图像，也不应替代质量不足时的人工复核。

### 3. 受限字符识别

针对日期、保质期和常见包装标示研究轻量 OCR。候选方法包括 CRNN、SVTR-Tiny 或其他适配目标数据分布的识别网络；受限字典、CTC 解码和规则后处理需要通过实验验证其收益与误纠风险。

### 4. 多分支信息支持

可以分别提取局部打码信息和全局包装说明，再将模型观察值交由 TrainPlatform 聚合为结构化证据。ModelTrain 输出观察值，不直接输出最终食品期限结论。

---

## 三、数据与评测规范

### 1. 数据划分

- 训练集、验证集和测试集相互隔离；
- 同一商品、包装批次或高度相似的连续拍摄样本不应跨集合泄漏；
- 测试集不参与超参数调整；
- 按包装材质、打码类型、拍摄角度、光照和清晰度进行分层统计；
- 数据清单和标注检查应可重复执行，原始图片由受控环境提供。

### 2. 检测指标

- mAP@0.5、Precision、Recall；
- 日期区域小目标召回率；
- 漏检率、误检率和不同材质分层结果；
- 单图推理耗时、参数量和模型大小。

### 3. OCR 指标

- 字符错误率（CER）；
- 字段完全匹配率；
- 日期字段解析成功率；
- 非法字符率；
- 误纠正率和低质量输入下的拒识能力。

### 4. 系统关联指标

- 标示状态可推导率；
- 信息缺失和证据冲突的识别率；
- 多图输入下的决策一致性；
- 端侧轻量模型与云端模型的结果差异；
- 置信度不足时转入复核的准确性。

指标数值必须关联数据版本、评测配置和运行环境。README 中的示例数字仅用于说明字段结构，不代表当前项目实测结果。

---

## 四、实验假设与消融设计

H1-H8 的唯一编号、问题描述和实验矩阵统一维护于
[ProjectPRD/RESEARCH.md](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/RESEARCH.md)。本仓库只负责实现训练、
评测和交付，不复制研究假设表，也不把候选方案写成已验证结论。

任何“提升”或“降低”的结论都应同时报告适用数据范围和可能的副作用，不能仅依据单一指标判断算法方案。

---

## 五、提议工程结构

以下目录是实现建议，不代表当前仓库已经具备对应文件。

```text
ModelTrain/
├── README.md
├── requirements.txt
├── configs/                     # 模型、数据和实验配置
├── datasets/
│   ├── adapters/               # 数据读取和标注格式适配
│   ├── manifests/              # 数据清单与划分配置
│   └── validators/             # 标注和数据质量检查
├── models/
│   ├── detectors/              # 日期区域检测网络
│   ├── recognizers/            # 字符与点阵 OCR 网络
│   └── postprocess/            # 解码、规整和纠错算法
├── experiments/                # 实验配置、运行记录和结果摘要
├── evaluation/                 # 统一评测和指标计算
├── tools/                      # 批量实验和基准测试工具
└── deploy/                     # ONNX 导出、转换和量化脚本
```

工程目录保存代码、配置和可审计的实验元数据，不固定公开原始图片、模型权重缓存或部署环境路径。

---

## 六、模型导出与交付

每次可发布模型都必须生成符合 [`ProjectPRD/contracts/MODEL.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md) 的自包含交付包，至少包括：

```text
release_bundle/
├── qingmou_det.onnx
├── qingmou_rec.onnx
├── dict.txt
├── preprocess.yaml
├── postprocess.yaml
├── labels.json
├── metadata.yaml
├── manifest.json
├── evaluation.json
└── smoke/
    ├── input.json
    └── expected.json
```

交付前应完成：

- 输入尺寸、通道顺序、归一化和输出张量检查；
- ONNX Runtime 加载和推理 smoke test；
- 字典文件与解码逻辑一致性检查；
- 模型版本、评测配置和运行环境记录；
- 文件完整性和 SHA-256 校验；
- 与基准模型的精度、速度和模型大小对比。

`metadata.yaml` 中的指标必须注明评测数据版本和运行环境，示例字段只用于说明格式，不作为项目性能承诺。

ModelTrain 只交付标准化模型资产和评测信息；模型发布后由 Inference Runtime 负责加载、校验和在线推理，TrainPlatform 只管理模型版本引用并编排推理调用。

---

## 七、技术选型

候选路线为 PyTorch + ONNX Runtime + OpenCV，或
PaddlePaddle + PaddleOCR + PaddleDetection。前者便于研究和独立 Runtime
适配，后者便于复用中文 OCR 组件；最终以可复现实验结果和统一交付契约为准。

无论采用哪条研发路线，最终交付都必须满足统一的 ONNX 输入输出、字符字典、元数据、版本和完整性校验契约。

---

## 八、相关文档

- [系统需求](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/REQUIREMENTS.md)
- [系统架构](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/ARCHITECTURE.md)
- [API 接口契约](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/API.md)
- [数据质量规范](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/DATA_QUALITY.md)
- [模型交付规约](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md)
