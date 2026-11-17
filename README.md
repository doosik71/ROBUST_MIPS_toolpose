# ROBUST-MIPS: A Combined Skeletal Pose and Instance Segmentation Dataset for Laparoscopic Surgical Instruments

This repository provides implementation of surgical-tool pose estimation (based on **ROBUST-MIPS**).
Built on top of [MMPose v1.3.0](https://github.com/open-mmlab/mmpose).

## Introduction

Localisation of surgical tools constitutes a foundational building block for computer-assisted interventional technologies. Works in this field typically focus on training deep learning models to perform segmentation tasks. Performance of learning-based approaches is limited by the availability of diverse annotated data. We argue that skeletal pose annotations are a more efficient annotation approach for surgical tools, striking a balance between richness of semantic information and ease of annotation, thus allowing for accelerated growth of available annotated data. To encourage adoption of this annotation style, we present, ROBUST-MIPS, a combined tool pose and tool instance segmentation dataset derived from the existing ROBUST-MIS dataset. Our enriched dataset facilitates the joint study of these two annotation styles and allow head-to-head comparison on various downstream tasks.
To demonstrate the adequacy of pose annotations for surgical tool localisation, we set up a simple benchmark using popular pose estimation methods and observe high-quality results. To ease adoption, together with the dataset, we release our benchmark models and custom tool pose annotation software.

[🔗 Download ROBUST_MIPS_toolpose](https://github.com/cai4cai/ROBUST_MIPS_toolpose)

[🔗 Download tool-pose-annotation-gui](https://github.com/cai4cai/tool-pose-annotation-gui)

[🔗 Download ROBUST_MIPS Dataset](https://doi.org/10.7303/syn64023381)

## Model Configuration

The configuration files for the benchmarked models are located in:  
`ROBUST_MIPS_toolpose/surgicaltool_bm/configs`
Since this project is built upon MMPose (which defaults to human pose estimation), specific overrides are required to adapt it for the ROBUST-MIPS dataset.

### dataset_info

Taking the ROBUST-MIPS dataset as an example. `keypoint_info` defines the keypoints, `skeleton_info` defines the connection relationships between keypoints, and `sigmas` represents the per-keypoint localization uncertainty used for OKS calculation.

### number fo keypoints

set `NUM_KEYPOINTS=4`, and update it in the `model` configuration

### bbox_file

Since the Top-Down paradigm is a two-stage approach, the first stage requires generating bounding box detection results. In this implementation, mmdetection was used to porvided the bbox detection results. The corresponding results should be updated in the `val_dataloader` and `test_dataloader`.

### test_evaluator

Given the unordered nature of the surgical tool endpoints, when evaluating prediction results, we should select the optimal match for the tip pairs (considering both the standard and swapped assignments of Tip1 and Tip2).  This metric is implemented via `swap_coco_metric` (`surgicaltool_bm/custom_src/evaluation/metric/swap_coco_metric.py`), which needs to be imported in the config file using `custom_imports`.

## Customize Dataset

Since our annotation software (tool-pose-annotation-gui) generates a separate JSON annotation file for each image, we need to aggregate these independent, single-image annotation files into three unified annotation files corresponding to the training, validation, and testing sets. The aggregated format should align with COCO.

Importantly, during the generation of these annotation files, we modified the formula for calculating `area`. Instead of storing the standard area, we store $\frac{\mathrm{width}^2 + \mathrm{height}^2}{2}$. This conversion process can be completed using the scripts provided in the `utilities` folder.

## Overview

This project extends the MMPose v1.3.0 framework to support:

- **Custom dataset**: `ROBUST-MIPS` for laparoscopic frames  
- **Custom evaluation metrics**: custom OKS metric parameters `swap_coco_metric`
- **Plug-and-play interface**: uses MMPose’s `tools/train.py` & `tools/test.py`  

## Docker

We provide a `Dockerfile` to lock in your Python/CUDA environment and install all dependencies.

## Prerequisites

Before you begin, ensure you have met the following requirements:

- **Docker image** Build the image based on the Dockerfile
- **MMPose** v1.3.0 installed (fork your copy and install in editable mode)  

## Table of Contents

- [Project Structure](#project-structure)
- [ROBUST-MIPS Dataset](#data)
- [License](#license)

## Project Structure

```text
ROBUST_MIPS_toolpose/
├── surgicaltool_bm/        # Main package for surgical tool pose estimation
│   ├── configs/            # Configuration files
│   ├── custom_src/         # Custom source code and extensions
│   ├── tools/              # Utility scripts and training/testing tools
│   └── setup.py            # Package setup script
│
├── dataset/                # ROBUST-MIPS
├── mmpose/                 # MMPose lib
├── utilities/              # Processing tools for ROBUST-MIPS datasets
│
├── .dockerignore           # Files ignored by Docker build
├── .gitignore              # Files ignored by Git
├── build.sh                # Shell script to build the environment
├── Dockerfile              # Docker build file
└── README.md               # Project documentation (this file)

```

### utilities/reorganizedata.py

The `reorganizedata.py` script restructures the dataset, originally organized in a hierarchical format as `Training/Testing -> Surgery type -> Procedure -> Frame -> Images and annotations`. It consolidates images and JSON annotation files into a simplified format:

- `rename_training/img` and `rename_training/json` for training images and annotations
- `rename_val/img` and `rename_val/json` for validation images and annotations
- `rename_testing/img` and `rename_testing/json` for testing images and annotations

### utilities/data2cocoformat.py

Converts all JSON files from training/val/testing into COCO format JSON files

### utilities/GTvisualization.py

Verifies contents of cocoformat_train/val/test.json and visualizes annotations

## ROBUST-MIPS Dataset

The **ROBUST-MIPS** dataset and our trained model weights are publicly available for download:

[🔗 Download ROBUST-MIPS](https://doi.org/10.7303/syn64023381)

## Data Layout

Your dataset directory should look like this:

```text
dataset/
├── training/
│   ├── img/                      # raw training images
│   └── json/                     # any original per-image metadata
├── val/
│   ├── img/                      # raw validation images
│   └── json/                     # any original per-image metadata
├── testing/
│   ├── img/                      # raw test images
│   └── json/                     # any original per-image metadata
├── converted_detections_val.json   # results of mmdetection model on val set
├── converted_detections_test.json  # results of mmdetection model on testing set
├── cocoformat_train.json         # COCO‐style train annotations
├── cocoformat_val.json           # COCO‐style val annotations
└── cocoformat_test.json          # COCO‐style test annotations
```

## Getting Started (uv, without Docker)

All commands below are run from the repository root (`ROBUST_MIPS_toolpose/`), because the configs use the relative path `data_root='dataset/'`.

### 1. Fixes applied to the released scripts

The following bugs in the original release have been fixed in this repository:

| File                                                            | Problem                                                                                                                                     | Fix                                                                         |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `utilities/reorganizedata.py`                                   | Searched for `**/raw.json`, but the annotation files are named `toolposes.json`, so no JSON files were copied.                              | Search for `raw.png` (images) and `toolposes.json` (annotations).           |
| `utilities/data2cocoformat.py`                                  | `file_name` stored the full path (`dataset/training/img/xxx.png`). MMPose joins it with `data_root + data_prefix`, which gave a wrong path. | Store `os.path.basename(img_file)` as `file_name`.                          |
| `utilities/GTvisualization.py`                                  | Read images from `file_name` directly; val output folder was the absolute path `/dataset/check_valImg`.                                     | Join `file_name` with the split's image folder; use `dataset/check_valImg`. |
| `surgicaltool_bm/configs/rtmpose-l_8xb256-420e_coco-256x192.py` | Typos in `train_dataloader`: `cocoformat_train.json.json`, `traing/img/`.                                                                   | Use `cocoformat_train.json` and `training/img/`.                            |

### 2. Get the MMPose source

Every config inherits from `../../mmpose/configs/_base_/default_runtime.py`, so the MMPose source must be at `./mmpose`:

```bash
git clone -b v1.3.1 https://github.com/open-mmlab/mmpose.git mmpose
```

### 3. Create the environment with uv

Recommended versions: Python 3.10, PyTorch 2.1.2 (CUDA 12.1), MMCV 2.1.0 (prebuilt wheel, so no CUDA compilation is needed).

```bash
uv venv --python 3.10 .venv
source .venv/bin/activate

# PyTorch
uv pip install torch==2.1.2 torchvision==0.16.2 --index-url https://download.pytorch.org/whl/cu121
# numpy 2.x is incompatible with torch 2.1.
# MMPose imports pkg_resources, which was removed from recent setuptools releases.
uv pip install "numpy<2" "setuptools<81"

# OpenMMLab (mmcv prebuilt wheel for torch2.1/cu121)
uv pip install mmengine
uv pip install mmcv==2.1.0 -f https://download.openmmlab.com/mmcv/dist/cu121/torch2.1/index.html
uv pip install "mmdet==3.3.0" "mmpretrain==1.2.0"

# MMPose (editable). --no-deps avoids building `chumpy`, which fails under uv.
uv pip install -e mmpose --no-deps
uv pip install json_tricks munkres scipy matplotlib pillow opencv-python xtcocotools "albumentations==1.3.1" tensorboard

# Custom metric (swap_coco_metric) used by the configs via custom_imports
uv pip install -e surgicaltool_bm --no-deps
```

Check the install:

```bash
python -c "import torch, mmcv, mmpose, mmdet; print(torch.__version__, torch.cuda.is_available(), mmcv.__version__, mmpose.__version__, mmdet.__version__)"
python -c "import evaluation.metric.swap_coco_metric"
```

If `xtcocotools` has to be built from source and fails, install its build dependencies first and turn off build isolation:
`uv pip install cython numpy<2 && uv pip install xtcocotools --no-build-isolation`.

### 4. Prepare the dataset

Download ROBUST-MIPS and place (or symlink) it at `dataset/RobustMIPS`, so that it contains `Training/` and `Testing/Stage_{1,2,3}/`:

```bash
ln -s /path/to/RobustMIPS dataset/RobustMIPS
```

Then reorganize it and build the COCO-style annotation files:

```bash
python utilities/reorganizedata.py     # -> dataset/{training,val,testing}/{img,json}
python utilities/data2cocoformat.py    # -> dataset/cocoformat_{train,val,test}.json
```

Split used: `Training` → train (5983 frames), `Testing/Stage_1` → val (663), `Testing/Stage_2` + `Stage_3` → test (514 + 2880).
The provided `converted_detections_{val,test}.json` refer to image ids `0..662` and `0..3393`, which match the ids assigned by `data2cocoformat.py` (sorted file order). Do not change the file naming, or the ids will no longer match.

Optional sanity check (not needed for training): `python utilities/GTvisualization.py`, which draws the GT on every image into `dataset/check_{train,val,test}Img/`. Add `--skip-existing` to skip images that were already drawn.

### 5. Train

Single GPU:

```bash
python surgicaltool_bm/tools/train.py \
       surgicaltool_bm/configs/rtmpose-m_8xb256-420e_coco-256x192.py
```

Multi-GPU (e.g. 4 GPUs):

```bash
torchrun --nproc_per_node=4 surgicaltool_bm/tools/train.py \
         surgicaltool_bm/configs/rtmpose-m_8xb256-420e_coco-256x192.py \
         --launcher pytorch
```

If training hangs at start-up and then fails with `Watchdog caught collective operation timeout ... OpType=BROADCAST`, GPU peer-to-peer (P2P) transfer is not working on your machine. This is common when GPUs have no NVLink and sit under different PCIe host bridges with IOMMU enabled (check with `nvidia-smi topo -m`). Disable P2P so NCCL routes traffic through host memory:

```bash
NCCL_P2P_DISABLE=1 torchrun --nproc_per_node=4 surgicaltool_bm/tools/train.py \
         surgicaltool_bm/configs/rtmpose-m_8xb256-420e_coco-256x192.py \
         --launcher pytorch
```

Checkpoints and logs are written to `work_dirs/<config name>/`. Add `--resume` to continue an interrupted run.

### 6. Test

Use your own checkpoint or the released weights:

```bash
python surgicaltool_bm/tools/test.py \
       surgicaltool_bm/configs/rtmpose-m_8xb256-420e_coco-256x192.py \
       work_dirs/rtmpose-m_8xb256-420e_coco-256x192/<best_checkpoint>.pth \
       --work-dir work_dirs/test/rtmpose-m_8xb256-420e_coco-256x192 \
       --out work_dirs/test/rtmpose-m_8xb256-420e_coco-256x192/metric_results.json
```

Add `--show-dir outputs` to save visualized predictions.

## Tool

Training model:

```text
cd /workspace/surgicaltool_bm

python /workspace/surgicaltool_bm/tools/train.py \
       /workspace/surgicaltool_bm/configs/rtmpose-l_8xb256-420e_coco-256x192.py \
       --resume
```

Testing:

```text
cd /workspace/surgicaltool_bm

python /workspace/surgicaltool_bm/tools/test.py \
       /workspace/surgicaltool_bm/configs/rtmpose-l_8xb256-420e_coco-256x192.py \
       /workspace/surgicaltool_bm/work_dirs/best_coco_AP_epoch_285.pth \
       --work-dir /workspace/surgicaltool_bm/work_dirs/test/rtmpose-l_8xb256-420e_coco-256x192 \
       --show-dir outputs \
       --out /workspace/surgicaltool_bm/work_dirs/test/rtmpose-l_8xb256-420e_coco-256x192/metric_results.json \
       --cfg-options='model.test_cfg.output_heatmaps=True' \
```
<!-- Should you wish to use or refer to this data set, you must cite the following papers:
1. **Zhe, H.**, **Charlie, B.**, **Gongyu, Z.**, **Huanyu, T.**, **Christos, B.**, **Tom, V.**, *ROBUST-MIPS: A Combined Skeletal Pose Representation and Instance Segmentation Dataset for Laparoscopic Surgical Instruments*, arXiv preprint arXiv:xxxx.xxxx, 2025. Available at: [arXiv link](https://arxiv.org/abs/xxxx.xxxx)
2. **Maier-Hein, L., Wagner, M., Ross, T., Reinke, A., Bodenstedt, S., Full, P. M., ... & Müller-Stich, B. P. (2021)**. *Heidelberg colorectal data set for surgical data science in the sensor operating room*. Scientific data, 8(1), 1-11.
3. **Roß, T., Reinke, A., Full, P. M., Wagner, M., Kenngott, H., Apitz, M., ... & Maier-Hein, L. (2021)**. *Comparative validation of multi-instance instrument segmentation in endoscopy: results of the ROBUST-MIS 2019 challenge*. Medical image analysis, 70, 101920. -->

## Citation

```bibtex
@misc{han2025robustmipscombinedskeletalpose,
      title={ROBUST-MIPS: A Combined Skeletal Pose and Instance Segmentation Dataset for Laparoscopic Surgical Instruments}, 
      author={Zhe Han and Charlie Budd and Gongyu Zhang and Huanyu Tian and Christos Bergeles and Tom Vercauteren},
      year={2025},
      eprint={2508.21096},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2508.21096}, 
}
```

## License

ROBUST-MIPS is realeased unde a Creative Commons Attribution-NonCommercial-ShareAlike (CC BY-NC-SA) license, which means that it will be publicly available for non-commercial usage.
