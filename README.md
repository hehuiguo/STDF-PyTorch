# STDF-PyTorch

- [STDF-PyTorch](#stdf-pytorch)
  - [0. Background](#0-background)
  - [1. Pre-request](#1-pre-request)
    - [1.1. Environment](#11-environment)
    - [1.2. DCNv2](#12-dcnv2)
    - [1.3. Dataset (MFQEv2)](#13-dataset-mfqev2)
  - [2. Train](#2-train)
  - [3. Test](#3-test)
  - [4. Results](#4-results)
  - [5. Q&A](#5-qa)
  - [6. License](#6-license)
  - [7. See more](#7-see-more)

## 0. Background

PyTorch implementation of [Spatio-Temporal Deformable Convolution for Compressed Video Quality Enhancement](https://www.aiide.org/ojs/index.php/AAAI/article/view/6697) (AAAI 2020).

- A simple yet effective video quality enhancement network.
- Adopt feature alignment by multi-frame deformable convolutions, instead of motion estimation and motion compensation.

```tex
@inproceedings{STDF,
  title={Spatio-Temporal Deformable Convolution for Compressed Video Quality Enhancement},
  author={Deng, Jianing and Wang, Li and Pu, Shiliang and Zhuo, Cheng},
  booktitle={Proceedings of the AAAI Conference on Artificial Intelligence},
  volume={34},
  number={07},
  pages={10696--10703},
  year={2020}
}
```
 
Special thanks to:

- Jianing Deng (邓家宁, the author of STDF): network structure and training details.
- [BasicSR](https://github.com/xinntao/BasicSR): useful tools and functions.

Note: The dataset and training method are different from those in the original paper.

If you have any advice or question, feel free to contact: ryanxingql@gmail.com.

## 1. Pre-request

### 1.1. Environment

- Ubuntu 20.04/18.04
- CUDA 10.1
- PyTorch 1.16

Suppose that you have installed CUDA 10.1, then:

```bash
$ conda create -n stdf python=3.7 -y
$ conda activate stdf
$ python -m pip install torch==1.6.0+cu101 torchvision==0.7.0+cu101 -f https://download.pytorch.org/whl/torch_stable.html
$ python -m pip install tqdm scikit-image lmdb opencv-python
```

### 1.2. DCNv2

**Build DCNv2.**

```bash
$ cd ops/dcn
$ bash build.sh
```

**[Optional] Simply check if DCNv2 works.**

```bash
$ python simple_check.py
```

> - The DCNv2 source files here is different from the [open-sourced version](https://github.com/chengdazhi/Deformable-Convolution-V2-PyTorch) due to incompatibility. [[issue]](https://github.com/open-mmlab/mmediting/issues/84#issuecomment-644974315)
> - If you move the repository into a new path, you must re-build DCNv2 under the new path.

### 1.3. Dataset (MFQEv2)

**Download the raw dataset.**

[[DropBox]](https://www.dropbox.com/sh/d04222pwk36n05b/AAC9SJ1QypPt79MVUZMosLk5a?dl=0)

(Chinese researchers: [[百度网盘]]((https://pan.baidu.com/s/1WL1WxFeRtwOh3HevPqeuTw)), 提取码: xuia)

> MFQEv2 dataset includes 108 lossless YUV sequences for training, and 18 test sequences recommended by ITU-T.

**Compress both training and test sequences by HM16.5 at LDP mode, QP=37.**

We also provide video compression tools in the above links.

Take 18 test sequences as examples.

1. Unzip the `test_18.zip` into `test_18/raw` folder. It contains 18 raw videos.
2. Generate video config files: `python main_generate_video_cfg.py`. Args:
   - `ubuntu` or `windows` (line 6)
3. Generate `.bat` or `.sh` files: `python main_generate_bat.py`. Args:
   - QPs to be encoded, e.g., `[22,27,32,37,42]` (line 7)
   - Num of bat files in parallel (line 8)
   - `ubuntu` or `windows` (line 9)
   - `test` or `train` (line 10)
4. Run all `.bat` or `.sh` in `video_compression/bat/test_18`.

Note: On Ubuntu system, first `$ chmod +x TAppEncoderStatic`.

**Place datasets as follows.**

```tex
MFQEv2/
├── train_108/
│   ├── raw/
│   └── HM16.5_LDP/
│       └── QP37/
└── test_18/
    ├── raw/
    └── HM16.5_LDP/
        └── QP37/
```

**Edit `option_R3_mfqev2_4G.yml`.**

Suppose the folder `MFQEv2/` is placed at `/media/x/Database/MFQEv2/`, then you should assign `/media/x/Database/MFQEv2` to `dataset -> train -> root` in YAML.

> - 4G means you will use 4 GPUs for training later. Similarly, you can also edit `option_R3_mfqev2_1G.yml` and `option_R3_mfqev2_2G.yml`.
> - R3 is one of the network structures provided in the paper.

**Generate LMDB to speed up IO during training.**

```bash
$ python create_lmdb_mfqev2.py
```

Now you will get all needed data:

```tex
MFQEv2/
├── train_108/
│   ├── raw/
│   └── HM16.5_LDP/
│       └── QP37/
├── test_18/
│   ├── raw/
│   └── HM16.5_LDP/
│       └── QP37/
├── mfqev2_train_gt.lmdb/
└── mfqev2_train_lq.lmdb/
```

Finally, the MFQEv2 dataset root will be sym-linked to the folder `./data/` automatically.

> So that we and programmes can access MFQEv2 dataset at `./data/` directly.

## 2. Train

See `script.sh`.

## 3. Test

See `script.sh`.

Pre-trained models will be released in Sep.

## 4. Results

Similar to that in the original paper. Results will be released in Sep.

## 5. Q&A

> How we enlarge the dataset?

Following BasicSR, we set `sampling index = target index % dataset len`.

For example, if we have a dataset which volume is 4 and enlargement ratio is 2, then we will sample images at indexes equal 0, 1, 2, 3, 0, 1, 2, 3. Note that at each sampling, we will randomly crop the image. Therefore, the patches cropped at the same image but different times can be different.

Besides, we will shuffle the data loader at the start of each epoch. Enlarging epoch can help reduce the building time.

> Why we set the number of iteration not epoch?

Because we can enlarge the dataset with various ratio, the number of epoch is meaningless for us. However, the number of iteration indicates the number of sampling batches.

> Maybe slow at testing.

We use a data loader with `batch_size=1` to load 7 low-quality frames and 1 ground-truth frame at one time. Although the memory cost is low, some frames will be loaded multiple times. A faster solution is to load all frames (Y channels) of a YUV sequence at one time.

> Other datasets such as Vimeo-90K.

We will release codes and guideline in Sep.

## 6. License

You can **use, redistribute, and adapt** the material for **non-commercial purposes**, as long as you give appropriate credit by **citing our paper** and **indicating any changes** that you've made.

## 7. See more

- [MFQEv2 (TPAMI 2019)](https://github.com/RyanXingQL/MFQEv2.0)
  - A multi-frame quality enhancement approach on compressed videos.
  - The first to use high-quality frames to enhance neighboring low-quality frame, i.e., taking advantage of the quality fluctuation in compressed videos.
  - A database including 108 multi-resolution long raw videos for training, 18 test sequences recommended by ITU-T, and an HEVC compression toolbox. We use it in this demo.

- [RBQE (ECCV 2020)](https://github.com/RyanXingQL/RBQE)
  - The first resource-efficient quality enhancement approach on compressed images.
  - A Tchebichef-moments-based IQA module.