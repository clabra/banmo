# BANMo 
#### [[Webpage]](https://banmo-www.github.io/) [[Arxiv]](https://arxiv.org/abs/2112.12761)

## Install
### Build with conda
We provide two versions. 
<details><summary>[A. torch1.10+cu113 (1.4x faster on V100)]</summary>

```
# clone repo
git clone git@github.com:facebookresearch/banmo.git --recursive
cd banmo
# install conda env
conda env create -f misc/banmo-cu113.yml
conda activate banmo-cu113
# install pytorch3d (takes minutes), kmeans-pytorch
pip install -e third_party/pytorch3d
pip install -e third_party/kmeans_pytorch
# install detectron2
python -m pip install detectron2 -f \
  https://dl.fbaipublicfiles.com/detectron2/wheels/cu113/torch1.10/index.html
```
</details>

<details><summary>[B. torch1.7+cu110]</summary>

```
# clone repo
git clone git@github.com:facebookresearch/banmo.git --recursive
cd banmo
# install conda env
conda env create -f misc/banmo.yml
conda activate banmo
# install kmeans-pytorch
pip install -e third_party/kmeans_pytorch
# install detectron2
python -m pip install detectron2 -f \
  https://dl.fbaipublicfiles.com/detectron2/wheels/cu110/torch1.7/index.html
```
</details>

### Data
We provide two ways to obtain data. 
The easiest way is to download and unzip the pre-processed data as follows.
<details><summary>[Download pre-processed data]</summary>

We provide preprocessed data for cat and human.
Download the pre-processed `rgb/mask/flow/densepose` images as follows
```
# (~8G for each)
bash misc/processed/download.sh cat-pikachiu
bash misc/processed/download.sh human-cap
```
</details>

<details><summary>[Download raw videos]</summary>

Download raw videos to `./raw/` folder
```
bash misc/vid/download.sh cat-pikachiu
bash misc/vid/download.sh human-cap
bash misc/vid/download.sh dog-tetres
bash misc/vid/download.sh cat-coco
```
</details>

**To use your own videos, or pre-process raw videos into banmo format, 
please follow the instructions [here](./preprocess).**

### PoseNet weights
<details><summary>[expand]</summary>

Download pre-trained PoseNet weights for human and quadrupeds
```
mkdir -p mesh_material/posenet && cd "$_"
wget $(cat ../../misc/posenet.txt); cd ../../
```
</details>


## Demo
This example shows how to reconstruct a cat from 11 videos and a human from 10 videos.
For more examples, see [here](./scripts/README.md).

<details><summary>Hardware/time for running the demo</summary>

By default, it takes 8 hours on 2 V100 GPUs and 15 hours on 1 V100 GPU.
We provide a [script](./scripts/template-accu.sh) that use gradient accumulation
 to support experiments on fewer GPUs / GPU with lower memory.
</details>

<details><summary>Setting good hyper-parameter for your own videos</summary>

When optimizing your own videos, a rule of thumb is to set 
"num gpus" x "batch size" x "accu steps" ~= num frames (default number 512 suits for cat-pikachiu and human-hap)
</details>

#### 1. Optimization
<details><summary>[cat-pikachiu]</summary>

```
seqname=cat-pikachiu
# To speed up data loading, we store images as lines of pixels). 
# only needs to run it once per sequence and data are stored
python preprocess/img2lines.py --seqname $seqname

# Optimization
bash scripts/template.sh 0,1 $seqname 10001 "no" "no"
# argv[1]: gpu ids separated by comma 
# args[2]: sequence name
# args[3]: port for distributed training
# args[4]: use_human, pass "" for human cse, "no" for quadreped cse
# args[5]: use_symm, pass "" to force x-symmetric shape

# Extract articulated meshes and render
bash scripts/render_mgpu.sh 0 $seqname logdir/$seqname-ft3/params_latest.pth \
        "0 1 2 3 4 5 6 7 8 9 10" 256
# argv[1]: gpu id
# argv[2]: sequence name
# argv[3]: weights path
# argv[4]: video id separated by space
# argv[5]: resolution of running marching cubes (256 by default)
```
</details>

<details><summary>[human-cap]</summary>

```
seqname=human-cap
python preprocess/img2lines.py --seqname $seqname
bash scripts/template.sh 0,1 $seqname 10001 "" ""
bash scripts/render_mgpu.sh 0 $seqname logdir/$seqname-ft3/params_latest.pth \
        "0 1 2 3 4 5 6 7 8 9" 256
```
</details>

#### 2. Visualization tools
<details><summary>[Tensorboard]</summary>

```
# You may need to set up ssh tunneling to view the tensorboard monitor locally.
screen -dmS "tensorboard" bash -c "tensorboard --logdir=logdir --bind_all"
```
</details>

<details><summary>[Root pose, rest mesh, bones]</summary>

To draw root pose trajectories (+rest shape) over epochs
```
# logdir
logdir=logdir/$seqname-e120-b256-init/
# first_idx, last_idx specifies what frames to be drawn
python scripts/visualize/render_root.py --testdir $logdir --first_idx 0 --last_idx 120
```
Find the output at `$logdir/mesh-cam.gif`. 
During optimization, the rest mesh and bones at each epoch are saved at `$logdir/*rest.obj`.
</details>

<details><summary>[Correspondence]</summary>

To visualize 2d-2d and 2d-3d matchings of the latest epoch weights
```
# 2d matches between frame 0 and 100 via 2d->feature matching->3d->geometric warp->2d
bash scripts/render_match.sh $logdir/params_latest.pth "0 100" "--render_size 128"
```
2d-2d matches will be saved to `tmp/match_%03d.jpg`. 
2d-3d feature matches of frame 0 will be saved to `tmp/match_line_pred.obj`.
2d-3d geometric warps of frame 0 will be saved to `tmp/match_line_exp.obj`.
near-plane frame 0 will be saved to `tmp/match_plane.obj`.
</details>

<details><summary>[Render novel views]</summary>

Render novel views at the canonical camera coordinate
```
bash scripts/render_nvs.sh 0 adult7 logdir/adult7-ft3/params_latest.pth 0 0
# argv[1]: gpu id
# argv[2]: sequence name
# argv[3]: path to the weights
# argv[4]: video id used for pose traj
# argv[5]: video id used for root traj
```
</details>

### Common issues
<details><summary>[expand]</summary>

* Q: pyrender reports `ImportError: Library "GLU" not found.`
    * install `sudo apt install freeglut3-dev`
* Q: ffmpeg reports `libopenh264.so.5` not fund
    * install ffmpeg `sudo apt-get install ffmpeg` and remove ~/anaconda/envs/banmo/bin/ffmpeg
</details>

### Note on arguments
<details><summary>[expand]</summary>

- use `--use_human` for human reconstruction, otherwise it assumes quadruped animals
- use `--full_mesh` at mesh extraction time to extract a complete surface (disable visibility check)
- use `--queryfw` at mesh extraction time to extract forward articulated meshes, which only needs to run marching cubes once.
</details>

### Acknowledgement
<details><summary>[expand]</summary>

Volume rendering code is borrowed from [Nerf_pl](https://github.com/kwea123/nerf_pl).
Flow estimation code is adapted from [VCN-robust](https://github.com/gengshan-y/rigidmask).
Other external repos:
- [Detectron2](https://github.com/facebookresearch/detectron2) (modified)
- [SoftRas](https://github.com/ShichenLiu/SoftRas) (modified, for synthetic data generation)
- [Chamfer3D](https://github.com/ThibaultGROUEIX/ChamferDistancePytorch) (for evaluation)
</details>

### License
<details><summary>[expand]</summary>

- [CC-BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/legalcode). 
See the [LICENSE](LICENSE) file. 
</details>
