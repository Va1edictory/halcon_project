# 基于深度学习的晶体管缺陷检测系统 — 答辩讲解文档

---

## 一、项目背景与选题意义

### 1.1 工业背景

在电子制造领域，晶体管是电路板的核心元件。引脚弯曲（bent_lead）、引脚切割不良（cut_lead）、外壳破损（damaged_case）、元件错位（misplaced）等外观缺陷直接影响产品良率和可靠性。传统人工目检效率低、主观性强，难以满足大批量生产的一致性和速度要求。

### 1.2 选题意义

本项目利用机器视觉与深度学习技术，构建自动化晶体管缺陷检测系统。核心目标是：

- **替代人工**：实现 7×24 无人值守的自动质检
- **提升一致性**：基于深度学习模型的标准判定，消除人工主观差异
- **缺陷定位**：不仅判断 OK/NG，还能标注缺陷位置，辅助工艺溯源

### 1.3 为什么选择 HALCON

HALCON（MVTec）是工业机器视觉领域事实上的标准平台，其 DLT（Deep Learning Tool）提供训练好的异常检测模型，无需从零搭建深度学习框架。优势包括：

- 工业级稳定性与丰富的图像处理算子库
- 支持 CPU/GPU 推理，部署灵活
- HDevelop 交互式开发环境，快速原型验证

---

## 二、数据集与模型

### 2.1 数据集

使用 MVTec Anomaly Detection Dataset 的晶体管（Transistor）子集：

| 类别 | 训练集 | 测试集 | Ground Truth |
|------|--------|--------|--------------|
| good（正常） | 60 张 | 20 张 | — |
| bent_lead（引脚弯曲） | — | 20 张 | 像素级标注 |
| cut_lead（引脚切割不良） | — | 20 张 | 像素级标注 |
| damaged_case（外壳破损） | — | 20 张 | 像素级标注 |
| misplaced（元件错位） | — | 20 张 | 像素级标注 |

### 2.2 深度学习模型

- **模型类型**：GC Anomaly Detection（全局上下文异常检测）
- **DLT 训练平台**：HALCON Deep Learning Tool
- **模型输入**：256×256 像素，3 通道，值域 [-127, 128]
- **模型输出**：
  - `anomaly_score`（异常分数）：全局异常程度标量值
  - `anomaly_image`（异常热力图）：256×256 像素级异常概率分布
- **预处理参数**：`full_domain`（全图推理）、`normalization_type: none`

---

## 三、系统架构设计

### 3.1 总体架构

```
┌─────────────────────────────────────────────────────┐
│                    main 入口脚本                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │  配置区：模型路径 / 阈值 / 检测模式               │ │
│  └─────────────────────────────────────────────────┘ │
│                        │                             │
│         ┌──────────────┴──────────────┐              │
│         ▼                             ▼              │
│  ┌─────────────┐              ┌─────────────┐        │
│  │ Mode 1      │              │ Mode 2      │        │
│  │ 单张检测    │              │ 批量检测    │        │
│  │ 交互式调试  │              │ 离线处理    │        │
│  └──────┬──────┘              └──────┬──────┘        │
│         │                            │               │
│         └──────────┬─────────────────┘               │
│                    ▼                                 │
│       ┌────────────────────────┐                     │
│       │    检测流水线           │                     │
│       │  加载模型 → 预处理 →    │                     │
│       │  推理 → 定位 → 可视化   │                     │
│       └────────────────────────┘                     │
└─────────────────────────────────────────────────────┘
```

### 3.2 数据流

```
原始图像 (900×900, 3ch)
    │
    ▼
[预处理] 通道检测 → 256×256缩放 → real转换 → [-127,128]映射 → full_domain → DLSample
    │
    ▼
[推理] apply_dl_model → anomaly_image(256×256) + anomaly_score
    │
    ├──────────────────────────────────┐
    ▼                                  ▼
[定位] scale_image_max                [判定] score > threshold ?
    → threshold                          ├─ NG (IsNG=1)
    → connection                         └─ OK (IsNG=0)
    → select_shape (过滤噪声)
    → shape_trans (矩形框)
    → 仿射缩放 (256 → 原图尺寸)
    │
    ▼
[可视化] 左窗: 原图 | 右窗: 原图+红框+判定文字+分数
    │
    ▼
[输出] 批量模式下保存到 output/OK_xxxx.png 或 output/NG_xxxx.png
```

---

## 四、核心技术实现

### 4.1 模型加载

```halcon
read_dl_model (ModelPath, DLModelHandle)
read_dict (DictPath, [], [], DLPreprocessParams)
get_dict_tuple (DLPreprocessParams, 'model_type', ModelType)    → 'gc_anomaly_detection'
get_dict_tuple (DLPreprocessParams, 'image_width', ImageWidth)  → 256
get_dict_tuple (DLPreprocessParams, 'image_height', ImageHeight)→ 256
```

### 4.2 图像预处理

模型训练时的输入规格与原始图像存在显著差异，需要手动五步预处理：

| 步骤 | 算子 | 输入 | 输出 | 目的 |
|------|------|------|------|------|
| ① 通道转换 | `count_channels` + `compose3` | 1ch 灰度 | 3ch 图像 | 匹配模型 3 通道要求 |
| ② 尺寸缩放 | `zoom_image_size` | 900×900 | 256×256 | 匹配模型输入尺寸 |
| ③ 类型转换 | `convert_image_type` | byte | real | 匹配模型数据类型 |
| ④ 值域映射 | `scale_image(x, 1.0, -127)` | [0,255] | [-127,128] | 匹配模型值域 |
| ⑤ 域的设置 | `full_domain` | 带 domain | 全图 | 消除 domain 裁剪影响 |
| ⑥ 样本生成 | `gen_dl_samples_from_images` | 图像 | DLSample | 创建推理输入 |

### 4.3 异常检测推理

```halcon
* 执行推理
apply_dl_model (DLModelHandle, DLSample, [], DLResult)

* 提取结果（注意：图像用 get_dict_object 避免 iconic/control 类型冲突）
get_dict_object (AnomalyImage, DLResult, 'anomaly_image')
get_dict_tuple  (DLResult, 'anomaly_score', AnomalyScore)

* 判定
if (AnomalyScore > AnomalyThreshold)
    IsNG := 1
else
    IsNG := 0
endif
```

### 4.4 缺陷定位

从异常热力图提取缺陷区域坐标：

```halcon
* 1. 拉伸值域到 [0,255]
scale_image_max (AnomalyImage, AnomalyImageScaled)

* 2. 二值化（高于阈值的像素 = 异常区域）
threshold (AnomalyImageScaled, AnomalyRegion, RegionThreshold, 255)

* 3. 分离不连通的独立缺陷
connection (AnomalyRegion, ConnectedRegions)

* 4. 过滤噪点（面积 < MinArea 的散点排除）
select_shape (ConnectedRegions, DefectRegions, 'area', 'and', MinArea, 999999)

* 5. 统计缺陷数量
count_obj (DefectRegions, NumDefects)

* 6. 生成外接矩形
if (NumDefects > 0)
    shape_trans (DefectRegions, BoundingBoxes, 'rectangle1')
endif
```

### 4.5 坐标缩放

异常热力图是 256×256，但原图是 900×900。使用仿射变换将矩形框映射到原图坐标系：

```halcon
hom_mat2d_identity (HomMat2D)
hom_mat2d_scale (HomMat2D, ImgWidth / 256.0, ImgHeight / 256.0, 0, 0, HomMat2D)
affine_trans_region (BoundingBoxes, BoundingBoxesScaled, HomMat2D, 'nearest_neighbor')
```

### 4.6 结果可视化

- **左窗**：原始晶体管图像 + "Original Image" 标签
- **右窗**：原始图像 + 红色矩形框标注缺陷位置 + 居中大号 OK/NG 判定 + 底部分数和缺陷数量

---

## 五、关键技术难点与解决方案

| 难点 | 问题描述 | 解决方案 |
|------|----------|----------|
| **HDevelop 26 XML 格式适配** | HALCON 26 使用 `<l>`/`<c>` 元素而非 CDATA 存储代码，普通文本无法直接打开 | 分析 HDevelop 原生导出格式，编写 XML 转换脚本 |
| **Iconic/Control 类型冲突** | `set_dict_tuple` 无法传递图像（要求 control 类型，图像是 iconic） | 改用 `gen_dl_samples_from_images` 创建 DLSample |
| **异常图像提取失败** | `get_dict_tuple` 返回 control 值，图像操作需要 iconic | 改用 `get_dict_object` 提取 `anomaly_image` |
| **preprocess_dl_model 不存在** | HALCON 26 中该算子已被移除 | 手动实现 5 步预处理流水线 |
| **坐标对齐偏差** | 异常图 256×256 vs 原图 900×900，矩形框位置不准 | 使用 `hom_mat2d_scale` + `affine_trans_region` 仿射变换 |
| **disp_message 文本不可见** | `'window'` 坐标系使用窗口像素坐标，与图像像素坐标不符 | 改用 `'image'` 坐标系 |

---

## 六、系统验证

### 6.1 单张检测测试

- 测试图像：bent_lead/000.png
- 左侧显示原始图像，右侧清晰标注 OK（绿色）/ NG（红色）结果
- 缺陷区域以红色矩形框准确定位
- 底部显示异常分数和缺陷区域数量

### 6.2 批量检测测试

- 批量处理 test/good/ 目录下 20 张正常图像
- 输出结果保存至 output/ 目录
- 文件名按 OK/NG 分类：`OK_0001.png`、`NG_0003.png`
- 支持异常容错：单张图片失败不中断整体批处理

---

## 七、总结与展望

### 7.1 项目总结

本项目基于 HALCON DLT 深度学习平台，完整实现了晶体管外观缺陷检测系统，涵盖：

- 模型加载与手动图像预处理
- 异常检测推理与 OK/NG 判定
- 缺陷区域提取与坐标映射
- 双窗口交互式可视化
- 单张与批量两种工作模式

### 7.2 后续改进方向

1. **缺陷分类**：将异常检测模型替换为目标检测或分类模型，支持具体缺陷类型识别（弯脚/切脚/破壳/错位）
2. **阈值自适应**：基于验证集的统计分布自动计算最优判定阈值
3. **实时检测**：集成工业相机 SDK，实现产线实时在线检测
4. **模型评估**：输出 precision/recall/F1-score 等量化指标
5. **界面升级**：使用 C#/C++ 开发独立 GUI 应用程序，脱离 HDevelop IDE 运行
