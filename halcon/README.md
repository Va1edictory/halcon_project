# HALCON Runtime Scripts

This directory contains source-controlled HALCON scripts and procedure files used by the backend.

Tracked source files:

- `predict_loaded.hdvp`: HDevEngine procedure used by the persistent classifier.
- `predict_template.hdev`: single-image HDevelop fallback template.
- `predict_batch_template.hdev`: batch HDevelop fallback template.

Machine-specific configuration is handled outside this directory:

- Set `HALCONROOT` to the HALCON installation path on the current computer.
- Put model files under `backend/models/`.
- Put image datasets under `data/`, or enter an absolute backend-local image folder path in the UI.

