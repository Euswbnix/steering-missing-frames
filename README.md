# steering-missing-frames

CSC413 course project comparing LSTM, CfC, LTC and Transformer models for steering-angle
prediction from Honda HDD driving video with missing history frames.

`requirements.txt` records the training server environment. PyTorch builds must match the
host platform; GPU training and HDD data are not needed for the current metrics and tracking tests.
With NumPy, PyTorch and pytest installed, run from the repository root:

```bash
python -m pytest -q
```

The HDD data is not included. It is licensed for non-commercial academic use and must be
requested individually from Honda Research Institute (https://usa.honda-ri.com/hdd).
