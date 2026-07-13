# Source 目录 HDevelop 文件分析

> 分析目录：`E:\halcon_project\source`  
> 分析日期：2026/07/09

---

## 目录

1. [文件总览与架构关系](#1-文件总览与架构关系)
2. [main_transistor_inspection.hdev 逐行分析](#2-main_transistor_inspectionhdev-逐行分析)
3. [tune_thresholds.hdev 逐行分析](#3-tune_thresholdshdev-逐行分析)
4. [load_model.hdvp 逐行分析](#4-load_modelhdvp-逐行分析)
5. [preprocess.hdvp 逐行分析](#5-preprocesshdvp-逐行分析)
6. [infer.hdvp 逐行分析](#6-inferhdvp-逐行分析)
7. [localize.hdvp 逐行分析](#7-localizehdvp-逐行分析)
8. [visualize.hdvp 逐行分析](#8-visualizehdvp-逐行分析)
9. [batch_process.hdvp 逐行分析](#9-batch_processhdvp-逐行分析)
10. [文件间调用关系图](#10-文件间调用关系图)
11. [关键差异对比](#11-关键差异对比)

---

## 1. 文件总览与架构关系

source 目录包含 8 个文件，分为两类：

| 文件类型 | 文件 | 用途 |
|----------|------|------|
| **主脚本 (.hdev)** | `main_transistor_inspection.hdev` | 主入口，包含完整检测流程（单张+批量） |
| **主脚本 (.hdev)** | `tune_thresholds.hdev` | 阈值调优脚本，辅助确定最佳参数 |
| **外部过程 (.hdvp)** | `load_model.hdvp` | 模型加载过程 |
| **外部过程 (.hdvp)** | `preprocess.hdvp` | 图像预处理过程（使用 `preprocess_dl_model`） |
| **外部过程 (.hdvp)** | `infer.hdvp` | 推理过程 |
| **外部过程 (.hdvp)** | `localize.hdvp` | 缺陷定位过程 |
| **外部过程 (.hdvp)** | `visualize.hdvp` | 结果可视化过程 |
| **外部过程 (.hdvp)** | `batch_process.hdvp` | 批量处理过程 |

**架构设计模式**：

- `.hdvp` 文件采用模块化设计，每个文件封装一个独立功能
- `.hdev` 文件作为入口，调用 `.hdvp` 过程或内联实现相同逻辑
- 存在两套并行实现：
  - **模块化版本**：`main_transistor_inspection.hdev` 直接内联所有逻辑
  - **过程化版本**：`batch_process.hdvp` 通过调用各 `.hdvp` 过程实现

---

## 2. main_transistor_inspection.hdev 逐行分析

### 2.1 文件信息与配置区

| 行号 | 代码 | 作用 |
|------|------|------|
| 1 | `<?xml version="1.0" encoding="UTF-8"?>` | XML 声明，指定编码 |
| 2 | `<hdevelop file_version="1.2" halcon_version="26.05.0.0">` | 根元素，声明 HALCON 版本 26.05 |
| 3 | `<procedure name="main">` | 定义主过程 `main` |
| 4 | `<interface/>` | 接口声明（无输入输出参数） |
| 5 | `<body>` | 过程体开始 |
| 6-14 | `<c>* ...</c>` | 注释块：项目说明、模型信息 |
| 21 | `ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'` | 设置模型文件路径 |
| 22 | `DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'` | 设置预处理参数字典路径 |
| 23 | `OutputFolder := 'E:/halcon_project/output'` | 设置输出文件夹路径 |
| 26 | `AnomalyThreshold := 0.5` | 异常分数阈值（高于此值为 NG） |
| 28 | `RegionThreshold := 30` | 异常热力图二值化阈值（0-255） |
| 30 | `MinArea := 50` | 最小缺陷区域面积（过滤噪点） |

### 2.2 模型加载

| 行号 | 代码 | 作用 |
|------|------|------|
| 37 | `read_dl_model (ModelPath, DLModelHandle)` | 读取训练好的深度学习模型 |
| 38 | `read_dict (DictPath, [], [], DLPreprocessParams)` | 读取预处理参数字典 |
| 39 | `get_dict_tuple (DLPreprocessParams, 'model_type', ModelType)` | 获取模型类型（gc_anomaly_detection） |
| 40 | `get_dict_tuple (DLPreprocessParams, 'image_width', ImageWidth)` | 获取模型输入宽度（256） |
| 41 | `get_dict_tuple (DLPreprocessParams, 'image_height', ImageHeight)` | 获取模型输入高度（256） |

### 2.3 模式选择

| 行号 | 代码 | 作用 |
|------|------|------|
| 47 | `Mode := 2` | 设置运行模式：1=单张检测，2=批量检测 |

### 2.4 模式 1：单张检测

#### 2.4.1 窗口创建

| 行号 | 代码 | 作用 |
|------|------|------|
| 53 | `read_image (TestImage, 'E:/halcon_project/dataset/transistor/test/good/000.png')` | 读取测试图像 |
| 56 | `dev_close_window ()` | 关闭现有窗口 |
| 57 | `get_image_size (TestImage, ImgWidth, ImgHeight)` | 获取图像尺寸 |
| 58 | `dev_open_window (0, 0, 512, 512, 'black', WindowHandleLeft)` | 打开左窗口（原始图像） |
| 59 | `dev_open_window (0, 522, 512, 512, 'black', WindowHandleRight)` | 打开右窗口（检测结果） |
| 60 | `dev_set_part (0, 0, ImgHeight - 1, ImgWidth - 1)` | 设置显示区域 |

#### 2.4.2 预处理（内联版本）

| 行号 | 代码 | 作用 |
|------|------|------|
| 64 | `count_channels (TestImage, Channels)` | 检测图像通道数 |
| 65-69 | `if (Channels = 1)...else...endif` | 判断是否为灰度图 |
| 66 | `compose3 (TestImage, TestImage, TestImage, MultiChannelImage)` | 单通道扩展为三通道 |
| 68 | `copy_image (TestImage, MultiChannelImage)` | 三通道直接复制 |
| 71 | `zoom_image_size (MultiChannelImage, ImageResized, 256, 256, 'constant')` | 缩放到 256×256 |
| 73 | `convert_image_type (ImageResized, ImageReal, 'real')` | 类型转换为 real |
| 74 | `scale_image (ImageReal, ImagePreprocessed, 1.0, -127)` | 值域映射 [0,255] → [-127,128] |
| 76 | `full_domain (ImagePreprocessed, ImageFullDomain)` | 设置全图域 |
| 78 | `gen_dl_samples_from_images (ImageFullDomain, DLSample)` | 创建 DLSample 对象 |

#### 2.4.3 推理

| 行号 | 代码 | 作用 |
|------|------|------|
| 81 | `apply_dl_model (DLModelHandle, DLSample, [], DLResult)` | 执行模型推理 |
| 82 | `get_dict_object (AnomalyImage, DLResult, 'anomaly_image')` | 获取异常热力图 |
| 83 | `get_dict_tuple (DLResult, 'anomaly_score', AnomalyScore)` | 获取异常分数 |
| 84-88 | `if (AnomalyScore > AnomalyThreshold)...endif` | OK/NG 判定 |

#### 2.4.4 缺陷定位

| 行号 | 代码 | 作用 |
|------|------|------|
| 91 | `scale_image_max (AnomalyImage, AnomalyImageScaled)` | 缩放异常图到 [0,255] |
| 92 | `threshold (AnomalyImageScaled, AnomalyRegion, RegionThreshold, 255)` | 二值化提取异常区域 |
| 93 | `connection (AnomalyRegion, ConnectedRegions)` | 分离不连通区域 |
| 94 | `select_shape (ConnectedRegions, DefectRegions, 'area', 'and', MinArea, 999999)` | 过滤小面积噪点 |
| 95 | `count_obj (DefectRegions, NumDefects)` | 统计缺陷数量 |
| 96-100 | `if (NumDefects > 0)...else...endif` | 生成外接矩形框 |
| 97 | `shape_trans (DefectRegions, BoundingBoxes, 'rectangle1')` | 转换为矩形区域 |
| 99 | `gen_empty_obj (BoundingBoxes)` | 生成空对象 |

#### 2.4.5 坐标缩放（仿射变换）

| 行号 | 代码 | 作用 |
|------|------|------|
| 102-108 | `if (NumDefects > 0)...else...endif` | 判断是否有缺陷 |
| 103 | `hom_mat2d_identity (HomMat2D)` | 创建单位变换矩阵 |
| 104 | `hom_mat2d_scale (HomMat2D, ImgWidth / 256.0, ImgHeight / 256.0, 0, 0, HomMat2D)` | 设置缩放参数 |
| 105 | `affine_trans_region (BoundingBoxes, BoundingBoxesScaled, HomMat2D, 'nearest_neighbor')` | 应用仿射变换 |
| 107 | `gen_empty_obj (BoundingBoxesScaled)` | 生成空对象 |

#### 2.4.6 可视化

| 行号 | 代码 | 作用 |
|------|------|------|
| 112 | `dev_set_window (WindowHandleLeft)` | 切换到左窗口 |
| 113 | `dev_display (TestImage)` | 显示原始图像 |
| 114 | `dev_disp_text ('Original Image', 'window', 'top', 'left', 'white', 'box', 'false')` | 添加标签 |
| 117 | `dev_set_window (WindowHandleRight)` | 切换到右窗口 |
| 118 | `dev_set_part (0, 0, ImgHeight - 1, ImgWidth - 1)` | 设置显示区域 |
| 120 | `dev_display (TestImage)` | 显示原始图像 |
| 123-128 | `if (NumDefects > 0)...endif` | 绘制红色缺陷框 |
| 131-137 | `if (IsNG = 1)...else...endif` | 设置判定文本和颜色 |
| 138 | `get_image_size (TestImage, ImgWidth2, ImgHeight2)` | 获取图像尺寸 |
| 139 | `disp_message (WindowHandleRight, ResultText, 'image', ImgHeight2 / 2 - 30, -1, TextColor, 'false')` | 居中显示 OK/NG |
| 141 | `disp_message (WindowHandleRight, 'Score: ' + AnomalyScore$'.4f' + ' | Defects: ' + NumDefects, 'image', ImgHeight2 - 10, 12, 'white', 'false')` | 显示分数和缺陷数 |
| 143 | `dev_disp_text ('Press F5 to continue...', 'window', 'bottom', 'right', 'white', [], [])` | 提示继续 |
| 144 | `stop ()` | 暂停等待用户操作 |

### 2.5 模式 2：批量检测

#### 2.5.1 初始化

| 行号 | 代码 | 作用 |
|------|------|------|
| 150 | `InputFolder := 'E:/halcon_project/test'` | 设置输入文件夹 |
| 153 | `file_exists (OutputFolder, FileExists)` | 检查输出文件夹是否存在 |
| 154-156 | `if (FileExists = 0)...endif` | 如果不存在则创建 |
| 155 | `make_dir (OutputFolder)` | 创建输出文件夹 |
| 159 | `list_image_files (InputFolder, 'default', [], ImageFiles)` | 列出所有图像文件 |
| 160 | `tuple_length (ImageFiles, TotalImages)` | 获取图像总数 |
| 161 | `CountOK := 0` | OK 计数初始化 |
| 162 | `CountNG := 0` | NG 计数初始化 |
| 164-166 | 窗口创建 | 同上 |

#### 2.5.2 循环处理（第 168-277 行）

**核心逻辑**：与模式 1 相同，但包装在 `for` 循环中，并增加以下步骤：

| 行号 | 代码 | 作用 |
|------|------|------|
| 168 | `for Index := 0 to TotalImages - 1 by 1` | 遍历所有图像 |
| 169 | `tuple_select (ImageFiles, Index, CurrentImagePath)` | 获取当前图像路径 |
| 171 | `try` | 异常处理开始 |
| 172 | `read_image (CurrentImage, CurrentImagePath)` | 读取图像 |
| 173 | `get_image_size (CurrentImage, ImgWidth, ImgHeight)` | 获取尺寸 |
| 175-191 | 预处理 | 同模式 1 |
| 193-201 | 推理 | 同模式 1 |
| 203-221 | 定位 | 同模式 1 |
| 223-232 | 构建保存路径 | 按 OK/NG 分类命名 |
| 234-267 | 可视化并保存 | 同模式 1 + 保存结果 |
| 269-271 | 进度显示 | 显示处理进度 |
| 273 | `catch (Exception)` | 捕获异常 |
| 274 | `dev_disp_text ('Error on ' + CurrentImagePath + ': ' + Exception, 'window', 'bottom', 'right', 'red', [], [])` | 显示错误信息 |
| 275 | `wait_seconds (1)` | 等待 1 秒 |
| 276 | `endtry` | 异常处理结束 |
| 277 | `endfor` | 循环结束 |

#### 2.5.3 批量完成总结

| 行号 | 代码 | 作用 |
|------|------|------|
| 280 | `dev_open_window (0, 0, 400, 200, 'black', SummaryWindow)` | 打开汇总窗口 |
| 281-284 | 设置汇总文本 | Total/OK/NG 数量 |
| 285 | `disp_message (SummaryWindow, SummaryText, 'window', 20, 20, 'white', 'false')` | 显示汇总信息 |
| 287 | `dev_disp_text ('Batch complete. Results saved to ' + OutputFolder, 'window', 'bottom', 'right', 'white', [], [])` | 提示完成 |
| 288 | `stop ()` | 暂停 |

### 2.6 无效模式处理

| 行号 | 代码 | 作用 |
|------|------|------|
| 290 | `else` | 无效模式分支 |
| 291 | `dev_disp_text ('Invalid mode. Set Mode := 1 or Mode := 2 and re-run.', 'window', 'center', 'center', 'red', [], [])` | 显示错误提示 |
| 292 | `stop ()` | 暂停 |
| 293 | `endif` | 模式判断结束 |

---

## 3. tune_thresholds.hdev 逐行分析

### 3.1 文件结构

| 行号 | 代码 | 作用 |
|------|------|------|
| 1-2 | XML 声明和根元素 | HALCON 25.11 版本 |
| 4-27 | `load_model` 过程定义 | 内部过程，用于加载模型 |
| 29-47 | `preprocess` 过程定义 | 内部过程，用于预处理（使用 `preprocess_dl_model`） |
| 50-132 | `main` 过程 | 主逻辑，阈值调优 |

### 3.2 load_model 内部过程

| 行号 | 代码 | 作用 |
|------|------|------|
| 6-7 | 输入参数：`ModelPath`, `DictPath` | 模型路径和字典路径 |
| 8-9 | 输出参数：`DLModelHandle`, `DLPreprocessParams` | 模型句柄和预处理参数 |
| 13 | `read_dl_model (ModelPath, DLModelHandle)` | 读取模型 |
| 16 | `read_dict (DictPath, [], [], DLPreprocessParams)` | 读取字典 |
| 19-21 | `get_dict_tuple (...)` | 获取模型类型、宽度、高度 |
| 22 | `return ()` | 返回 |

### 3.3 preprocess 内部过程（⚠️ 使用已弃用算子）

| 行号 | 代码 | 作用 | 问题 |
|------|------|------|------|
| 31 | 输入参数：`Image`, `DLPreprocessParams` | 图像和预处理参数 | |
| 33 | 输出参数：`DLSample` | 预处理后的样本 | |
| 37 | `create_dict (DLSample)` | 创建空字典 | |
| 38 | `set_dict_tuple (DLSample, 'image', Image)` | 设置图像到字典 | ⚠️ 类型冲突 |
| 41 | `preprocess_dl_model (DLSample, DLPreprocessParams)` | 应用预处理 | ⚠️ HALCON 26 已移除 |
| 43 | `return ()` | 返回 | |

### 3.4 main 过程 - 阈值调优逻辑

#### 3.4.1 模型加载

| 行号 | 代码 | 作用 |
|------|------|------|
| 60-61 | 设置 `ModelPath` 和 `DictPath` | 模型路径 |
| 63 | `load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)` | 调用内部过程加载 |

#### 3.4.2 测试图像列表

| 行号 | 代码 | 作用 |
|------|------|------|
| 66-71 | 设置 `ImagePaths` 数组 | 包含 5 类测试图像各一张 |
| 73 | `tuple_length (ImagePaths, NumImages)` | 获取图像数量 |

#### 3.4.3 结果表头

| 行号 | 代码 | 作用 |
|------|------|------|
| 76 | `ResultHeader := 'Image | AnomalyScore | MaxPixelValue | SuggestedRegionThr'` | 结果表头 |
| 77 | `ResultLines := []` | 结果行数组 |

#### 3.4.4 循环处理（第 79-104 行）

| 行号 | 代码 | 作用 |
|------|------|------|
| 79 | `for i := 0 to NumImages - 1 by 1` | 遍历测试图像 |
| 80 | `read_image (Image, ImagePaths[i])` | 读取图像 |
| 81 | `preprocess (Image, DLPreprocessParams, DLSample)` | 预处理（使用弃用算子） |
| 84 | `apply_dl_model (DLModelHandle, DLSample, [], DLResult)` | 推理 |
| 85 | `get_dict_tuple (DLResult, 'anomaly_score', AnomalyScore)` | 获取异常分数 |
| 86 | `get_dict_tuple (DLResult, 'anomaly_image', AnomalyImage)` | 获取异常图像（⚠️ 类型问题） |
| 89 | `min_max_gray (AnomalyImage, AnomalyImage, 0, MinVal, MaxVal, Range)` | 分析异常图像值范围 |
| 90 | `intensity (AnomalyImage, AnomalyImage, MeanVal, Deviation)` | 计算均值和标准差 |
| 93 | `SuggestedThr := MeanVal + 2 * Deviation` | 建议阈值 = 均值 + 2×标准差 |
| 97-100 | 路径解析 | 提取文件名 |
| 101-103 | `ResultLines[i] := ...` | 格式化结果行 |
| 104 | `endfor` | 循环结束 |

#### 3.4.5 结果显示

| 行号 | 代码 | 作用 |
|------|------|------|
| 107 | `dev_close_window ()` | 关闭窗口 |
| 108 | `dev_open_window (0, 0, 700, 400, 'black', ResultWindow)` | 打开结果窗口 |
| 111-112 | 显示标题 | `=== Threshold Tuning Results ===` |
| 113-116 | 循环显示结果行 | 逐行显示 |
| 118-125 | 显示建议 | 如何设置阈值的指导 |
| 127 | `stop ()` | 暂停 |

---

## 4. load_model.hdvp 逐行分析

### 4.1 文件结构

`.hdvp` 文件是 HALCON 外部过程文件，使用 XML 格式定义：

| 行号 | 标签 | 作用 |
|------|------|------|
| 2 | `<procedure>` | 根元素 |
| 3 | `<name>load_model</name>` | 过程名称 |
| 4 | `<short_description>...</short_description>` | 简短描述 |
| 5 | `<location>external</location>` | 过程位置（外部） |
| 6-26 | `<parameter_list>` | 参数列表 |
| 28-47 | `<implementation>` | 实现代码（CDATA 块） |

### 4.2 参数定义

| 参数名 | 类型 | 语义类型 | 说明 |
|--------|------|----------|------|
| `ModelPath` | input | string | 模型文件路径 |
| `DictPath` | input | string | 预处理参数字典路径 |
| `DLModelHandle` | output | - | 深度学习模型句柄 |
| `DLPreprocessParams` | output | - | 预处理参数字典 |

### 4.3 实现代码

| 行号 | 代码 | 作用 |
|------|------|------|
| 32 | `read_dl_model (ModelPath, DLModelHandle)` | 读取模型文件 |
| 35 | `read_dict (DictPath, [], [], DLPreprocessParams)` | 读取预处理参数 |
| 38-40 | `get_dict_tuple (...)` | 获取模型元信息（类型、宽、高） |
| 42 | `dev_disp_text ('Model loaded: ' + ModelType + ' | Input: ' + ImageWidth + 'x' + ImageHeight, 'window', 'top', 'left', 'white', [], [])` | 显示加载信息 |
| 44 | `return ()` | 返回 |

---

## 5. preprocess.hdvp 逐行分析

### 5.1 参数定义

| 参数名 | 类型 | 语义类型 | 说明 |
|--------|------|----------|------|
| `Image` | input | image | 原始输入图像 |
| `DLPreprocessParams` | input | - | 预处理参数字典 |
| `DLSample` | output | - | 预处理后的样本对象 |

### 5.2 实现代码（⚠️ 使用已弃用算子）

| 行号 | 代码 | 作用 | 问题 |
|------|------|------|------|
| 27 | `create_dict (DLSample)` | 创建空字典 | |
| 28 | `set_dict_tuple (DLSample, 'image', Image)` | 将图像放入字典 | ⚠️ `set_dict_tuple` 不支持 iconic 类型 |
| 31 | `preprocess_dl_model (DLSample, DLPreprocessParams)` | 应用预处理 | ⚠️ HALCON 26 已移除该算子 |
| 33 | `return ()` | 返回 | |

**问题说明**：
1. `set_dict_tuple` 要求 `control` 类型参数，但图像是 `iconic` 类型，会导致类型冲突错误
2. `preprocess_dl_model` 在 HALCON 26 中已被移除，无法使用

**推荐替代方案**：使用 `gen_dl_samples_from_images` 替代，见 `main_transistor_inspection.hdev` 中的手动预处理流程。

---

## 6. infer.hdvp 逐行分析

### 6.1 参数定义

| 参数名 | 类型 | 语义类型 | 说明 |
|--------|------|----------|------|
| `DLModelHandle` | input | - | 模型句柄 |
| `DLSample` | input | - | 预处理后的样本 |
| `AnomalyThreshold` | input | - | 异常分数阈值 |
| `IsNG` | output | - | 是否为不合格品（0/1） |
| `AnomalyScore` | output | - | 异常分数 |
| `AnomalyImage` | output | image | 异常热力图 |

### 6.2 实现代码（⚠️ 存在类型问题）

| 行号 | 代码 | 作用 | 问题 |
|------|------|------|------|
| 42 | `apply_dl_model (DLModelHandle, DLSample, [], DLResult)` | 执行推理 | |
| 45 | `get_dict_tuple (DLResult, 'anomaly_image', AnomalyImage)` | 获取异常图像 | ⚠️ 应使用 `get_dict_object` |
| 46 | `get_dict_tuple (DLResult, 'anomaly_score', AnomalyScore)` | 获取异常分数 | |
| 49-53 | `if (AnomalyScore > AnomalyThreshold)...endif` | OK/NG 判定 | |
| 55 | `return ()` | 返回 | |

**问题说明**：`anomaly_image` 是图像对象（iconic 类型），需要使用 `get_dict_object` 而非 `get_dict_tuple` 来提取。

---

## 7. localize.hdvp 逐行分析

### 7.1 参数定义

| 参数名 | 类型 | 语义类型 | 说明 |
|--------|------|----------|------|
| `AnomalyImage` | input | image | 异常热力图 |
| `RegionThreshold` | input | - | 二值化阈值 |
| `MinArea` | input | - | 最小区域面积 |
| `DefectRegions` | output | region | 缺陷区域 |
| `BoundingBoxes` | output | region | 边界框 |
| `NumDefects` | output | - | 缺陷数量 |

### 7.2 实现代码

| 行号 | 代码 | 作用 |
|------|------|------|
| 42 | `scale_image_max (AnomalyImage, AnomalyImageScaled)` | 缩放值域到 [0,255] |
| 45 | `threshold (AnomalyImageScaled, AnomalyRegion, RegionThreshold, 255)` | 二值化提取异常区域 |
| 48 | `connection (AnomalyRegion, ConnectedRegions)` | 分离不连通区域 |
| 51 | `select_shape (ConnectedRegions, DefectRegions, 'area', 'and', MinArea, 999999)` | 过滤小面积噪点 |
| 54 | `count_obj (DefectRegions, NumDefects)` | 统计缺陷数量 |
| 57-61 | `if (NumDefects > 0)...else...endif` | 生成边界框或空对象 |
| 58 | `shape_trans (DefectRegions, BoundingBoxes, 'rectangle1')` | 转换为矩形 |
| 60 | `gen_empty_obj (BoundingBoxes)` | 生成空对象 |
| 63 | `return ()` | 返回 |

**注意**：此过程处理的是 256×256 的异常热力图，边界框坐标也是 256×256 坐标系，需要在调用方进行坐标映射。

---

## 8. visualize.hdvp 逐行分析

### 8.1 参数定义

| 参数名 | 类型 | 语义类型 | 说明 |
|--------|------|----------|------|
| `Image` | input | image | 原始图像 |
| `AnomalyImage` | input | image | 异常热力图 |
| `IsNG` | input | - | 是否为 NG |
| `AnomalyScore` | input | - | 异常分数 |
| `BoundingBoxes` | input | region | 边界框 |
| `NumDefects` | input | - | 缺陷数量 |
| `WindowHandleLeft` | input | - | 左窗口句柄 |
| `WindowHandleRight` | input | - | 右窗口句柄 |
| `SavePath` | input | string | 保存路径（可选） |

### 8.2 实现代码

#### 8.2.1 左窗口显示

| 行号 | 代码 | 作用 |
|------|------|------|
| 57 | `dev_set_window (WindowHandleLeft)` | 切换到左窗口 |
| 58 | `dev_display (Image)` | 显示原始图像 |
| 59 | `dev_disp_text ('Original Image', 'window', 'top', 'left', 'white', 'box', 'false')` | 添加标签 |

#### 8.2.2 右窗口显示（热力图叠加）

| 行号 | 代码 | 作用 |
|------|------|------|
| 62 | `dev_set_window (WindowHandleRight)` | 切换到右窗口 |
| 65 | `scale_image_max (AnomalyImage, AnomalyImageScaled)` | 缩放异常图到 [0,255] |
| 66 | `convert_image_type (AnomalyImageScaled, AnomalyImageByte, 'byte')` | 转换为 byte 类型 |
| 69 | `dev_display (Image)` | 显示原始图像作为底层 |
| 73-78 | 图像混合操作 | 将异常热力图以 50% 透明度叠加到原图 |
| 73 | `convert_image_type (Image, ImageReal, 'real')` | 原图转 real |
| 74 | `convert_image_type (AnomalyImageByte, AnomalyReal, 'real')` | 异常图转 real |
| 75 | `scale_image (ImageReal, ImageBase, 0.5, 0)` | 原图缩放 0.5 |
| 76 | `scale_image (AnomalyReal, AnomalyOverlay, 0.5, 0)` | 异常图缩放 0.5 |
| 77 | `add_image (ImageBase, AnomalyOverlay, BlendedReal, 1.0, 0)` | 图像相加 |
| 78 | `convert_image_type (BlendedReal, BlendedImage, 'byte')` | 转回 byte |
| 81 | `dev_display (BlendedImage)` | 显示混合结果 |

#### 8.2.3 绘制边界框

| 行号 | 代码 | 作用 |
|------|------|------|
| 84-89 | `if (NumDefects > 0)...endif` | 绘制红色边界框 |
| 85 | `dev_set_color ('red')` | 设置红色 |
| 86 | `dev_set_line_width (2)` | 设置线宽 |
| 87 | `dev_set_draw ('margin')` | 设置绘制模式为边缘 |
| 88 | `dev_display (BoundingBoxes)` | 显示边界框 |

#### 8.2.4 显示结果文本

| 行号 | 代码 | 作用 |
|------|------|------|
| 92-98 | `if (IsNG = 1)...else...endif` | 设置结果文本和颜色 |
| 100 | `get_image_size (Image, ImgWidth, ImgHeight)` | 获取图像尺寸 |
| 103 | `disp_message (WindowHandleRight, ResultText + ' | Score: ' + AnomalyScore$'.4f', 'window', 12, 12, TextColor, 'false')` | 显示状态和分数 |
| 105-107 | `if (NumDefects > 0)...endif` | 显示缺陷数量 |

#### 8.2.5 保存结果

| 行号 | 代码 | 作用 |
|------|------|------|
| 110 | `if (|SavePath| > 0)` | 检查保存路径是否非空 |
| 112 | `dump_window_image (ResultImage, WindowHandleRight)` | 捕获窗口图像 |
| 113 | `write_image (ResultImage, 'png', 0, SavePath)` | 保存为 PNG 文件 |
| 115 | `return ()` | 返回 |

---

## 9. batch_process.hdvp 逐行分析

### 9.1 参数定义

| 参数名 | 类型 | 语义类型 | 说明 |
|--------|------|----------|------|
| `InputFolder` | input | string | 输入文件夹 |
| `OutputFolder` | input | string | 输出文件夹 |
| `DLModelHandle` | input | - | 模型句柄 |
| `DLPreprocessParams` | input | - | 预处理参数 |
| `AnomalyThreshold` | input | - | 异常分数阈值 |
| `RegionThreshold` | input | - | 区域阈值 |
| `MinArea` | input | - | 最小面积 |
| `WindowHandleLeft` | input | - | 左窗口句柄 |
| `WindowHandleRight` | input | - | 右窗口句柄 |

### 9.2 实现代码

#### 9.2.1 初始化

| 行号 | 代码 | 作用 |
|------|------|------|
| 57 | `file_exists (OutputFolder, FileExists)` | 检查输出文件夹 |
| 58-60 | `if (FileExists = 0)...endif` | 创建输出文件夹 |
| 63 | `list_image_files (InputFolder, 'default', [], ImageFiles)` | 列出图像文件 |
| 67 | `tuple_length (ImageFiles, TotalImages)` | 获取图像总数 |
| 70-71 | `CountOK := 0`, `CountNG := 0` | 初始化计数器 |

#### 9.2.2 循环处理（第 74-115 行）

| 行号 | 代码 | 作用 |
|------|------|------|
| 74 | `for Index := 0 to TotalImages - 1 by 1` | 遍历图像 |
| 76 | `tuple_select (ImageFiles, Index, CurrentImagePath)` | 获取路径 |
| 79 | `try` | 异常处理开始 |
| 80 | `read_image (CurrentImage, CurrentImagePath)` | 读取图像 |
| 83 | `preprocess (CurrentImage, DLPreprocessParams, DLSample)` | ⚠️ 调用有问题的 preprocess |
| 84 | `infer (DLModelHandle, DLSample, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)` | ⚠️ 调用有问题的 infer |
| 85 | `localize (AnomalyImage, RegionThreshold, MinArea, DefectRegions, BoundingBoxes, NumDefects)` | 调用 localize |
| 88-94 | OK/NG 判定和计数 | 分类计数 |
| 97-98 | 构建保存路径 | OK_NNNN.png / NG_NNNN.png |
| 101 | `visualize (CurrentImage, AnomalyImage, IsNG, AnomalyScore, BoundingBoxes, NumDefects, WindowHandleLeft, WindowHandleRight, SavePath)` | 调用 visualize |
| 104 | `dev_disp_text ('Processing ' + (Index + 1) + '/' + TotalImages + '  OK:' + CountOK + '  NG:' + CountNG, 'window', 'bottom', 'right', 'white', [], [])` | 显示进度 |
| 107 | `wait_seconds (0.5)` | 等待 0.5 秒 |
| 109 | `catch (Exception)` | 捕获异常 |
| 111 | `dev_disp_text ('Error on ' + CurrentImagePath + ': ' + Exception, 'window', 'bottom', 'right', 'red', [], [])` | 显示错误 |
| 112 | `wait_seconds (1)` | 等待 1 秒 |
| 113 | `endtry` | 异常处理结束 |
| 115 | `endfor` | 循环结束 |

#### 9.2.3 汇总显示

| 行号 | 代码 | 作用 |
|------|------|------|
| 118 | `dev_open_window (0, 0, 400, 200, 'black', SummaryWindow)` | 打开汇总窗口 |
| 119-123 | 设置并显示汇总文本 | Total/OK/NG 数量 |
| 125 | `return ()` | 返回 |

---

## 10. 文件间调用关系图

```
main_transistor_inspection.hdev (主入口)
    │
    ├─ Mode=1: 单张检测
    │   ├─ [内联] 模型加载
    │   ├─ [内联] 预处理 (6步)
    │   ├─ [内联] 推理
    │   ├─ [内联] 缺陷定位
    │   ├─ [内联] 坐标缩放 (仿射变换)
    │   └─ [内联] 可视化
    │
    └─ Mode=2: 批量检测
        ├─ [内联] 模型加载
        ├─ for 循环:
        │   ├─ [内联] 预处理
        │   ├─ [内联] 推理
        │   ├─ [内联] 缺陷定位
        │   ├─ [内联] 坐标缩放
        │   ├─ [内联] 可视化
        │   └─ [内联] 保存结果
        └─ [内联] 汇总显示

tune_thresholds.hdev (阈值调优)
    │
    ├─ load_model (内部过程)
    │   ├─ read_dl_model
    │   └─ read_dict
    │
    ├─ preprocess (内部过程) ⚠️ 使用弃用算子
    │   └─ preprocess_dl_model
    │
    └─ [内联] 推理 + 分析 + 结果显示

batch_process.hdvp (批量处理)
    │
    ├─ preprocess.hdvp ⚠️ 使用弃用算子
    ├─ infer.hdvp ⚠️ 类型问题
    ├─ localize.hdvp (正常)
    └─ visualize.hdvp (正常)

模块依赖关系:
    load_model.hdvp ─────┐
                         ├──► batch_process.hdvp
    preprocess.hdvp ─────┤
                         │
    infer.hdvp ──────────┤
                         │
    localize.hdvp ───────┤
                         │
    visualize.hdvp ──────┘
```

---

## 11. 关键差异对比

### 11.1 两套实现方式对比

| 特性 | main_transistor_inspection.hdev | tune_thresholds.hdev + batch_process.hdvp |
|------|--------------------------------|-------------------------------------------|
| **预处理方式** | 手动 6 步预处理（兼容 HALCON 26） | 使用 `preprocess_dl_model`（⚠️ HALCON 26 已移除） |
| **图像提取方式** | `get_dict_object`（正确处理 iconic 类型） | `get_dict_tuple`（⚠️ 类型冲突） |
| **模块化程度** | 全内联，无过程调用 | 模块化设计，使用 .hdvp 过程 |
| **HALCON 版本兼容性** | HALCON 26（兼容） | HALCON 25.11（不兼容 26） |
| **坐标缩放** | 包含仿射变换，映射到原图 | 无坐标缩放，边界框在 256×256 坐标系 |
| **可视化效果** | OK/NG 居中大字 + 分数底部显示 | 热力图叠加 + 顶部状态栏 |

### 11.2 `.hdev` vs `.hdvp` 文件格式

| 特性 | `.hdev` | `.hdvp` |
|------|---------|---------|
| **文件格式** | 完整 HDevelop 脚本 | 外部过程定义 |
| **代码存储** | `<l>`/`<c>` 标签 | CDATA 块 |
| **过程定义** | `<procedure>` 内联 | 独立 `<procedure>` 标签 |
| **接口声明** | `<interface>` 在过程内 | `<parameter_list>` 显式定义 |
| **调用方式** | 直接运行 | 作为过程被其他脚本调用 |

### 11.3 已知问题与修复建议

| 问题 | 位置 | 影响 | 修复建议 |
|------|------|------|----------|
| `preprocess_dl_model` 不存在 | `preprocess.hdvp`, `tune_thresholds.hdev` | HALCON 26 运行报错 | 使用手动 6 步预处理替代 |
| `set_dict_tuple` 类型冲突 | `preprocess.hdvp` | 无法传递图像 | 使用 `gen_dl_samples_from_images` |
| `get_dict_tuple` 获取图像 | `infer.hdvp`, `tune_thresholds.hdev` | 获取到 control 类型而非 image | 使用 `get_dict_object` |
| 坐标未缩放 | `batch_process.hdvp` | 边界框位置错误 | 添加仿射变换坐标映射 |

---

## 总结

source 目录包含了两套并行的缺陷检测实现：

1. **main_transistor_inspection.hdev**：完整的内联实现，兼容 HALCON 26，包含单张和批量两种模式，是项目的**主要运行入口**

2. **tune_thresholds.hdev + .hdvp 过程组**：模块化设计，但使用了 HALCON 26 中已移除的 `preprocess_dl_model` 算子，存在类型处理问题，需要**升级修复**才能在 HALCON 26 中使用

**推荐使用方式**：
- 直接运行 `main_transistor_inspection.hdev` 进行缺陷检测
- 使用 `tune_thresholds.hdev` 的思路进行阈值调优（需先修复预处理方式）
- 将 `.hdvp` 过程组按 `main_transistor_inspection.hdev` 的模式进行升级，实现真正的模块化调用