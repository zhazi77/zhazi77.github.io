---
draft: true
date:
  created: 2025-04-05
categories:
  - Misc
tags:
  - OpenPCDet
  - 3D Detection
authors:
  - zhazi
---

# 记录：复现 DetZero 并在自定义数据集上测试

论文需要一个新的目标检测方法做为基线，因此开始复现最新工作并测试其效果。测试方法为：在 Waymo 数据集的子集上训练，然后在自定义数据集上测试效果。这里记录 DetZero 的测试过程。

## 搭建环境

```srl
git https://github.com/PJLab-ADG/DetZero.git

conda create --name detzero python=3.8
conda activate detzero

pip install torch==1.10.0+cu111 torchvision==0.11.0+cu111 torchaudio==0.10.0 -f https://download.pytorch.org/whl/torch_stable.html
conda install cmake
pip install spconv-cu111
pip install waymo-open-dataset-tf-2-5-0

cd DetZero && pip install -r requirements.txt
cd ..

cd DetZero/utils && python setup.py develop
cd ../..

cd DetZero/detection && python setup.py develop
cd ../..
```

## 数据预处理

仓库文档有误，数据集要放在根目录（DetZero）下，配置文件的名称不是 `waymo_one_sweep.yaml` 是 `waymo_1sweep.yaml`
```srl
ln -s /path/to/waymo/raw_data DetZero/detection/data/waymo/raw_data

cd detection
python -m detzero_det.datasets.waymo.waymo_preprocess --cfg_file tools/cfgs/det_dataset_cfgs/waymo_1sweep.yaml --func create_waymo_infos
python -m detzero_det.datasets.waymo.waymo_preprocess --cfg_file tools/cfgs/det_dataset_cfgs/waymo_1sweep.yaml --func create_waymo_database
```

