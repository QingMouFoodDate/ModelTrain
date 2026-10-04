# ModelTrain · 轻眸识刻算法研发与评测中心

> 面向冻结数据只读适配、模型训练、评测、消融与候选模型包导出的离线执行仓库

ModelTrain 提供训练、评测、消融和导出的执行入口，由 TrainPlatform 调度或在可信环境中以 CLI 单机运行。输入是受控冻结数据与配置，输出是可追溯实验/评测记录和候选模型包，交 Train 登记治理。候选方法不代表已验证结论。

## 当前状态与 M0 边界

现有 8 仓（`.github`、`ProjectPRD`、`TrainPlatform`、`InferPlatform`、`ModelTrain`、`WebClient`、`MobileClient`、`WxClient`）保持不新增、不合并、不重命名，统一处于**设计阶段，M0 待实现**。本 README 不能证明已有训练实现、可复现实验结果、真实候选包或发布模型；能力以代码、实验记录、评测报告、产物和测试为准。

M0 可从 CLI 单机训练、评测与导出入口起步，候选包手工审核并按 Train 职责登记；不以前置完成 Train 管理平台、容器/Kubernetes、全部 H1-H8、多端或端侧推理扩大范围。手工流程也不能省略冻结数据、防泄漏、评测、血缘与发布门禁。M0 是 C0 单图切片，不等于完整 C0；多图与完整纠偏审核回流为后续 C0，MobileClient/WxClient 暂停实现但保留原仓。

## 一、职责与核心模块

| 模块 | 设计职责 |
| :--- | :--- |
| 数据适配与校验 | 只读消费冻结数据、标注和 split，核验版本/哈希/许可与防泄漏信息 |
| 检测与局部识别 | 研发日期区域定位、局部 OCR 及模型预后处理算法 |
| 训练入口 | 消费训练配置、代码与环境版本，执行训练并记录状态/产物 |
| 评测与消融入口 | 执行统一评测与中性对照，产生可复现记录供 Train 登记 |
| 导出入口 | 生成符合上位 MODEL 的候选包、文件校验信息与 smoke 证据 |

不实现客户端 API、RecognitionSession、在线 Evidence/Decision、日期业务规则或生产推理服务；不承担 Train 的资产登记、标注审核、版本冻结、任务调度和发布治理。**本仓不执行 register / validate / release，也不 activate / rollback 线上模型**；本地导出校验不等于发布审核通过。

ModelTrain 产生的模型及模型侧处理在在线部署时仅支撑 Observation。InferPlatform 在线业务层负责将 Observation 聚合为 Evidence、按日期规则生成 Decision；其内部 runtime 执行模型前向，不维护会话或业务结论。Runtime 可以同进程或独立部署，不新增仓库。

## 二、执行入口、输入输出与异常

执行契约统一引用 [TRAINING](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/TRAINING.md)，具体 CLI 参数/机器 DTO 不在 README 自定义；以下是待实现的职责约束，不是可直接运行的命令。

| 入口 | 输入 | 输出 |
| :--- | :--- | :--- |
| 训练 | 冻结 DatasetVersion/标注/split、配置、代码/环境版本，调度时含 TrainJob 与 attempt | 执行状态、日志摘要、配置哈希、checkpoint 与训练记录 |
| 评测 | 固定评测版本、模型引用、指标/分层配置与环境 | EvalRecord 所需记录及可复核指标，交 Train 登记 |
| 消融 | RESEARCH 中的实验引用、固定切分、基线与对照配置 | 带实验关联、随机种子、配置和适用范围的结果记录 |
| 导出 | 通过离线评测的模型、字典/标签、预后处理与兼容性配置 | 候选 ModelBundle、manifest、evaluation 与 smoke 证据，交 Train 登记 |

处理流程为：冻结输入校验 -> 训练/评测/消融 -> 导出及本地交付检查 -> 交 Train 登记与审核。算法入口不反向依赖平台内部 API；状态与产物可由调度器收集或手工提交，不要求实现平台才能执行实验。

- 输入未冻结、哈希/标注引用不符、来源许可不足或 group 跨 split 时停止执行，不自行补标或重新切分。
- OOM、磁盘不足、异常退出、取消或导出失败均产生明确失败记录，不返回成功状态或可发布包；不完整输出隔离，不能覆盖已完成产物。
- 调度时关联 `(train_job_id, attempt)`；同 attempt 重复执行/提交不得覆盖或产生歧义，重复登记由 Train 按该键幂等处理。重试、worker 终止/隔离与 watchdog 属 Train 调度职责，本仓不擅自并行重启旧任务。
- checkpoint 续训须匹配冻结版本、配置、代码/环境约束与 ParentBundle；不相容时明确拒绝，不能静默换输入。
- 原始图片、缓存和输出由可信环境配置，不向跨仓记录或客户端暴露本地路径，不提交敏感原图、权重或秘密。

## 三、冻结数据与评测规范

训练数据由 Train 管理。本仓**只读消费冻结 DatasetVersion 的数据、标注修订和 split**，不修改样本、标注、版本或划分。发现问题输出校验记录交 Train，新建并冻结版本后再执行；不能在适配器内悄悄修正历史输入。

- 训练、验证和测试集合隔离，group 为单位切分；同一商品、包装批次、拍摄会话、连拍和近重复簇不得跨集合。测试集不参与调参，量化校准也不得挪用测试集。
- 保留 source/许可校验信息、数据/清单哈希与 ParentDataset lineage；数据加载、裁剪或增强是运行派生，不回写冻结资产或标注。
- 在线 Correction 不等于训练标签；Infer 只归集待审核记录，Train 训练质量审核通过才晋为候选并显式纳入新版本。本仓不从线上记录直接训练，不实现自动训练。
- 按包装材质、打码类型、角度、光照和清晰度分层报告。检测关注 mAP/Precision/Recall、小目标召回、漏检与误检；OCR 关注 CER、字段完全匹配、非法字符、误纠正与拒识。
- 性能记录注明硬件、运行环境、输入规格、预热与计时范围；指标关联评测数据、模型、配置和代码版本，不写无证据的性能承诺。
- 业务相关指标仅用于离线系统联评，由 Infer 的 Evidence/Decision 与统一日期规则产生；不能把这些统计写成 ModelTrain 直接输出在线业务决策。

实验记录应包含输入版本、配置哈希、代码提交、随机种子、环境/依赖、评测设置及产物校验和，使 Train 可登记 EvalRecord 和 ModelBundle 并复核血缘。README 示例或候选目标不作为实测结果。

## 四、研究唯一源

H1-H8 的编号、问题描述、基线与实验矩阵只维护于 [ProjectPRD/RESEARCH.md](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/RESEARCH.md)。本仓仅实现执行与记录，不复制研究假设表或另一套编号；P2、局部增强、受限字典、纠错、多分支与 FP16/INT8 均属待评测候选，不是已验证收益，也不是 M0 必须全部完成的清单。

任何提升/降低结论必须同时报告基线、适用数据范围、分层指标与副作用，不能只凭单一数字。实验引用与记录交 Train 登记，不用 README 替代研究证据。

## 五、候选包导出、血缘与发布

候选包须符合 [ProjectPRD/contracts/MODEL.md 的统一必需文件清单](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md)。不另画第二棵交付目录树。完整候选交付要求包含：`qingmou_det.onnx`、`qingmou_rec.onnx`、`dict.txt`、`preprocess.yaml`、`postprocess.yaml`、`labels.json`、`metadata.yaml`、`manifest.json`、`evaluation.json`、`smoke/input.json`、`smoke/expected.json`。这仅是**候选交付要求，不表示当前已有真实包**；不得退回只交付两份 ONNX、字典与元数据的精简包。

导出前的本地检查至少包括：

- 模型可加载，输入输出张量、通道/归一化与预后处理符合上位规格；不在 README 再定义尺寸、阈值或算子契约。
- 字典、标签、识别输出与解码逻辑一致；检测、识别及配置必须同包同版。
- manifest 覆盖完整文件及 SHA-256；metadata/evaluation 关联数据、配置、代码、环境和实测结果，不用示例指标充当评测证据。
- smoke 输入与预期能复核模型加载和推理；FP16/INT8 等可选候选有 FP32 对照，无结果时不宣称轻量化收益。
- 保留 ParentDataset 与 ParentBundle lineage：使用的冻结数据版本、父数据集、续训/微调父模型、配置哈希、代码提交、环境与评测引用均可追溯，具体字段按 DATA/TRAINING/MODEL。

发布职责链严格为：

```text
ModelTrain 导出候选包与实验/评测记录
  -> TrainPlatform register -> validate -> release
  -> InferPlatform activate / rollback
  -> InferPlatform 内部 Runtime 校验、加载与执行
```

本仓本地检查通过不代表 `validated` 或 `released` 已获治理授权；Train 登记评测与模型血缘后执行发布门禁。发布状态不等于线上 active，Infer 切换失败保持原 active；授权且符合兼容性/安全要求的 N-1 旧版可回滚。本仓不直接部署到生产、切换 active 或自动替换线上模型。

## 六、提议工程结构与候选技术

以下为**提议目录，不代表已有文件、训练代码或模型产物**：

```text
ModelTrain/                     # 离线算法执行与候选交付仓库
├── README.md                   # 边界、输入输出与交付约束
├── requirements.txt            # 候选研发依赖清单
├── configs/                    # 训练、评测与导出配置
├── datasets/                   # 冻结数据的只读消费模块
│   ├── adapters/               # 数据读取与标注格式适配
│   ├── manifests/              # 冻结清单引用与只读校验配置
│   └── validators/             # 哈希、标注、许可与防泄漏检查
├── models/                     # 候选视觉算法实现
│   ├── detectors/              # 日期区域检测网络
│   ├── recognizers/            # 局部字符与点阵 OCR 网络
│   └── postprocess/            # 模型解码与规整，不含日期业务决策
├── experiments/                # 实验配置与可审计元数据
├── evaluation/                 # 统一评测、分层统计与记录生成
├── tools/                      # 单机 CLI 训练、评测与消融入口
└── deploy/                     # 候选包导出、转换、量化与 smoke 检查
```

候选路线为 PyTorch + ONNX Runtime + OpenCV，或 PaddlePaddle + PaddleOCR/PaddleDetection；最终以可复现实验、环境兼容性和统一 ONNX 交付契约为准。这里不承诺两套实现、容器化或端侧部署，不以选型替代评测。

## 七、上位文档与访问权限

- [需求](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/REQUIREMENTS.md)、[架构](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/ARCHITECTURE.md)、[推进路线](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/ROADMAP.md)
- [API](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/API.md)、[DATA](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/DATA.md)、[DATE_RULES](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/DATE_RULES.md)、[MODEL](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md)、[TRAINING](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/TRAINING.md)
- [研究唯一源](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/RESEARCH.md)、[数据质量](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/DATA_QUALITY.md)、[工程规范](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/ENGINEERING.md)
- [ADR-001 八仓职责与在线离线边界](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/decisions/ADR-001-REPOSITORY_BOUNDARIES.md)

ADR-001 已记录批准的职责设计；具体执行字段、完整 MODEL 清单与职责链仍须上位契约同步评审。DATE_RULES、TRAINING 的规格由上位文档统一维护；候选引用不代表已有训练产物或生产契约冻结。日期规则和 DTO 只引用上位，不在本仓重复自定义。

当前仓库为私有仓库；访问 README、上位文档及 [组织主页](https://github.com/QingMouFoodDate) 的内容取决于登录、组织/仓库授权和主页可见性设置。GitHub 绝对链接不承诺公共可读；敏感数据与权重仅在受控环境中交付，不以公开复制解决访问权限。
