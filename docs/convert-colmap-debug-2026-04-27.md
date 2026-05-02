# convert.py 与 COLMAP 调试记录

日期：2026-04-27

## 背景

本次调试的转换命令如下：

```bash
conda run -n gaussian-splatting python convert.py -s data/2024_06_04_13_44_39_test/
```

数据集根目录：

```text
data/2024_06_04_13_44_39_test/
```

图片实际存放目录：

```text
data/2024_06_04_13_44_39_test/inputs/
```

图片数量：206 张。

当前使用的 COLMAP 可执行文件：

```text
/usr/local/bin/colmap
COLMAP 3.13.0，启用 CUDA
```

## 问题一：COLMAP GPU 参数名不兼容

最初报错如下：

```text
Failed to parse options - unrecognised option '--SiftExtraction.use_gpu'
ERROR:root:Feature extraction failed with code 256. Exiting.
```

根因是仓库原始 `convert.py` 使用了旧版本 COLMAP 的 GPU 参数：

```text
--SiftExtraction.use_gpu
--SiftMatching.use_gpu
```

当前环境中的 COLMAP 3.13.0 已不再接受这两个参数。通过以下命令查看帮助信息：

```bash
colmap feature_extractor -h
colmap exhaustive_matcher -h
```

确认新版本支持的参数是：

```text
--FeatureExtraction.use_gpu
--FeatureMatching.use_gpu
```

因此在 `convert.py` 中做如下替换：

```diff
- --SiftExtraction.use_gpu
+ --FeatureExtraction.use_gpu

- --SiftMatching.use_gpu
+ --FeatureMatching.use_gpu
```

修复后，COLMAP 不再因为参数解析失败而立即退出。

## 问题二：图片目录名称不匹配

修复 GPU 参数后，出现新的报错：

```text
[option_manager.cc:901] Check failed: ExistsDir(*image_path)
Invalid options provided.
ERROR:root:Feature extraction failed with code 256. Exiting.
```

根因是原始 `convert.py` 默认图片目录为：

```text
<source_path>/input/
```

但当前数据集的图片目录为：

```text
<source_path>/inputs/
```

当执行以下命令时：

```bash
python convert.py -s data/2024_06_04_13_44_39_test/
```

脚本实际传给 COLMAP 的图片路径是：

```text
data/2024_06_04_13_44_39_test/input
```

该目录不存在，所以 COLMAP 报 `ExistsDir(*image_path)` 检查失败。

使用以下命令确认目录结构和图片数量：

```bash
find data/2024_06_04_13_44_39_test -maxdepth 2 -type d | sort
find data/2024_06_04_13_44_39_test/inputs -maxdepth 1 -type f | wc -l
```

图片数量检查结果：

```text
206
```

修复方式是在 `convert.py` 中增加图片目录自动选择逻辑：优先使用 `input/`，不存在时回退到 `inputs/`。

```python
image_path = os.path.join(args.source_path, "input")
if not os.path.isdir(image_path):
    image_path = os.path.join(args.source_path, "inputs")
if not os.path.isdir(image_path):
    logging.error(f"No image directory found. Expected '{args.source_path}/input' or '{args.source_path}/inputs'.")
    exit(1)
```

同时把后续 COLMAP 命令中的硬编码路径：

```python
args.source_path + "/input"
```

改为：

```python
image_path
```

涉及的 COLMAP 阶段包括：

- `feature_extractor`
- `mapper`
- `image_undistorter`

## 运行过程排查

完成上述两处修复后，转换命令不再立即失败，而是进入 COLMAP 的 `mapper` 阶段。

实际运行的 mapper 命令类似如下：

```text
colmap mapper \
  --database_path data/2024_06_04_13_44_39_test//distorted/database.db \
  --image_path data/2024_06_04_13_44_39_test/inputs \
  --output_path data/2024_06_04_13_44_39_test//distorted/sparse \
  --Mapper.ba_global_function_tolerance=0.000001
```

运行过程中看起来长时间没有输出。原因是通过 `conda run` 启动时，COLMAP 输出会被缓冲，通常要等子进程结束后才一次性打印。

用以下命令检查进程状态：

```bash
ps -eo pid,ppid,etime,pcpu,pmem,stat,cmd | rg 'colmap mapper|convert.py|conda run'
```

观察到 `colmap mapper` 持续处于运行状态，并且 CPU 占用较高：

```text
colmap mapper ... Rl ... high CPU usage
```

这说明进程不是卡在文件缺失或数据库锁，而是在正常执行重建计算。

## COLMAP 数据库检查

系统中没有安装 `sqlite3` 命令行工具，因此使用 Python 内置的 `sqlite3` 模块检查 COLMAP 数据库。

检查命令：

```bash
conda run -n gaussian-splatting python -c "import sqlite3; p='data/2024_06_04_13_44_39_test/distorted/database.db'; con=sqlite3.connect(p, timeout=1); cur=con.cursor(); print([(t, cur.execute(f'select count(*) from {t}').fetchone()[0]) for t in ['cameras','images','keypoints','descriptors','matches','two_view_geometries']]); print('keypoints', cur.execute('select min(rows), avg(rows), max(rows) from keypoints').fetchone()); print('verified_pairs', cur.execute('select count(*) from two_view_geometries where rows > 0').fetchone()[0]); print('verified_rows', cur.execute('select min(rows), avg(rows), max(rows) from two_view_geometries where rows > 0').fetchone())"
```

检查结果：

```text
cameras: 1
images: 206
keypoints: 206
descriptors: 206
matches: 21115
two_view_geometries: 21115
keypoints min/avg/max: 8312 / 10806.95 / 17030
verified pairs with rows > 0: 8963
verified rows min/avg/max: 15 / 289.01 / 5135
```

结论：特征提取和特征匹配都已经成功，数据库中有足够的 verified pairs，`mapper` 阶段是在进行真实重建，不是前置步骤失败。

## 最终结果

转换最终成功完成，输出：

```text
Done.
```

关键 COLMAP 日志：

```text
Keeping successful reconstruction
Reconstruction with 206 images and 105634 points
Image undistortion
Writing reconstruction...
Writing configuration...
Writing scripts...
```

生成的关键文件：

```text
data/2024_06_04_13_44_39_test/sparse/0/cameras.bin
data/2024_06_04_13_44_39_test/sparse/0/frames.bin
data/2024_06_04_13_44_39_test/sparse/0/images.bin
data/2024_06_04_13_44_39_test/sparse/0/points3D.bin
data/2024_06_04_13_44_39_test/sparse/0/rigs.bin
data/2024_06_04_13_44_39_test/images/
```

验证 undistorted 图片数量：

```bash
find data/2024_06_04_13_44_39_test/images -maxdepth 1 -type f | wc -l
```

输出：

```text
206
```

## 注意事项

之前曾错误执行过：

```bash
python convert.py -s data/2024_06_04_13_44_39_test/inputs
```

这会把 `inputs/` 当成数据集根目录，并产生一个无用旧目录：

```text
data/2024_06_04_13_44_39_test/inputs/distorted/
```

正确做法是传入数据集根目录：

```bash
conda run -n gaussian-splatting python convert.py -s data/2024_06_04_13_44_39_test/
```

不要把 `inputs/` 目录本身传给 `-s`。

