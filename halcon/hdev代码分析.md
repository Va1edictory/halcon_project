# HALCON 缺陷检测推理脚本代码分析

> 分析目录：`E:\借鉴\vision-project-main\vision-project-main\halcon`  
> 分析文件：`predict.hdev`、`predict_template.hdev`、`predict_batch_template.hdev`  
> 分析日期：2026/06/22

---

## 目录

1. [文件总览与关系](#1-文件总览与关系)
2. [predict.hdev 逐行分析](#2-predicthdev-逐行分析)
3. [predict_template.hdev 逐行分析](#3-predict_templatehdev-逐行分析)
4. [predict_batch_template.hdev 逐行分析](#4-predict_batch_templatehdev-逐行分析)
5. [三个文件核心差异对比](#5-三个文件核心差异对比)
6. [HALCON 算子速查表](#6-halcon-算子速查表)

---

## 1. 文件总览与关系

这三个 `.hdev` 文件都是 **HALCON HDevelop** 深度学习推理脚本，用于**缺陷检测（Defect Detection）**任务。它们共享同一套核心推理流程：

```
加载预处理参数 → 加载模型 → 读取图片 → 生成DL样本 → 预处理 → 推理 → 提取结果 → 输出JSON → 清理
```

| 文件 | 用途 | 特点 |
|------|------|------|
| `predict.hdev` | 单图推理（命令行版） | 通过 `argv[]` 接收外部参数，直接运行 |
| `predict_template.hdev` | 单图推理（模板版） | 使用 `{占位符}`，由 Python 后端动态替换后生成实际脚本 |
| `predict_batch_template.hdev` | 批量推理（模板版） | 支持多张图片批量处理，带进度文件反馈，由 Python 后端动态替换 |

---

## 2. predict.hdev 逐行分析

> **用途**：单张图片推理，通过命令行参数接收输入，适合直接由外部程序（如 Python）调用 HALCON 引擎执行。  
> **HALCON 版本**：24.11.1.0

```hdev
<?xml version="1.0" encoding="UTF-8"?>
```
**第1行**：XML 声明，指定版本 1.0 和 UTF-8 编码。`.hdev` 文件本质上是 XML 格式的脚本文件。

```hdev
<hdevelop file_version="1.2" halcon_version="24.11.1.0">
```
**第2行**：根元素 `<hdevelop>`，声明 HDevelop 文件格式版本为 `1.2`，以及创建/兼容的 HALCON 版本为 `24.11.1.0`。

```hdev
<procedure name="main">
```
**第3行**：定义一个名为 `main` 的过程（procedure）。在 HDevelop 中，`main` 是默认的入口过程，程序从这里开始执行。

```hdev
<interface/>
```
**第4行**：接口声明，为空表示该过程没有输入/输出参数（所有数据通过全局变量或命令行参数传递）。

```hdev
<body>
```
**第5行**：过程体开始标签，包含实际的程序代码。

```hdev
<c>* ===== 缺陷检测推理脚本（命令行版）=====</c>
```
**第6行**：`<c>` 标签表示**注释行**。注释说明了该脚本的用途：缺陷检测推理脚本，命令行版本。

```hdev
<c>* argv[1] = 图片路径</c>
```
**第7行**：注释说明第一个命令行参数 `argv[1]` 是待检测图片的文件路径。

```hdev
<c>* argv[2] = 结果JSON输出路径</c>
```
**第8行**：注释说明第二个命令行参数 `argv[2]` 是推理结果 JSON 文件的输出路径。

```hdev
<c>* argv[3] = 模型.hdl路径</c>
```
**第9行**：注释说明第三个命令行参数 `argv[3]` 是训练好的深度学习模型文件（`.hdl` 格式）的路径。

```hdev
<c>* argv[4] = 预处理.hdict路径</c>
```
**第10行**：注释说明第四个命令行参数 `argv[4]` 是预处理参数字典文件（`.hdict` 格式）的路径。

```hdev
<l>ImageFile := argv[1]</l>
```
**第11行**：`<l>` 标签表示**代码行**。将命令行参数 `argv[1]`（图片路径）赋值给变量 `ImageFile`。

```hdev
<l>ResultFile := argv[2]</l>
```
**第12行**：将命令行参数 `argv[2]`（结果输出路径）赋值给变量 `ResultFile`。

```hdev
<l>ModelFile := argv[3]</l>
```
**第13行**：将命令行参数 `argv[3]`（模型文件路径）赋值给变量 `ModelFile`。

```hdev
<l>PreprocessFile := argv[4]</l>
```
**第14行**：将命令行参数 `argv[4]`（预处理参数文件路径）赋值给变量 `PreprocessFile`。

```hdev
<c>* --- 加载预处理参数 ---</c>
```
**第15行**：注释，标记“加载预处理参数”步骤的开始。

```hdev
<l>read_dict (PreprocessFile, [], [], DLPreprocessParam)</l>
```
**第16行**：调用 `read_dict` 算子，从 `PreprocessFile` 指定的 `.hdict` 文件中读取预处理参数字典，存储到变量 `DLPreprocessParam` 中。`[]` 表示使用默认参数。

```hdev
<c>* --- 加载模型 ---</c>
```
**第17行**：注释，标记“加载模型”步骤的开始。

```hdev
<l>read_dl_model (ModelFile, DLModelHandle)</l>
```
**第18行**：调用 `read_dl_model` 算子，从 `ModelFile` 指定的 `.hdl` 文件中读取深度学习模型，返回模型句柄 `DLModelHandle`。该句柄后续用于推理。

```hdev
<c>* --- 获取类别名称 ---</c>
```
**第19行**：注释，标记“获取类别名称”步骤的开始。

```hdev
<l>get_dl_model_param (DLModelHandle, 'class_names', AllClassNames)</l>
```
**第20行**：调用 `get_dl_model_param` 算子，从模型句柄 `DLModelHandle` 中获取参数 `'class_names'`（类别名称列表），存储到 `AllClassNames` 中。这是模型训练时定义的类别标签。

```hdev
<c>* --- 读取图片 ---</c>
```
**第21行**：注释，标记“读取图片”步骤的开始。

```hdev
<l>read_image (Image, ImageFile)</l>
```
**第22行**：调用 `read_image` 算子，从 `ImageFile` 路径读取图片文件，存储到图像变量 `Image` 中。HALCON 支持多种图片格式（BMP、PNG、JPG、TIFF 等）。

```hdev
<c>* --- 生成DL样本 ---</c>
```
**第23行**：注释，标记“生成深度学习样本”步骤的开始。

```hdev
<l>gen_dl_samples_from_images (Image, DLSampleBatch)</l>
```
**第24行**：调用 `gen_dl_samples_from_images` 算子，将读取的图像 `Image` 转换为深度学习样本格式 `DLSampleBatch`。HALCON 的深度学习模块要求输入必须是特定的 `DLSample` 结构。

```hdev
<c>* --- 预处理 ---</c>
```
**第25行**：注释，标记“预处理”步骤的开始。

```hdev
<l>preprocess_dl_samples (DLSampleBatch, DLPreprocessParam)</l>
```
**第26行**：调用 `preprocess_dl_samples` 算子，使用加载的预处理参数 `DLPreprocessParam` 对样本 `DLSampleBatch` 进行预处理。预处理通常包括：归一化、尺寸调整、通道顺序转换等，确保输入符合模型要求。

```hdev
<c>* --- 推理 ---</c>
```
**第27行**：注释，标记“推理”步骤的开始。

```hdev
<l>apply_dl_model (DLModelHandle, DLSampleBatch, [], DLResultBatch)</l>
```
**第28行**：调用 `apply_dl_model` 算子，使用模型句柄 `DLModelHandle` 对预处理后的样本 `DLSampleBatch` 进行前向推理（forward inference）。`[]` 表示使用默认输出参数设置。推理结果存储在 `DLResultBatch` 中。

```hdev
<c>* --- 提取结果 ---</c>
```
**第29行**：注释，标记“提取结果”步骤的开始。

```hdev
<l>TopClass := DLResultBatch.classification_class_names[0]</l>
```
**第30行**：从推理结果字典 `DLResultBatch` 中提取 `classification_class_names` 数组的第 0 个元素，赋值给 `TopClass`。这是置信度最高的类别名称（Top-1 预测）。

```hdev
<l>TopConfidence := DLResultBatch.classification_confidences[0]</l>
```
**第31行**：从推理结果字典中提取 `classification_confidences` 数组的第 0 个元素，赋值给 `TopConfidence`。这是 Top-1 类别的置信度分数（通常为 0~1 之间的概率值）。

```hdev
<l>AllConfidences := DLResultBatch.classification_confidences</l>
```
**第32行**：将所有的类别置信度数组赋值给 `AllConfidences`，用于后续构建完整的 JSON 结果（包含所有类别的置信度）。

```hdev
<c>* --- 写入结果JSON ---</c>
```
**第33行**：注释，标记“写入结果 JSON”步骤的开始。

```hdev
<c>* 构造简单的结果字符串供后端解析</c>
```
**第34行**：注释说明接下来是手动构造 JSON 字符串，供后端（Python）程序解析。

```hdev
<l>NumClasses := |AllClassNames|</l>
```
**第35行**：计算 `AllClassNames` 数组的长度（即类别总数），赋值给 `NumClasses`。HALCON 中使用 `|变量|` 表示取元组/数组的长度。

```hdev
<l>JSONLine := '{"top_class":"' + TopClass + '","top_confidence":' + TopConfidence + ',"classes":['</l>
```
**第36行**：初始化 JSON 字符串 `JSONLine`，包含 `top_class`（最高置信度类别）、`top_confidence`（最高置信度值）和 `classes` 数组的开头。

```hdev
<l>for I := 0 to NumClasses - 1 by 1</l>
```
**第37行**：`for` 循环开始，遍历所有类别。`I` 从 0 开始，到 `NumClasses - 1` 结束，步长为 1。

```hdev
<l>    JSONLine := JSONLine + '{"class":"' + AllClassNames[I] + '","confidence":' + AllConfidences[I] + '}'</l>
```
**第38行**：在循环中，为每个类别追加一个 JSON 对象，包含该类别的名称 `AllClassNames[I]` 和置信度 `AllConfidences[I]`。

```hdev
<l>    if (I &lt; NumClasses - 1)</l>
```
**第39行**：`if` 条件判断，如果当前类别不是最后一个（索引小于 `NumClasses - 1`）。注意 `&lt;` 是 XML 中对 `<` 的转义。

```hdev
<l>        JSONLine := JSONLine + ','</l>
```
**第40行**：如果不是最后一个类别，则在 JSON 对象后追加逗号 `,`，以符合 JSON 数组的格式要求。

```hdev
<l>    endif</l>
```
**第41行**：结束 `if` 条件块。

```hdev
<l>endfor</l>
```
**第42行**：结束 `for` 循环。

```hdev
<l>JSONLine := JSONLine + ']}'</l>
```
**第43行**：JSON 数组和 JSON 对象闭合，完成整个 JSON 字符串的构造。

```hdev
<l>file_open (ResultFile, FileHandle, 'output', 'utf8')</l>
```
**第44行**：调用 `file_open` 算子，以输出模式（`'output'`）和 UTF-8 编码打开 `ResultFile`，返回文件句柄 `FileHandle`。

```hdev
<l>file_write_string (FileHandle, JSONLine)</l>
```
**第45行**：调用 `file_write_string` 算子，将构造好的 JSON 字符串 `JSONLine` 写入到文件中。

```hdev
<l>file_close (FileHandle)</l>
```
**第46行**：调用 `file_close` 算子，关闭文件句柄，释放资源并确保数据落盘。

```hdev
<c>* --- 清理 ---</c>
```
**第47行**：注释，标记“清理”步骤的开始。

```hdev
<l>clear_dl_model (DLModelHandle)</l>
```
**第48行**：调用 `clear_dl_model` 算子，释放深度学习模型 `DLModelHandle` 占用的内存和资源。这是良好的资源管理习惯，防止内存泄漏。

```hdev
</body>
```
**第49行**：过程体结束标签。

```hdev
<docu id="main">
<parameters/>
</docu>
```
**第50-52行**：文档说明块，`<docu>` 标签用于记录过程的文档信息。`id="main"` 对应 `main` 过程。`<parameters/>` 为空表示没有参数文档。

```hdev
</procedure>
</hdevelop>
```
**第53-54行**：关闭 `procedure` 和 `hdevelop` 标签，文件结束。

---

## 3. predict_template.hdev 逐行分析

> **用途**：单图推理模板，由 Python 后端进行字符串替换后生成实际可执行的 `.hdev` 文件。  
> **HALCON 版本**：25.11.0.0  
> **与 `predict.hdev` 的核心区别**：使用 `{占位符}` 代替命令行参数，且使用了 HALCON 25.11 的新算子。

```hdev
<?xml version="1.0" encoding="UTF-8"?>
```
**第1行**：同上，XML 声明。

```hdev
<hdevelop file_version="1.2" halcon_version="25.11.0.0">
```
**第2行**：根元素，HALCON 版本为 `25.11.0.0`（比 `predict.hdev` 更新）。

```hdev
<procedure name="main">
<interface/>
<body>
```
**第3-5行**：同上，定义 `main` 过程，空接口，开始过程体。

```hdev
<c>* ===== 缺陷检测推理脚本（命令行版 - HALCON 25.11）=====</c>
```
**第6行**：注释，说明这是 HALCON 25.11 版本的单图推理脚本。

```hdev
<c>* 路径由 Python 后端动态替换</c>
```
**第7行**：注释，说明文件中的路径不是硬编码的，而是由 Python 后端程序动态替换进去的。

```hdev
<l>ImageFile := '{IMAGE_PATH}'</l>
```
**第8行**：将字符串 `'{IMAGE_PATH}'` 赋值给 `ImageFile`。这是一个**占位符**，Python 后端会用实际的图片路径替换它。

```hdev
<l>ResultFile := '{RESULT_PATH}'</l>
```
**第9行**：占位符 `'{RESULT_PATH}'`，将被替换为实际的 JSON 结果输出路径。

```hdev
<l>ModelFile := '{MODEL_PATH}'</l>
```
**第10行**：占位符 `'{MODEL_PATH}'`，将被替换为实际的模型文件路径。

```hdev
<l>PreprocessFile := '{HDICT_PATH}'</l>
```
**第11行**：占位符 `'{HDICT_PATH}'`，将被替换为实际的预处理参数字典路径。

```hdev
<c>* --- 加载预处理参数 ---</c>
```
**第12行**：注释，标记预处理参数加载步骤。

```hdev
<l>read_dict (PreprocessFile, [], [], DLPreprocessParam)</l>
```
**第13行**：同上，读取预处理参数字典。

```hdev
<c>* --- 加载模型 ---</c>
```
**第14行**：注释。

```hdev
<l>read_dl_model (ModelFile, DLModelHandle)</l>
```
**第15行**：同上，读取深度学习模型。

```hdev
<c>* --- 读取图片 ---</c>
```
**第16行**：注释。

```hdev
<l>read_image (Image, ImageFile)</l>
```
**第17行**：同上，读取图片。

```hdev
<c>* --- 生成DL样本 ---</c>
```
**第18行**：注释。

```hdev
<l>gen_dl_samples_from_images (Image, DLSampleBatch)</l>
```
**第19行**：同上，生成深度学习样本。

```hdev
<c>* --- 预处理 ---</c>
```
**第20行**：注释。

```hdev
<l>preprocess_dl_samples (DLSampleBatch, DLPreprocessParam)</l>
```
**第21行**：同上，预处理样本。

```hdev
<c>* --- 推理 ---</c>
```
**第22行**：注释。

```hdev
<l>apply_dl_model (DLModelHandle, DLSampleBatch, [], DLResultBatch)</l>
```
**第23行**：同上，执行模型推理。

```hdev
<c>* --- 提取结果 ---</c>
```
**第24行**：注释。

```hdev
<c>* classification_confidences 按降序排列，classification_class_names 与之对应，[0] 即 top1</c>
```
**第25行**：关键注释！说明 `classification_confidences` 已经按**降序**排列，`classification_class_names` 与之对应，所以索引 `[0]` 就是 Top-1 结果。这省去了手动排序的步骤。

```hdev
<l>AllConfidences := DLResultBatch.classification_confidences</l>
```
**第26行**：提取所有置信度数组。

```hdev
<l>AllClassNames := DLResultBatch.classification_class_names</l>
```
**第27行**：提取所有类别名称数组。

```hdev
<c>* 如果 classification_class_names 键不存在，会在此处报错，便于排查</c>
```
**第28行**：注释说明这是一个天然的检查点——如果模型输出格式异常（缺少 `classification_class_names` 键），程序会在这里报错，便于调试排查。

```hdev
<c>* confidences 已按降序排列，class_names 与之对应，所以 [0] 就是 top1</c>
```
**第29行**：再次强调 `[0]` 就是 Top-1 结果，因为结果已经排序。

```hdev
<l>TopClass := AllClassNames[0]</l>
```
**第30行**：取排序后的第 0 个类别作为 Top-1 类别。

```hdev
<l>TopConfidence := AllConfidences[0]</l>
```
**第31行**：取排序后的第 0 个置信度作为 Top-1 置信度。

```hdev
<c>* --- 写入结果JSON ---</c>
```
**第32行**：注释。

```hdev
<l>NumClasses := |AllClassNames|</l>
```
**第33行**：计算类别总数。

```hdev
<l>JSONLine := '{"top_class":"' + TopClass + '","top_confidence":' + TopConfidence + ',"classes":['</l>
```
**第34行**：初始化 JSON 字符串。

```hdev
<l>for I := 0 to NumClasses - 1 by 1</l>
```
**第35行**：循环遍历所有类别。

```hdev
<l>    JSONLine := JSONLine + '{"class":"' + AllClassNames[I] + '","confidence":' + AllConfidences[I] + '}'</l>
```
**第36行**：追加每个类别的 JSON 对象。

```hdev
<l>    if (I &lt; NumClasses - 1)</l>
```
**第37行**：判断是否最后一个类别。

```hdev
<l>        JSONLine := JSONLine + ','</l>
```
**第38行**：非最后一个则追加逗号。

```hdev
<l>    endif</l>
```
**第39行**：结束 if。

```hdev
<l>endfor</l>
```
**第40行**：结束 for。

```hdev
<l>JSONLine := JSONLine + ']}'</l>
```
**第41行**：闭合 JSON 结构。

```hdev
<l>open_file (ResultFile, 'output', FileHandle)</l>
```
**第42行**：⚠️ **关键差异**：使用了 HALCON 25.11 的新算子 `open_file`（而非旧版的 `file_open`）。功能相同：以输出模式打开文件，返回句柄 `FileHandle`。参数顺序和名称有所调整。

```hdev
<l>fwrite_string (FileHandle, JSONLine)</l>
```
**第43行**：使用 `fwrite_string`（新版算子）将 JSON 字符串写入文件。与旧版 `file_write_string` 功能相同，只是命名风格统一为 `f` 前缀。

```hdev
<l>close_file (FileHandle)</l>
```
**第44行**：使用 `close_file`（新版算子）关闭文件。与旧版 `file_close` 对应。

```hdev
<c>* --- 清理 ---</c>
```
**第45行**：注释。

```hdev
<l>clear_dl_model (DLModelHandle)</l>
```
**第46行**：同上，释放模型资源。

```hdev
</body>
<docu id="main">
<parameters/>
</docu>
</procedure>
</hdevelop>
```
**第47-52行**：关闭过程体、文档块、过程和文件。

---

## 4. predict_batch_template.hdev 逐行分析

> **用途**：批量图片推理模板，支持一次性处理多张图片，并实时写入进度文件供前端展示进度条。  
> **HALCON 版本**：25.11.0.0

```hdev
<?xml version="1.0" encoding="UTF-8"?>
```
**第1行**：XML 声明。

```hdev
<hdevelop file_version="1.2" halcon_version="25.11.0.0">
```
**第2行**：HALCON 25.11 版本。

```hdev
<procedure name="main">
<interface/>
<body>
```
**第3-5行**：main 过程定义。

```hdev
<c>* ===== 批量缺陷检测推理脚本（HALCON 25.11）=====</c>
```
**第6行**：注释，说明这是批量推理脚本。

```hdev
<c>* 路径由 Python 后端动态替换</c>
```
**第7行**：注释，说明使用占位符机制。

```hdev
<l>ModelFile := '{MODEL_PATH}'</l>
```
**第8行**：模型文件路径占位符。

```hdev
<l>PreprocessFile := '{HDICT_PATH}'</l>
```
**第9行**：预处理参数文件路径占位符。

```hdev
<l>OutputFile := '{OUTPUT_PATH}'</l>
```
**第10行**：总结果 JSON 文件的输出路径占位符。

```hdev
<l>ProgressFile := '{PROGRESS_PATH}'</l>
```
**第11行**：**新增！** 进度文件路径占位符。Python 后端会替换为实际路径，脚本在处理每张图片后都会更新此文件，供前端读取进度。

```hdev
<c>* --- 图片列表 ---</c>
```
**第12行**：注释。

```hdev
{IMAGE_LIST}
```
**第13行**：**占位符！** 这里不是 HALCON 代码，而是一个模板占位符。Python 后端会将其替换为类似以下的实际代码：
```hdev
<l>ImageList := ['image1.jpg', 'image2.jpg', 'image3.jpg']</l>
```
即生成一个包含所有待处理图片路径的 HALCON 元组（tuple）。

```hdev
<c>* --- 加载预处理参数 ---</c>
```
**第14行**：注释。

```hdev
<l>read_dict (PreprocessFile, [], [], DLPreprocessParam)</l>
```
**第15行**：同上，读取预处理参数。

```hdev
<c>* --- 加载模型（只加载一次）---</c>
```
**第16行**：关键注释！强调模型**只加载一次**，然后在循环中复用。这是批量推理的核心优化点，避免每张图片都重复加载模型带来的巨大开销。

```hdev
<l>read_dl_model (ModelFile, DLModelHandle)</l>
```
**第17行**：读取模型，获取句柄。

```hdev
<l>NumClasses := 3</l>
```
**第18行**：**硬编码**类别数量为 3。这里假设缺陷检测任务是三分类（例如：正常、缺陷A、缺陷B）。也可以由 Python 后端替换为占位符。

```hdev
<c>* --- 初始化结果 ---</c>
```
**第19行**：注释。

```hdev
<l>NumImages := |ImageList|</l>
```
**第20行**：计算 `ImageList` 元组的长度，即待处理的图片总数，赋值给 `NumImages`。

```hdev
<l>Results := '['</l>
```
**第21行**：初始化结果字符串 `Results`，以 `[` 开头，表示这是一个 JSON 数组，最终将包含所有图片的推理结果。

```hdev
<l>Count := 0</l>
```
**第22行**：初始化计数器 `Count` 为 0，用于记录已处理的图片数量，并写入进度文件。

```hdev
<c>* --- 循环处理每张图片 ---</c>
```
**第23行**：注释，标记批量处理循环的开始。

```hdev
<l>for I := 0 to NumImages - 1 by 1</l>
```
**第24行**：`for` 循环，遍历 `ImageList` 中的每张图片。`I` 是图片索引。

```hdev
<c>    * 读取图片</c>
```
**第25行**：循环内注释，标记读取图片步骤。

```hdev
<l>    read_image (Image, ImageList[I])</l>
```
**第26行**：使用当前索引 `I` 从 `ImageList` 中取出图片路径，调用 `read_image` 读取图片到 `Image`。

```hdev
<c>    * 推理</c>
```
**第27行**：注释。

```hdev
<l>    gen_dl_samples_from_images (Image, DLSampleBatch)</l>
```
**第28行**：生成深度学习样本。

```hdev
<l>    preprocess_dl_samples (DLSampleBatch, DLPreprocessParam)</l>
```
**第29行**：预处理样本。

```hdev
<l>    apply_dl_model (DLModelHandle, DLSampleBatch, [], DLResultBatch)</l>
```
**第30行**：执行推理。注意：`DLModelHandle` 在循环外部加载，这里直接复用，效率高。

```hdev
<c>    * 提取结果：classification_confidences 和 classification_class_names 都按降序排列，[0] 即 top1</c>
```
**第31行**：注释，说明结果已按降序排列，与 `predict_template.hdev` 一致。

```hdev
<l>    AllConfidences := DLResultBatch.classification_confidences</l>
```
**第32行**：提取所有置信度。

```hdev
<l>    AllClassNames := DLResultBatch.classification_class_names</l>
```
**第33行**：提取所有类别名称。

```hdev
<l>    TopClass := AllClassNames[0]</l>
```
**第34行**：取 Top-1 类别。

```hdev
<l>    TopConfidence := AllConfidences[0]</l>
```
**第35行**：取 Top-1 置信度。

```hdev
<c>    * 构建 JSON</c>
```
**第36行**：注释。

```hdev
<l>    ItemJSON := '{"filename":"' + ImageList[I] + '","top_class":"' + TopClass + '","top_confidence":' + TopConfidence + ',"classes":['</l>
```
**第37行**：为**当前图片**构建 JSON 对象字符串的开头。与单图版本不同的是，这里额外包含了 `filename` 字段，标识这是哪张图片的结果。

```hdev
<l>    for J := 0 to NumClasses - 1 by 1</l>
```
**第38行**：内层 `for` 循环，遍历所有类别，为当前图片构建 `classes` 数组。

```hdev
<l>        ItemJSON := ItemJSON + '{"class":"' + AllClassNames[J] + '","confidence":' + AllConfidences[J] + '}'</l>
```
**第39行**：追加每个类别的 JSON 对象。

```hdev
<l>        if (J &lt; NumClasses - 1)</l>
```
**第40行**：判断是否最后一个类别。

```hdev
<l>            ItemJSON := ItemJSON + ','</l>
```
**第41行**：非最后一个追加逗号。

```hdev
<l>        endif</l>
```
**第42行**：结束 if。

```hdev
<l>    endfor</l>
```
**第43行**：结束内层 for。

```hdev
<l>    ItemJSON := ItemJSON + ']}'</l>
```
**第44行**：闭合当前图片的 JSON 对象。

```hdev
<l>    if (I &gt; 0)</l>
```
**第45行**：判断当前图片是否不是第一张（`I > 0`）。如果是第一张，前面不需要逗号；否则需要追加逗号以分隔 JSON 数组中的对象。

```hdev
<l>        Results := Results + ','</l>
```
**第46行**：非第一张图片时，在 `Results` 末尾追加逗号。

```hdev
<l>    endif</l>
```
**第47行**：结束 if。

```hdev
<l>    Results := Results + ItemJSON</l>
```
**第48行**：将当前图片的 JSON 结果 `ItemJSON` 追加到总结果字符串 `Results` 中。

```hdev
<c>    * 更新进度文件（每张图片处理后都写入）</c>
```
**第49行**：注释，说明接下来是更新进度文件的逻辑。

```hdev
<l>    Count := Count + 1</l>
```
**第50行**：计数器加 1，表示已完成一张图片的处理。

```hdev
<l>    open_file (ProgressFile, 'output', ProgHandle)</l>
```
**第51行**：使用 `open_file` 以输出模式打开进度文件，返回句柄 `ProgHandle`。**注意**：每次处理完一张图片都会重新打开文件（覆盖写入），这样外部程序读取时总能拿到最新进度。

```hdev
<l>    fwrite_string (ProgHandle, Count + ' ' + NumImages)</l>
```
**第52行**：将当前进度写入进度文件，格式为 `"已完成数量 总数量"`（例如 `"5 20"`）。外部程序可以读取此文件并计算百分比进度。

```hdev
<l>    fnew_line (ProgHandle)</l>
```
**第53行**：调用 `fnew_line` 在进度文件中写入换行符，使文件格式更规范。

```hdev
<l>    close_file (ProgHandle)</l>
```
**第54行**：关闭进度文件句柄，确保数据写入磁盘。

```hdev
<l>endfor</l>
```
**第55行**：结束外层 for 循环（所有图片处理完毕）。

```hdev
<c>* --- 写入总结果文件 ---</c>
```
**第56行**：注释。

```hdev
<l>Results := Results + ']'</l>
```
**第57行**：在 `Results` 字符串末尾追加 `]`，闭合 JSON 数组。此时 `Results` 是一个完整的 JSON 数组字符串，包含所有图片的推理结果。

```hdev
<l>open_file (OutputFile, 'output', FileHandle)</l>
```
**第58行**：打开总结果输出文件。

```hdev
<l>fwrite_string (FileHandle, Results)</l>
```
**第59行**：将完整的 JSON 结果数组写入文件。

```hdev
<l>fnew_line (FileHandle)</l>
```
**第60行**：追加换行符。

```hdev
<l>close_file (FileHandle)</l>
```
**第61行**：关闭结果文件。

```hdev
<c>* --- 清理 ---</c>
```
**第62行**：注释。

```hdev
<l>clear_dl_model (DLModelHandle)</l>
```
**第63行**：释放模型资源。

```hdev
</body>
<docu id="main">
<parameters/>
</docu>
</procedure>
</hdevelop>
```
**第64-69行**：文件结束标签。

---

## 5. 三个文件核心差异对比

| 对比维度 | `predict.hdev` | `predict_template.hdev` | `predict_batch_template.hdev` |
|---------|----------------|------------------------|------------------------------|
| **HALCON 版本** | 24.11.1.0 | 25.11.0.0 | 25.11.0.0 |
| **输入方式** | 命令行参数 `argv[1~4]` | Python 模板替换 `{占位符}` | Python 模板替换 `{占位符}` |
| **处理模式** | 单张图片 | 单张图片 | **批量多张图片** |
| **文件操作算子** | `file_open` / `file_write_string` / `file_close` | `open_file` / `fwrite_string` / `close_file` | `open_file` / `fwrite_string` / `close_file` |
| **模型加载次数** | 1 次 | 1 次 | **1 次（循环外加载，循环内复用）** |
| **进度反馈** | 无 | 无 | **有（ProgressFile，每张图片更新）** |
| **结果结构** | 单个 JSON 对象 | 单个 JSON 对象 | **JSON 对象数组**（含 `filename` 字段） |
| `class_names` 获取 | 通过 `get_dl_model_param` 预取 | 从推理结果 `DLResultBatch` 直接取 | 从推理结果 `DLResultBatch` 直接取 |
| 运行方式 | 直接运行 | 需 Python 替换后运行 | 需 Python 替换后运行 |

### 关键设计模式总结

1. **模板机制**：`predict_template.hdev` 和 `predict_batch_template.hdev` 使用了**字符串模板**设计模式。Python 后端读取模板内容，将 `{MODEL_PATH}`、`{IMAGE_LIST}` 等占位符替换为实际值，然后写入临时 `.hdev` 文件，再调用 HALCON 引擎执行。这种方式避免了命令行传参的长度限制和转义问题。

2. **模型复用**：`predict_batch_template.hdev` 将 `read_dl_model` 放在循环外部，是典型的**资源池化/复用**思想。深度学习模型加载通常耗时数秒甚至更长，批量推理时只加载一次可以显著提高效率。

3. **进度通信**：`predict_batch_template.hdev` 通过**文件作为进程间通信（IPC）媒介**。HALCON 脚本将进度写入文件，Python 后端（或前端）定期读取该文件，实现异步进度展示。这是跨语言/跨进程通信的简单有效方案。

4. **结果自包含**：批量版本在每个结果对象中加入了 `filename` 字段，使得结果数组可以独立解析，不需要依赖外部顺序信息，增强了数据的**自描述性**。

---

## 6. HALCON 算子速查表

| 算子名称 | 所属版本 | 功能说明 |
|---------|---------|---------|
| `read_dict` | 通用 | 从 `.hdict` 文件读取字典（键值对集合） |
| `read_dl_model` | 通用 | 从 `.hdl` 文件读取深度学习模型 |
| `get_dl_model_param` | 通用 | 获取深度学习模型的参数（如类别名称、输入尺寸等） |
| `read_image` | 通用 | 读取图片文件到图像变量 |
| `gen_dl_samples_from_images` | 通用 | 将图像转换为深度学习样本格式（DLSample） |
| `preprocess_dl_samples` | 通用 | 对深度学习样本进行预处理（归一化、Resize 等） |
| `apply_dl_model` | 通用 | 使用深度学习模型对样本进行前向推理 |
| `clear_dl_model` | 通用 | 释放深度学习模型占用的内存和资源 |
| `file_open` | 旧版 (24.x) | 打开文件（旧版命名） |
| `file_write_string` | 旧版 (24.x) | 向文件写入字符串（旧版命名） |
| `file_close` | 旧版 (24.x) | 关闭文件句柄（旧版命名） |
| `open_file` | 新版 (25.x) | 打开文件（新版命名） |
| `fwrite_string` | 新版 (25.x) | 向文件写入字符串（新版命名） |
| `close_file` | 新版 (25.x) | 关闭文件句柄（新版命名） |
| `fnew_line` | 新版 (25.x) | 向文件写入换行符 |

---

*文档生成完毕。如有需要补充的算子参数细节或 JSON 输出示例，可以继续扩展。*
