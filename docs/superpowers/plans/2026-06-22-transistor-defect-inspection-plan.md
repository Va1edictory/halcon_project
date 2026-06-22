# Transistor Defect Inspection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a HDevelop-based transistor defect inspection system using a pre-trained DLT GC Anomaly Detection model that performs OK/NG classification, anomaly heatmap visualization, and defect localization with bounding boxes.

**Architecture:** Modular HDevelop design — 1 main entry script + 6 external procedure files (.hdvp). Each procedure has a single responsibility and communicates via typed parameters. Single-image mode for interactive debugging; batch mode for folder processing.

**Tech Stack:** HALCON 20.05+ (HDevelop), DLT (Deep Learning Tool), GC Anomaly Detection model

---

## Global Constraints

- Model: `dlt_project/model_训练-260507-151052_opt.hdl` (GC Anomaly Detection, 256×256×3, range [-127, 128])
- Preprocess params: `dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict`
- Environment: HDevelop IDE, interactive execution
- Anomaly score threshold: hard-coded constant (tunable after initial testing)
- Output: dual-window display (left=original, right=result), batch results saved to `output/`
- Error handling: skip unsupported formats in batch; abort on model load failure
- No git versioning for this phase
- All files created under project root `E:\halcon_project\`

---

## File Structure

```
E:\halcon_project\
├── dlt_project\                          (existing — model + params)
│   ├── model_训练-260507-151052_opt.hdl
│   └── model_训练-260507-151052_opt_dl_preprocess_params.hdict
├── dataset\transistor\                   (existing — test images)
├── source\                               (create — HDevelop source files)
│   ├── main_transistor_inspection.hdev   — entry point, mode selection
│   ├── load_model.hdvp                   — model & preprocessing params loading
│   ├── preprocess.hdvp                   — image preprocessing → DLSample
│   ├── infer.hdvp                        — anomaly detection inference
│   ├── localize.hdvp                     — anomaly regions → bounding boxes
│   ├── visualize.hdvp                    — overlay rendering & dual-window display
│   └── batch_process.hdvp               — folder traversal orchestration
└── output\                               (created at runtime — batch result images)
```

| File | Responsibility |
|------|---------------|
| `load_model.hdvp` | Read .hdl model + .hdict params, return handles |
| `preprocess.hdvp` | Convert raw image to DLSample using training params |
| `infer.hdvp` | Run `apply_dl_model`, extract anomaly_score + anomaly_image, classify OK/NG |
| `localize.hdvp` | Threshold anomaly image → connected regions → filtered bounding boxes |
| `visualize.hdvp` | Pseudo-color heatmap, red bounding boxes, text overlay, dual-window display, optional save |
| `batch_process.hdvp` | List folder → loop → call pipeline → save → summary |
| `main_transistor_inspection.hdev` | Constants + mode switch (single / batch) + orchestration |

---

## Task 1: load_model — Model & Parameter Loading

**Files:**
- Create: `source/load_model.hdvp`

**Interfaces:**
- Produces: `load_model(ModelPath, DictPath, DLModelHandle, DLPreprocessParams)`
  - `ModelPath` (input, string): path to .hdl file
  - `DictPath` (input, string): path to preprocessing params .hdict
  - `DLModelHandle` (output, handle): loaded model handle
  - `DLPreprocessParams` (output, dict): preprocessing parameters

**Test verification:** Run procedure, confirm DLModelHandle is not -1 and DLPreprocessParams contains expected keys.

### Steps

- [ ] **Step 1: Create procedure interface in HDevelop**

In HDevelop: Procedure → Create New Procedure. Set:
- Name: `load_model`
- Location: external, saved to `source/load_model.hdvp`
- Parameters:
  - `ModelPath` — input, string
  - `DictPath` — input, string  
  - `DLModelHandle` — output, handle (HANDLE_DL_MODEL)
  - `DLPreprocessParams` — output, dict (DICT_HANDLE)

- [ ] **Step 2: Write procedure body**

```halcon
* load_model — Load DLT model and preprocessing parameters
*
* Input:
*   ModelPath — path to trained .hdl model file
*   DictPath  — path to _dl_preprocess_params.hdict
* Output:
*   DLModelHandle      — handle to the loaded DL model
*   DLPreprocessParams — dict with preprocessing configuration

* Load the deep learning model
read_dl_model (ModelPath, DLModelHandle)

* Load preprocessing parameters dictionary
read_dict (DictPath, [], [], DLPreprocessParams)

* Log what was loaded
get_dict_tuple (DLPreprocessParams, 'model_type', ModelType)
get_dict_tuple (DLPreprocessParams, 'image_width', ImageWidth)
get_dict_tuple (DLPreprocessParams, 'image_height', ImageHeight)

dev_get_window (CurrentWindow)
dev_disp_text ('Model loaded: ' + ModelType + ' | Input: ' + ImageWidth + 'x' + ImageHeight, 'window', 'top', 'left', 'white', [], [])

return ()
```

- [ ] **Step 3: Test in HDevelop**

In a scratch HDevelop window, run:
```halcon
ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'
load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)
dev_disp_text ('DLModelHandle = ' + DLModelHandle, 'window', 'top', 'left', 'white', [], [])
```

Expected: DLModelHandle is a positive integer. ModelType = 'gc_anomaly_detection'. ImageWidth = 256, ImageHeight = 256.

---

## Task 2: preprocess — Image Preprocessing

**Files:**
- Create: `source/preprocess.hdvp`

**Interfaces:**
- Consumes: `DLPreprocessParams` (dict from Task 1)
- Produces: `preprocess(Image, DLPreprocessParams, DLSample)`
  - `Image` (input, image): raw HImage
  - `DLPreprocessParams` (input, dict): from load_model
  - `DLSample` (output, dict): preprocessed sample dict ready for inference

**Test verification:** Preprocess a test image, check DLSample dict contains 'image' key with correct dimensions (256×256).

### Steps

- [ ] **Step 1: Create procedure interface in HDevelop**

Procedure → New:
- Name: `preprocess`
- Location: `source/preprocess.hdvp`
- Parameters:
  - `Image` — input, image
  - `DLPreprocessParams` — input, dict
  - `DLSample` — output, dict

- [ ] **Step 2: Write procedure body**

```halcon
* preprocess — Convert raw image to DLSample using training preprocessing params
*
* Input:
*   Image              — raw input image (HImage)
*   DLPreprocessParams — dict from _dl_preprocess_params.hdict
* Output:
*   DLSample — dict containing preprocessed image ready for apply_dl_model

* Create a new dict and place the image into it
create_dict (DLSample)
set_dict_tuple (DLSample, 'image', Image)

* Apply the same preprocessing used during training
preprocess_dl_model (DLSample, DLPreprocessParams)

return ()
```

- [ ] **Step 3: Test in HDevelop**

```halcon
* Load model first (Task 1)
ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'
load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)

* Read a test image and preprocess
read_image (TestImage, 'E:/halcon_project/dataset/transistor/test/good/000.png')
preprocess (TestImage, DLPreprocessParams, DLSample)

* Verify DLSample contents
get_dict_tuple (DLSample, 'image', PreprocessedImage)
get_image_size (PreprocessedImage, Width, Height)
dev_disp_text ('Preprocessed: ' + Width + 'x' + Height, 'window', 'top', 'left', 'white', [], [])
```

Expected: Width = 256, Height = 256. No errors from `preprocess_dl_model`.

---

## Task 3: infer — Anomaly Detection Inference

**Files:**
- Create: `source/infer.hdvp`

**Interfaces:**
- Consumes: `DLModelHandle` (from Task 1), `DLSample` (from Task 2)
- Produces: `infer(DLModelHandle, DLSample, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)`
  - `DLModelHandle` (input, handle)
  - `DLSample` (input, dict)
  - `AnomalyThreshold` (input, real): score threshold for OK/NG decision
  - `IsNG` (output, integer): 1 if NG, 0 if OK
  - `AnomalyScore` (output, real): overall anomaly score
  - `AnomalyImage` (output, image): anomaly heatmap (real-valued image)

**Test verification:** Run inference on a good image (expect low score, IsNG=0) and a defect image (expect higher score, ideally IsNG=1).

### Steps

- [ ] **Step 1: Create procedure interface in HDevelop**

Procedure → New:
- Name: `infer`
- Location: `source/infer.hdvp`
- Parameters:
  - `DLModelHandle` — input, handle
  - `DLSample` — input, dict
  - `AnomalyThreshold` — input, real
  - `IsNG` — output, integer
  - `AnomalyScore` — output, real
  - `AnomalyImage` — output, image

- [ ] **Step 2: Write procedure body**

```halcon
* infer — Run GC anomaly detection inference and classify
*
* Input:
*   DLModelHandle    — loaded DL model handle
*   DLSample         — preprocessed sample dict
*   AnomalyThreshold — score above which image is classified as NG
* Output:
*   IsNG         — 1 = defective, 0 = OK
*   AnomalyScore — overall anomaly score (real)
*   AnomalyImage — anomaly heatmap (real-valued image)

* Run the model
apply_dl_model (DLModelHandle, DLSample, [], DLResult)

* Extract results from the result dictionary
get_dict_tuple (DLResult, 'anomaly_image', AnomalyImage)
get_dict_tuple (DLResult, 'anomaly_score', AnomalyScore)

* Classify based on threshold
if (AnomalyScore > AnomalyThreshold)
    IsNG := 1
else
    IsNG := 0
endif

return ()
```

- [ ] **Step 3: Test with good image**

```halcon
* Setup
ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'
AnomalyThreshold := 0.5

load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)
read_image (TestImage, 'E:/halcon_project/dataset/transistor/test/good/000.png')
preprocess (TestImage, DLPreprocessParams, DLSample)
infer (DLModelHandle, DLSample, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)
dev_disp_text ('Score: ' + AnomalyScore + ' | IsNG: ' + IsNG, 'window', 'top', 'left', 'white', [], [])
```

Expected: AnomalyScore displayed, AnomalyImage is a valid real-valued image. Note the score value for later threshold tuning.

- [ ] **Step 4: Test with defect image**

```halcon
read_image (TestImageNG, 'E:/halcon_project/dataset/transistor/test/bent_lead/000.png')
preprocess (TestImageNG, DLPreprocessParams, DLSampleNG)
infer (DLModelHandle, DLSampleNG, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)
dev_disp_text ('Score: ' + AnomalyScore + ' | IsNG: ' + IsNG, 'window', 'top', 'left', 'white', [], [])
```

Expected: AnomalyScore should be higher than the good image. If not, the anomaly_threshold may need adjustment.

---

## Task 4: localize — Defect Localization

**Files:**
- Create: `source/localize.hdvp`

**Interfaces:**
- Consumes: `AnomalyImage` (from Task 3)
- Produces: `localize(AnomalyImage, RegionThreshold, MinArea, DefectRegions, BoundingBoxes, NumDefects)`
  - `AnomalyImage` (input, image): anomaly heatmap from infer
  - `RegionThreshold` (input, real): threshold for binarizing anomaly map (0–1 or 0–255 depending on scaling)
  - `MinArea` (input, integer): minimum pixel area to filter noise
  - `DefectRegions` (output, region): connected filtered anomaly regions
  - `BoundingBoxes` (output, region): rectangle1 bounding regions
  - `NumDefects` (output, integer): number of defect regions found

**Test verification:** On a defect image, `NumDefects > 0` and `BoundingBoxes` contains valid rectangle regions.

### Steps

- [ ] **Step 1: Create procedure interface in HDevelop**

Procedure → New:
- Name: `localize`
- Location: `source/localize.hdvp`
- Parameters:
  - `AnomalyImage` — input, image
  - `RegionThreshold` — input, real
  - `MinArea` — input, integer
  - `DefectRegions` — output, region
  - `BoundingBoxes` — output, region
  - `NumDefects` — output, integer

- [ ] **Step 2: Write procedure body**

```halcon
* localize — Extract defect regions and bounding boxes from anomaly heatmap
*
* Input:
*   AnomalyImage     — real-valued anomaly heatmap from infer
*   RegionThreshold  — threshold value for binarization (pixel values > threshold = anomalous)
*   MinArea          — minimum region area in pixels (filters noise speckles)
* Output:
*   DefectRegions — connected, filtered anomaly regions
*   BoundingBoxes — bounding rectangles (rectangle1) for each region
*   NumDefects    — number of defect regions found

* Convert real anomaly image to byte for thresholding
* First scale to [0, 255] range
scale_image_max (AnomalyImage, AnomalyImageScaled)

* Threshold: pixels above RegionThreshold become the anomaly region
* RegionThreshold is specified in byte range [0, 255]
threshold (AnomalyImageScaled, AnomalyRegion, RegionThreshold, 255)

* Separate disconnected regions
connection (AnomalyRegion, ConnectedRegions)

* Filter out noise: keep only regions with area >= MinArea pixels
select_shape (ConnectedRegions, DefectRegions, 'area', 'and', MinArea, 999999)

* Count the remaining defect regions
count_obj (DefectRegions, NumDefects)

* Convert to bounding rectangles
if (NumDefects > 0)
    shape_trans (DefectRegions, BoundingBoxes, 'rectangle1')
endif

return ()
```

- [ ] **Step 3: Test with defect image**

```halcon
* Continue from Task 3 test
RegionThreshold := 30
MinArea := 50

localize (AnomalyImage, RegionThreshold, MinArea, DefectRegions, BoundingBoxes, NumDefects)
dev_disp_text ('Defects found: ' + NumDefects, 'window', 'top', 'left', 'white', [], [])

* Visually verify regions
dev_set_color ('red')
dev_set_draw ('margin')
dev_set_line_width (2)
dev_display (DefectRegions)
```

Expected: Red outlines appear on the defect image at anomaly locations. NumDefects > 0 for defect images.

- [ ] **Step 4: Test with good image to confirm no false detections**

```halcon
* Using the good image from Task 3 step 3
localize (AnomalyImageGood, RegionThreshold, MinArea, DefectRegionsGood, BoundingBoxesGood, NumDefectsGood)
dev_disp_text ('Defects found (good): ' + NumDefectsGood, 'window', 'top', 'left', 'white', [], [])
```

Expected: NumDefectsGood should be 0 (or very small). If many false positives, increase RegionThreshold or MinArea.

---

## Task 5: visualize — Result Visualization

**Files:**
- Create: `source/visualize.hdvp`

**Interfaces:**
- Consumes: `AnomalyImage` (from Task 3), `BoundingBoxes` + `NumDefects` (from Task 4)
- Produces: `visualize(Image, AnomalyImage, IsNG, AnomalyScore, BoundingBoxes, NumDefects, WindowHandleLeft, WindowHandleRight, SavePath)`
  - `Image` (input, image): original input image
  - `AnomalyImage` (input, image): anomaly heatmap
  - `IsNG` (input, integer): OK/NG flag
  - `AnomalyScore` (input, real): anomaly score value
  - `BoundingBoxes` (input, region): defect bounding boxes
  - `NumDefects` (input, integer): defect count
  - `WindowHandleLeft` (input, window): left display window
  - `WindowHandleRight` (input, window): right display window
  - `SavePath` (input, string): file path to save result image, or '' for no save

**Test verification:** Run with a defect image; left window shows raw image, right window shows heatmap + red boxes + text overlay.

### Steps

- [ ] **Step 1: Create procedure interface in HDevelop**

Procedure → New:
- Name: `visualize`
- Location: `source/visualize.hdvp`
- Parameters:
  - `Image` — input, image
  - `AnomalyImage` — input, image
  - `IsNG` — input, integer
  - `AnomalyScore` — input, real
  - `BoundingBoxes` — input, region
  - `NumDefects` — input, integer
  - `WindowHandleLeft` — input, window handle
  - `WindowHandleRight` — input, window handle
  - `SavePath` — input, string

- [ ] **Step 2: Write procedure body**

```halcon
* visualize — Display detection results in dual-window layout
*
* Input:
*   Image            — original input image
*   AnomalyImage     — anomaly heatmap (real-valued)
*   IsNG             — 1 = NG, 0 = OK
*   AnomalyScore     — overall anomaly score
*   BoundingBoxes    — defect bounding rectangles (region)
*   NumDefects       — number of defect regions
*   WindowHandleLeft — handle for left display (original)
*   WindowHandleRight — handle for right display (result overlay)
*   SavePath         — path to save result, '' if no save

* --- Left window: original image ---
dev_set_window (WindowHandleLeft)
dev_display (Image)
dev_disp_text ('Original Image', 'window', 'top', 'left', 'white', 'box', 'false')

* --- Right window: result overlay ---
dev_set_window (WindowHandleRight)

* Display the anomaly image with a color LUT for heatmap effect
dev_set_lut ('hot')
* Convert real anomaly to byte for display
convert_image_type (AnomalyImage, AnomalyImageByte, 'byte')
dev_display (AnomalyImageByte)

* Reset LUT to default for other overlays
dev_set_lut ('default')

* Draw red bounding boxes for each defect region
if (NumDefects > 0)
    dev_set_color ('red')
    dev_set_line_width (2)
    dev_set_draw ('margin')
    dev_display (BoundingBoxes)
endif

* Display result text overlay
if (IsNG = 1)
    ResultText := 'NG'
    TextColor := 'red'
else
    ResultText := 'OK'
    TextColor := 'green'
endif

get_image_size (Image, ImgWidth, ImgHeight)
* Top bar: status + score
dev_set_color (TextColor)
disp_message (WindowHandleRight, ResultText + ' | Score: ' + AnomalyScore$'.4f', 'window', 12, 12, TextColor, 'false')
* Bottom bar: defect count
if (NumDefects > 0)
    disp_message (WindowHandleRight, 'Defects: ' + NumDefects, 'window', ImgHeight - 30, 12, 'red', 'false')
endif

* Optionally save the result image
if (|SavePath| > 0)
    * Dump the right window content to file
    dump_window_image (ResultImage, WindowHandleRight)
    write_image (ResultImage, 'png', 0, SavePath)
endif

return ()
```

- [ ] **Step 3: Test full pipeline with single image**

```halcon
* Full pipeline setup
ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'
AnomalyThreshold := 0.5
RegionThreshold := 30
MinArea := 50

* Open display windows
dev_close_window ()
dev_open_window (0, 0, 512, 512, 'black', WindowHandleLeft)
dev_open_window (0, 520, 512, 512, 'black', WindowHandleRight)

* Load and run
load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)
read_image (TestImage, 'E:/halcon_project/dataset/transistor/test/bent_lead/000.png')
preprocess (TestImage, DLPreprocessParams, DLSample)
infer (DLModelHandle, DLSample, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)
localize (AnomalyImage, RegionThreshold, MinArea, DefectRegions, BoundingBoxes, NumDefects)
visualize (TestImage, AnomalyImage, IsNG, AnomalyScore, BoundingBoxes, NumDefects, WindowHandleLeft, WindowHandleRight, '')

* Validate visually
```

Expected: Two windows open. Left shows original transistor image. Right shows heatmap with red boxes on defect areas and "NG" text.

---

## Task 6: batch_process — Batch Folder Processing

**Files:**
- Create: `source/batch_process.hdvp`

**Interfaces:**
- Consumes: DLModelHandle, DLPreprocessParams (from Task 1), AnomalyThreshold, RegionThreshold, MinArea, all pipeline procedures
- Produces: `batch_process(InputFolder, OutputFolder, DLModelHandle, DLPreprocessParams, AnomalyThreshold, RegionThreshold, MinArea, WindowHandleLeft, WindowHandleRight)`
  - `InputFolder` (input, string): folder containing images to inspect
  - `OutputFolder` (input, string): folder to save result images
  - `DLModelHandle` (input, handle)
  - `DLPreprocessParams` (input, dict)
  - `AnomalyThreshold` (input, real)
  - `RegionThreshold` (input, real)
  - `MinArea` (input, integer)
  - `WindowHandleLeft` (input, window)
  - `WindowHandleRight` (input, window)

**Test verification:** Run on the test/good folder, confirm all images processed and OK results saved. Run on test/bent_lead folder, confirm NG results saved.

### Steps

- [ ] **Step 1: Create procedure interface in HDevelop**

Procedure → New:
- Name: `batch_process`
- Location: `source/batch_process.hdvp`
- Parameters:
  - `InputFolder` — input, string
  - `OutputFolder` — input, string
  - `DLModelHandle` — input, handle
  - `DLPreprocessParams` — input, dict
  - `AnomalyThreshold` — input, real
  - `RegionThreshold` — input, real
  - `MinArea` — input, integer
  - `WindowHandleLeft` — input, window
  - `WindowHandleRight` — input, window

- [ ] **Step 2: Write procedure body**

```halcon
* batch_process — Process all images in a folder, save results
*
* Input:
*   InputFolder        — directory containing image files
*   OutputFolder       — directory to save result overlays
*   DLModelHandle      — loaded DL model
*   DLPreprocessParams — preprocessing params dict
*   AnomalyThreshold   — OK/NG score threshold
*   RegionThreshold    — anomaly map binarization threshold
*   MinArea            — minimum defect region area
*   WindowHandleLeft   — left display window
*   WindowHandleRight  — right display window

* Ensure output folder exists
file_exists (OutputFolder, FileExists)
if (FileExists = 0)
    make_dir (OutputFolder)
endif

* List all image files in the input folder
list_image_files (InputFolder, 'default', [], ImageFiles)

* Extract just the file count from the tuple
* ImageFiles is a tuple of full paths
tuple_length (ImageFiles, TotalImages)

* Initialize counters
CountOK := 0
CountNG := 0

* Process each image
for Index := 0 to TotalImages - 1 by 1
    * Get current image path
    tuple_select (ImageFiles, Index, CurrentImagePath)
    
    * Read and process with error handling
    try
        read_image (CurrentImage, CurrentImagePath)
        
        * Run the detection pipeline
        preprocess (CurrentImage, DLPreprocessParams, DLSample)
        infer (DLModelHandle, DLSample, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)
        localize (AnomalyImage, RegionThreshold, MinArea, DefectRegions, BoundingBoxes, NumDefects)
        
        * Build save path
        * Extract base filename
        parse_filename (CurrentImagePath, BaseName, Extension, Directory)
        
        if (IsNG = 1)
            Prefix := 'NG'
            CountNG := CountNG + 1
        else
            Prefix := 'OK'
            CountOK := CountOK + 1
        endif
        
        * Zero-padded sequence number
        tuple_string (Index, '04d', SeqStr)
        SavePath := OutputFolder + '/' + Prefix + '_' + SeqStr + '.png'
        
        * Display and save
        visualize (CurrentImage, AnomalyImage, IsNG, AnomalyScore, BoundingBoxes, NumDefects, WindowHandleLeft, WindowHandleRight, SavePath)
        
        * Progress feedback in console
        dev_disp_text ('Processing ' + (Index + 1) + '/' + TotalImages + '  OK:' + CountOK + '  NG:' + CountNG, 'window', 'bottom', 'right', 'white', [], [])
        
        * Brief pause so user can see each result (adjust as needed)
        wait_seconds (0.5)
        
    catch (Exception)
        * Single image failure does not stop batch
        dev_disp_text ('Error on ' + CurrentImagePath + ': ' + Exception, 'window', 'bottom', 'right', 'red', [], [])
        wait_seconds (1)
    endtry
    
endfor

* Final summary
dev_close_window ()
dev_open_window (0, 0, 400, 200, 'black', SummaryWindow)
dev_set_window (SummaryWindow)
SummaryText := 'Batch Complete'
SummaryText[1] := 'Total: ' + TotalImages
SummaryText[2] := 'OK: ' + CountOK
SummaryText[3] := 'NG: ' + CountNG
disp_message (SummaryWindow, SummaryText, 'window', 20, 20, 'white', 'false')

return ()
```

- [ ] **Step 3: Test with good test images**

```halcon
ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'
AnomalyThreshold := 0.5
RegionThreshold := 30
MinArea := 50

load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)

dev_close_window ()
dev_open_window (0, 0, 400, 400, 'black', WindowHandleLeft)
dev_open_window (0, 420, 400, 400, 'black', WindowHandleRight)

batch_process ('E:/halcon_project/dataset/transistor/test/good', 'E:/halcon_project/output', DLModelHandle, DLPreprocessParams, AnomalyThreshold, RegionThreshold, MinArea, WindowHandleLeft, WindowHandleRight)
```

Expected: All good images processed. Summary shows OK count matching total. Result images saved to `output/OK_*.png`.

- [ ] **Step 4: Test with defect images**

```halcon
batch_process ('E:/halcon_project/dataset/transistor/test/bent_lead', 'E:/halcon_project/output', DLModelHandle, DLPreprocessParams, AnomalyThreshold, RegionThreshold, MinArea, WindowHandleLeft, WindowHandleRight)
```

Expected: Defect images classified as NG. Heatmap and boxes on defect areas. Output files prefixed `NG_`.

---

## Task 7: main_transistor_inspection.hdev — Entry Script

**Files:**
- Create: `source/main_transistor_inspection.hdev`

**Interfaces:**
- Consumes: All 6 procedure files
- Produces: Working application with single-image and batch modes

**Test verification:** Run script, select mode, confirm pipeline works end-to-end.

### Steps

- [ ] **Step 1: Create the main script file**

Create `source/main_transistor_inspection.hdev` with the following content:

```halcon
*==========================================================================
* Transistor Defect Inspection System
*==========================================================================
* Uses a pre-trained DLT GC Anomaly Detection model to inspect transistor
* images and detect defects with anomaly heatmap visualization.
*
* Model training date: 2026-05-07
* Input size: 256x256, 3 channels
*==========================================================================

*==========================================================================
* CONFIGURATION — Adjust these values for your setup
*==========================================================================

* Paths (use forward slashes)
ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'
OutputFolder := 'E:/halcon_project/output'

* Detection thresholds — tune these after initial testing
AnomalyThreshold := 0.5      * Score above which image is NG (0.0 to 1.0 typically)
RegionThreshold := 30         * Pixel value threshold for anomaly map binarization [0-255]
MinArea := 50                 * Minimum defect region area in pixels (filters noise)

*==========================================================================
* MAIN — Select mode and run
*==========================================================================

* Close any open windows first
dev_close_window ()
dev_update_off ()

* Load the model (done once for either mode)
try
    load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)
catch (Exception)
    dev_disp_text ('FATAL: Failed to load model: ' + Exception, 'window', 'center', 'center', 'red', [], [])
    stop ()
endtry

* Mode selection dialog
dev_disp_text ('Select Mode: 1 = Single Image | 2 = Batch Folder', 'window', 'center', 'center', 'white', [], [])
Mode := 0

* --- Mode 1: Single Image Inspection ---
if (Mode = 1)
    * Select an image file
    read_image (TestImage, 'E:/halcon_project/dataset/transistor/test/bent_lead/000.png')
    * TODO after testing: replace with dev_get_dialog or hardcoded path
    
    * Create display windows
    dev_close_window ()
    get_image_size (TestImage, ImgWidth, ImgHeight)
    * Scale display to 512x512 while keeping aspect
    DispSize := 512
    dev_open_window (0, 0, DispSize, DispSize, 'black', WindowHandleLeft)
    dev_open_window (0, DispSize + 10, DispSize, DispSize, 'black', WindowHandleRight)
    dev_set_part (0, 0, ImgHeight - 1, ImgWidth - 1)
    
    * Run pipeline
    preprocess (TestImage, DLPreprocessParams, DLSample)
    infer (DLModelHandle, DLSample, AnomalyThreshold, IsNG, AnomalyScore, AnomalyImage)
    localize (AnomalyImage, RegionThreshold, MinArea, DefectRegions, BoundingBoxes, NumDefects)
    visualize (TestImage, AnomalyImage, IsNG, AnomalyScore, BoundingBoxes, NumDefects, WindowHandleLeft, WindowHandleRight, '')
    
    * Keep windows open
    dev_disp_text ('Press F5 to continue...', 'window', 'bottom', 'right', 'white', [], [])
    stop ()

* --- Mode 2: Batch Folder Processing ---
elseif (Mode = 2)
    * Select input folder containing images
    InputFolder := 'E:/halcon_project/dataset/transistor/test/good'
    * TODO after testing: replace with dev_get_dialog or hardcoded path
    
    * Create display windows (smaller for batch mode)
    dev_close_window ()
    dev_open_window (0, 0, 400, 400, 'black', WindowHandleLeft)
    dev_open_window (0, 420, 400, 400, 'black', WindowHandleRight)
    
    * Run batch
    batch_process (InputFolder, OutputFolder, DLModelHandle, DLPreprocessParams, AnomalyThreshold, RegionThreshold, MinArea, WindowHandleLeft, WindowHandleRight)
    
    dev_disp_text ('Batch complete. Results saved to ' + OutputFolder, 'window', 'bottom', 'right', 'white', [], [])
    stop ()

* --- Invalid mode ---
else
    dev_disp_text ('Invalid mode selected. Set Mode := 1 or Mode := 2 and re-run.', 'window', 'center', 'center', 'red', [], [])
    stop ()
endif
```

- [ ] **Step 2: Test Mode 1 (single image)**

Set `Mode := 1` in the script. Run (F5). Verify:
- Left window: original image
- Right window: heatmap overlay with red boxes
- Text overlay showing OK/NG and score

- [ ] **Step 3: Test Mode 2 (batch)**

Set `Mode := 2` in the script. Run (F5). Verify:
- All images processed
- Result images saved to `E:/halcon_project/output/`
- Summary window shown at end

---

## Task 8: Threshold Tuning — Find Optimal Parameters

**Files:**
- Create: `source/tune_thresholds.hdev` (diagnostic script for finding best threshold values)

**Interfaces:** None — standalone tuning tool.

**Test verification:** Run on a mix of good and defect images; find threshold values that separate OK/NG cleanly.

### Steps

- [ ] **Step 1: Create tuning script**

Create `source/tune_thresholds.hdev`:

```halcon
*==========================================================================
* Threshold Tuning Script
*==========================================================================
* Run on samples from each class to find optimal AnomalyThreshold,
* RegionThreshold, and MinArea values.
*==========================================================================

ModelPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt.hdl'
DictPath := 'E:/halcon_project/dlt_project/model_训练-260507-151052_opt_dl_preprocess_params.hdict'

load_model (ModelPath, DictPath, DLModelHandle, DLPreprocessParams)

* Test images — one from each category
ImagePaths := []
ImagePaths[0] := 'E:/halcon_project/dataset/transistor/test/good/000.png'
ImagePaths[1] := 'E:/halcon_project/dataset/transistor/test/bent_lead/000.png'
ImagePaths[2] := 'E:/halcon_project/dataset/transistor/test/cut_lead/000.png'
ImagePaths[3] := 'E:/halcon_project/dataset/transistor/test/damaged_case/000.png'
ImagePaths[4] := 'E:/halcon_project/dataset/transistor/test/misplaced/000.png'

tuple_length (ImagePaths, NumImages)

* Header for results table
ResultHeader := 'Image | AnomalyScore | MaxPixelValue | SuggestedRegionThr'
ResultLines := []

for i := 0 to NumImages - 1 by 1
    read_image (Image, ImagePaths[i])
    preprocess (Image, DLPreprocessParams, DLSample)
    
    * Inference
    apply_dl_model (DLModelHandle, DLSample, [], DLResult)
    get_dict_tuple (DLResult, 'anomaly_score', AnomalyScore)
    get_dict_tuple (DLResult, 'anomaly_image', AnomalyImage)
    
    * Analyze anomaly image values
    min_max_gray (AnomalyImage, AnomalyImage, 0, MinVal, MaxVal, Range)
    intensity (AnomalyImage, AnomalyImage, MeanVal, Deviation)
    
    * Suggested region threshold = mean + 2*deviation
    SuggestedThr := MeanVal + 2 * Deviation
    
    * Format result
    parse_filename (ImagePaths[i], BaseName, Extension, Directory)
    ResultLines[i] := BaseName + ' | ' + AnomalyScore$'.4f' + ' | ' + MaxVal$'.2f' + ' | ' + SuggestedThr$'.2f'
endfor

* Display results
dev_close_window ()
dev_open_window (0, 0, 700, 400, 'black', ResultWindow)
dev_set_window (ResultWindow)

disp_message (ResultWindow, '=== Threshold Tuning Results ===', 'window', 12, 12, 'white', 'false')
for i := 0 to |ResultLines| - 1 by 1
    disp_message (ResultWindow, ResultLines[i], 'window', 30 + i * 18, 12, 'cyan', 'false')
endfor

disp_message (ResultWindow, '', 'window', 30 + |ResultLines| * 18 + 10, 12, 'white', 'false')
disp_message (ResultWindow, 'Set AnomalyThreshold between good max and defect min score.', 'window', 30 + |ResultLines| * 18 + 28, 12, 'yellow', 'false')
disp_message (ResultWindow, 'Set RegionThreshold to ~ SuggestedRegionThr of worst defect.', 'window', 30 + |ResultLines| * 18 + 46, 12, 'yellow', 'false')

stop ()
```

- [ ] **Step 2: Run tuning script and record results**

Run `source/tune_thresholds.hdev`. Note:
- The anomaly scores for good vs. defect images
- Suggested RegionThreshold values
- Update `AnomalyThreshold` and `RegionThreshold` in `main_transistor_inspection.hdev`

- [ ] **Step 3: Update main script with tuned thresholds**

After determining good threshold values, update the constants at the top of `main_transistor_inspection.hdev`.

---

## Integration Test

After all tasks are complete, run a full end-to-end test:

- [ ] **Test 1** — Single good image → displays "OK", no red boxes, low anomaly score
- [ ] **Test 2** — Single defect image → displays "NG", red boxes on defect, higher score
- [ ] **Test 3** — Batch on `test/good/` → all OK, files saved to output/
- [ ] **Test 4** — Batch on `test/bent_lead/` → all NG, boxes on bent leads
- [ ] **Test 5** — Batch on mixed folder (good + defect) → correct classification per image
- [ ] **Test 6** — Batch on `test/cut_lead/`, `test/damaged_case/`, `test/misplaced/` → all NG
