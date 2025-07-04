# [Uni-Fusion: Universal Continuous Mapping](https://jarrome.github.io/Uni-Fusion/)

[Yijun Yuan](https://jarrome.github.io/), [Andreas Nüchter](https://www.informatik.uni-wuerzburg.de/robotics/team/nuechter/)

[Preprint](https://arxiv.org/abs/2303.12678) |  [website](https://jarrome.github.io/Uni-Fusion/)

#### Uni-Fusion is   *<ins>nothing to do with NeRF!</ins>*  
#### It is a Fusion method (only forward and fusion)!

<p align="">
      <img src="assets/encoder.png" align="" width="45%">
      <img src="assets/PLV.png" align="" width="37%">
</p>

*Universal encoder **no need data train** | Picture on the right is voxel grid for mapping*

*Therefore, it supports **any mapping**:*

<p align="">
<img src="assets/cover_2.png" align="" width="50%">
</p>

<!-- TABLE OF CONTENTS -->
<details open="open" style='padding: 10px; border-radius:5px 30px 30px 5px; border-style: solid; border-width: 1px;'>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#env-setting-and-install">Installation</a>
    </li>
    <li>
      <a href="#demo">Demo</a>
    </li>
    <li>
      <a href="#todo">TODO</a>
    </li>
    <li>
      <a href="#citation">Citation</a>
    </li>
    <li>
      <a href="#acknowledgement">Acknowledgement</a>
    </li>
  </ol>
</details>

## Env setting and install
<details>
      <summary> Unfold this for installation </summary>
      
* Create env
```bash
conda create -n unifusion python=3.10
conda activate unifusion

conda install cuda -c nvidia/label/cuda-11.8.0
conda install pytorch==2.1.2 torchvision==0.16.2 torchaudio==2.1.2 pytorch-cuda=11.8 -c pytorch -c nvidia

conda install -c conda-forge gcc_linux-64=11 gxx_linux-64=11

pip install torch-scatter torch-sparse torch-geometric # -f https://data.pyg.org/whl/torch-1.12.1+cu113.html
pip install ninja

```

* install package
```bash
git clone https://github.com/Jarrome/Uni-Fusion.git && cd Uni-Fusion
# install uni package
pip install numpy==1.26.4
python setup.py install
# install cuda function, this may take several minutes, please use `top` or `ps` to check
python uni/ext/__init__.py

pip install numba open3d opencv-python trimesh 
```

* train a uni encoder from nothing in 1 second
```bash
python uni/encoder/uni_encoder_v2.py
```


<details>
<summary> optionally, you can install the [ORB-SLAM2](https://github.com/Jarrome/Uni-Fusion-use-ORB-SLAM2) that we use for tracking</summary>
  
```bash
cd external
git clone https://github.com/Jarrome/Uni-Fusion-use-ORB-SLAM2
cd [this_folder]
# this_folder is the absolute path for the orbslam2
# Add ORB_SLAM2/lib to PYTHONPATH and LD_LIBRARY_PATH environment variables
# I suggest putting this in ~/.bashrc
export PYTHONPATH=$PYTHONPATH:[this_folder]/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:[this_folder]/lib

./build.sh && ./build_python.sh
```
</details>
</details>

## Demo

### 0. Quick try
We provide a toy example to quick try our algorithm.
You can either `python example/toy.py` or code as following:
```python
import torch
import numpy as np

from example.util import get_modules, get_example_data

device = torch.device("cuda", index=0)

# get mapper and tracker
sm, cm, tracker, config = get_modules(device)

# prepare data
colors, depths, customs, calib, poses = get_example_data(device)

for i in [0, 1]:
    # preprocess rgbd to point cloud
    frame_pose = tracker.track_camera(colors[i], depths[i], customs, calib, poses[i], scene = config.sequence_type)
    # transform data
    tracker_pc, tracker_normal, tracker_customs= tracker.last_processed_pc
    opt_depth = frame_pose @ tracker_pc
    opt_normal = frame_pose.rotation @ tracker_normal
    color_pc, color, color_normal = tracker.last_colored_pc
    color_pc = frame_pose @ color_pc
    color_normal = frame_pose.rotation @ color_normal if color_normal is not None else None

    # mapping pc
    sm.integrate_keyframe(opt_depth, opt_normal)
    cm.integrate_keyframe(color_pc, color, color_normal)

# mesh extraction
map_mesh = sm.extract_mesh(config.resolution, int(4e7), max_std=0.15, extract_async=False, interpolate=True)

import open3d as o3d
o3d.io.write_triangle_mesh('example/mesh.ply', map_mesh)

```
You will get a mesh looks like this:

<p align="">
      <img src="assets/toy_result.png" align="" width="89%">
</p>




---
Then

* **All demo can be run with ```python demo.py [config]```**
* **Mesh for color, style, infrad, semantic can be extracted with ```python vis_LIM.py [config]```**
* **Rendering for RGB and Depth image can be extracted with ```python example/render_w_LIM.py [config] [optionally traj with GT poses]```**

### 1. Reconstruction Demo 
```bash
# download replica data
source scripts/download_replica.sh

# with gt pose
python demo.py configs/replica/office0.yaml

# with slam
python demo.py configs/replica/office0_w_slam.yaml
```

Then you can find results in `output/replica/office0` where was specified in the `[config]` file:  
```console
$ ls output/replica/office0 

surface.lim
color.lim  
final_recons.ply  
pred_traj.txt  
```

* *in [scene_w_slam.yaml], we can choose 3 mode*

|Usage| load_gt| slam|
|---|---|---|
|use SLAM track|False|True|
|use SLAM pred pose|True|True|
|use GT pose|True|False|

* *you can set ```vis=True``` for online vis (```False``` by default), which is more Di-Fusion. You can tap keyboard ',' for step and '.' for continue running with GUI*

* *LIM extraction for mesh*
```
python vis_LIM.py configs/replica/office0.yaml
```

will generate a `output/replica/office0/color_recons.ply`

* *LIM rendering given result LIMs*
```
# with gt pose
python example/render_w_lim.py configs/replica/office0.yaml data/replica/office0/traj.txt

# otherwise 
python example/render_w_lim.py configs/replica/office0_w_slam.yaml 
```

This will creat a `render` folder under `output/replica/office0` where was specified in the `[config]` file: 

```console
$ ls output/replica/office0 

surface.lim
color.lim  
final_recons.ply  
pred_traj.txt  
render/ # here contains rendered RGB and Depth images
```


### 2. Custom context Demo

[```office0_custom.yaml```](https://github.com/Jarrome/Uni-Fusion/blob/main/configs/replica/office0_custom.yaml) contains all mapping you need

```bash
# if you need saliency
pip install transparent-background numba
# if you need style
cd external
git clone https://github.com/Jarrome/PyTorch-Multi-Style-Transfer.git
mv PyTorch-Multi-Style-Transfer style_transfer
cd style_transfer/experiments
bash models/download_model.sh
cd ../../../

# run demo
python demo.py configs/replica/office0_custom.yaml


# LIM extraction of custom property shown on mesh
python vis_LIM.py configs/replica/office0_custom.yaml
```


### 3. Open Vocabulary Scene Understanding Demo
This Text-Visual CLIP is from [OpenSeg](https://github.com/tensorflow/tpu/tree/641c1ac6e26ed788327b973582cbfa297d7d31e7/models/official/detection/projects/openseg)
```bash
# install requirements
pip install tensorflow==2.5.0
pip install git+https://github.com/openai/CLIP.git

# download openseg ckpt
# can use `sudo snap install google-cloud-cli --classic` to install gsutil
gsutil cp -r gs://cloud-tpu-checkpoints/detection/projects/openseg/colab/exported_model ./external/openseg/

python demo.py configs/replica/office0_w_clip.yaml

# LIM extraction of semantic shown on mesh
python vis_LIM.py configs/replica/office0_w_clip.yaml
```

### 4. Self-captured data
#### Azure capturing
We provide the script to extract RGB, D and IR from azure.mp4: [azure_process](https://github.com/Jarrome/azure_process).

The captured apartment data stores [here](https://robotik.informatik.uni-wuerzburg.de/telematics/download/appartment2.tgz).

---
## TODO:
- [x] Upload the uni-encoder src (Jan.3)
- [x] Upload the env script (Jan.4)
- [x] Upload the recon. application (By Jan.8)
- [x] Upload the used ORB-SLAM2 support (Jan.8)
- [x] Upload the azure process for RGB,D,IR (Jan.8)
- [x] Upload the seman. application (Jan.14)
- [x] Upload the Custom context demo (Jan.14)
- [x] Toy example for fast essembling Uni-Fusion into custom project
- [x] Extraction of Mesh w properties from Latent Implicit Maps (LIMs) (Jun.26) [Sry for the delay... Yijun just get some free time...]
- [x] Rendering of RGB and Depth images from Latent Implicit Maps (LIMs) (Jun.26)
- [ ] Our current new project [SceneFactory](https://jarrome.github.io/SceneFactory/) has a better option, I plan to replace this ORB-SLAM2 with that option after open-release that work.

---
## Citation
If you find this work interesting, please cite us:
```bibtex
@article{yuan2024uni,
  title={Uni-Fusion: Universal Continuous Mapping},
  author={Yuan, Yijun and N{\"u}chter, Andreas},
  journal={IEEE Transactions on Robotics},
  year={2024},
  publisher={IEEE}
}
```

## Acknowledgement
* This implementation is on top of [DI-Fusion](https://github.com/huangjh-pub/di-fusion).
* We also borrow some dataset code from [NICE-SLAM](https://github.com/cvg/nice-slam).
* We thank the detailed response of questions from Kejie Li, Björn Michele, Songyou Peng and Golnaz Ghiasi.
