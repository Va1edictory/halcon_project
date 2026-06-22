# Transistor Defect Inspection System — Design Spec

**Date:** 2026-06-22  
**Model:** DLT Anomaly Detection (`model_训练-260507-151052_opt.hdl`)  
**Dataset:** MVTec Anomaly Detection — Transistor  

## 1. Overview

A HDevelop-based interactive inspection system that uses a pre-trained DLT anomaly detection model to inspect transistor images and detect defects. The system provides anomaly heatmap visualization and defect localization with bounding boxes.

## 2. Architecture

Modular HDevelop design with one main entry script and six procedure files:

```
main_transistor_inspection.hdev    — entry point, mode selection
├── load_model.hdvp                — model & preprocessing params loading
├── preprocess.hdvp                — image preprocessing (DLSample)
├── infer.hdvp                     — anomaly detection inference
├── localize.hdvp                  — anomaly region → bounding boxes
├── visualize.hdvp                 — overlay rendering & display
└── batch_process.hdvp             — folder traversal & orchestration
```

### Data Flow

```
Image → preprocess → DLSample → infer → anomaly_map + anomaly_score
                                         ↓
                                    localize → defect regions + bounding boxes
                                                  ↓
               Original image + anomaly_map + boxes → visualize → display + save
```

## 3. Components

### 3.1 load_model

- Read the `.hdl` model file via `read_dl_model`
- Read the accompanying `_dl_preprocess_params.hdict` via `read_dict`
- Return: model handle + preprocess params dict

### 3.2 preprocess

- Extract preprocessing parameters from the dict (image dimensions, gray value range, etc.)
- Call `preprocess_dl_model` on the input image
- Return: DLSample object ready for inference

### 3.3 infer

- Call `apply_dl_model` with the DLSample and model handle
- Extract `anomaly_score` and `anomaly_map` from the DLResult dict
- Compare anomaly_score against a fixed threshold → OK (score ≤ threshold) or NG (score > threshold)
- Return: OK/NG decision, anomaly_score, anomaly_map

### 3.4 localize

- Apply `threshold` to the anomaly_map to create a binary region of anomalous pixels
- Apply `connection` to separate disconnected anomaly regions
- Apply `select_shape` (min_area filter) to remove noise speckles
- Apply `shape_trans` ('rectangle1') to get bounding rectangles
- Return: defect regions, bounding box coordinates, defect count

### 3.5 visualize

- Map anomaly_map to pseudo-color via `scale_image_max` + LUT
- Blend pseudo-color heatmap onto original image at 50% opacity
- Draw red bounding rectangles from localize output
- Overlay text: OK/NG status, anomaly score, defect region count
- Single-image mode: display in dual HDevelop windows (left=original, right=result), pause for user
- Batch mode: display fixed duration, save result image to output folder
- Result image filename format: `OK_001.png` or `NG_001.png`

### 3.6 batch_process

- Parse all image files (`.png`, `.jpg`, `.bmp`) from input folder via `list_image_files`
- Loop through each image, calling preprocess → infer → localize → visualize
- Save each result image to `output/` subfolder
- Display progress indicator during processing
- Show summary after completion: total count, OK count, NG count
- Skip unsupported file formats with a warning; abort on model load failure

### 3.7 main_transistor_inspection

- Mode A (Single): user selects one image file → full pipeline → display
- Mode B (Batch): user selects input folder → batch_process → summary
- Fixed anomaly threshold defined as a constant at the top of the file

## 4. Error Handling

| Scenario | Behavior |
|----------|----------|
| Model file not found | Error message, exit |
| Preprocess params dict not found | Error message, exit |
| Unsupported image format | Warning, skip file, continue batch |
| Inference fails on a single image | Warning, skip file, continue batch |
| Empty anomaly_map (all zeros) | Treat as OK, no regions drawn |

## 5. File Output

Batch mode saves results to `output/` relative to the script directory:

```
output/
├── OK_0001.png
├── OK_0002.png
├── NG_0003.png
├── ...
```

## 6. Out of Scope

- Model performance evaluation / metrics calculation
- Defect type classification (anomaly detection model does not distinguish defect types)
- GUI application (stays within HDevelop IDE)
- Real-time camera capture
