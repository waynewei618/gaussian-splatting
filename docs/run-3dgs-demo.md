# 如何运行 3DGS Demo

本文档基于当前仓库实际状态整理，目标是先把官方 3DGS 的最小流程跑通。

## 1. 当前仓库状态

这个仓库是官方 `graphdeco-inria/gaussian-splatting` 结构，训练入口是：

- `train.py`
- `render.py`
- `convert.py`

当前环境检查结果：

- `ubuntu_dev` 容器已经在运行
- 容器内 `pixi` 可用
- 容器内 `conda` 可用
- 容器内 NVIDIA GPU 可见
- 但当前直接执行 `python train.py` 会报 `ModuleNotFoundError: torch`

这说明容器已经就绪，但项目依赖还没有安装。

## 2. 重要约束

根据仓库约定，任何脚本、`pixi run ...`、`python ...` 都必须通过下面的方式在容器内执行：

```bash
~/enter_ubuntu_dev.sh <command ...>
```

不要在宿主机直接跑 `python`、`pixi run` 或训练脚本。

## 3. 先安装依赖

这个仓库没有 `pixi.toml`，官方依赖定义在 `environment.yml` 中。最直接的做法是先创建 conda 环境：

```bash
~/enter_ubuntu_dev.sh conda env create -f environment.yml
```

创建完成后，建议这样调用：

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python train.py --help
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python render.py --help
```

如果这两条命令都能正常输出帮助信息，说明基础 Python 依赖已经装好。

## 4. 准备数据集

### 4.1 如果你已经有 COLMAP 输出

数据目录至少应满足：

```text
<scene>
├── images
└── sparse/0
    ├── cameras.bin
    ├── images.bin
    └── points3D.bin
```

这是最适合直接跑 demo 的方式。

### 4.2 如果你只有原始图片

先把图片放成下面的结构：

```text
<scene>
└── input
    ├── 0001.jpg
    ├── 0002.jpg
    └── ...
```

然后执行：

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python convert.py \
  -s /workspace/gaussian-splatting/data/<scene> \
  --resize
```

注意：

- 这一步需要容器内可用的 `COLMAP`
- 使用 `--resize` 还需要 `ImageMagick`
- 训练数据必须是去畸变后的 `PINHOLE` 或 `SIMPLE_PINHOLE` 相机模型
- 如果 `cameras.bin` 里是 `SIMPLE_RADIAL`、`RADIAL`、`OPENCV` 等模型，需要先做 undistort

```bash
SCENE=/workspace/gaussian-splatting/data/2024_06_04_13_44_39
FIXED=/workspace/gaussian-splatting/data/2024_06_04_13_44_39_undist

~/enter_ubuntu_dev.sh bash -lc "
set -e
rm -rf \"$FIXED\"
mkdir -p \"$FIXED\"
colmap image_undistorter \
  --image_path \"$SCENE/images\" \
  --input_path \"$SCENE/sparse/0\" \
  --output_path \"$FIXED\" \
  --output_type COLMAP
mkdir -p \"$FIXED/sparse/0\"
find \"$FIXED/sparse\" -maxdepth 1 -type f -exec mv {} \"$FIXED/sparse/0/\" \;
"
```

如果容器里还没装这两个工具，`convert.py` 不能直接跑通。

## 5. 跑最小训练 Demo

建议先跑训练，再跑渲染，不要一上来就折腾 viewer。

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python train.py \
  -s /workspace/gaussian-splatting/data/<scene> \
  -m /workspace/gaussian-splatting/output/demo
```

说明：

- `-s` 指向你的场景目录
- `-m` 指定模型输出目录
- 不写 `-m` 也可以，但输出目录会随机，不利于后续查找

如果显存紧张，可以先补一个更保守的版本：

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python train.py \
  -s /workspace/gaussian-splatting/data/<scene> \
  -m /workspace/gaussian-splatting/output/demo \
  --data_device cpu
```

这样会降低显存占用，但训练速度会慢一些。

## 6. 渲染验证结果

训练完成后，先验证模型能否正常渲染：

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python render.py \
  -m /workspace/gaussian-splatting/output/demo
```

如果模型目录不是这个路径，把 `-m` 改成你的实际输出目录。

这一步能跑通，通常说明：

- 训练流程正常
- 模型输出正常
- CUDA 扩展和渲染链路基本正常

## 7. 如果你要跑实时 Viewer

Viewer 不是最小闭环必需项。建议在 `train.py` 和 `render.py` 都跑通之后再做。

先编译：

```bash
~/enter_ubuntu_dev.sh bash -lc "cd SIBR_viewers && cmake -B build . -DCMAKE_BUILD_TYPE=Release && cmake --build build -j24 --target install"
```

再运行：

```bash
~/enter_ubuntu_dev.sh bash -lc "cd SIBR_viewers/install/bin && ./SIBR_gaussianViewer_app -m /workspace/gaussian-splatting/output/demo"
```

说明：

- Viewer 依赖比训练更多
- Ubuntu 22.04 下通常还需要额外系统库
- 如果只是验证 demo 是否能跑，优先完成训练和渲染即可

## 8. 推荐的最短闭环

如果目标只是“把 3DGS demo 跑起来”，推荐顺序如下：

1. 创建 `gaussian-splatting` conda 环境
2. 准备一个已经是 COLMAP 格式的场景
3. 跑 `train.py`
4. 跑 `render.py`
5. 最后再决定要不要编译 `SIBR_viewers`

这是当前仓库里最稳、最短、最容易定位问题的路径。

## 9. 当前已确认的报错

未进入 conda 环境时，下面两条命令会失败：

```bash
~/enter_ubuntu_dev.sh python train.py --help
~/enter_ubuntu_dev.sh python render.py --help
```

这是因为没有使用 `conda run -n gaussian-splatting`。正确方式是：

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python train.py --help
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python render.py --help
```

## 10. 下一步建议

如果你准备正式开跑，下一步优先执行：

```bash
~/enter_ubuntu_dev.sh conda env create -f environment.yml
```

然后再用：

```bash
~/enter_ubuntu_dev.sh conda run -n gaussian-splatting python train.py --help
```

确认环境是否已经完整可用。
