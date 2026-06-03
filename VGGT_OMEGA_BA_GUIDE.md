# VGGT-Omega + Bundle Adjustment Integration Guide

## 📋 概述

这个集成脚本支持两种模型的 Bundle Adjustment 优化：

| 特性 | VGGT (原版) | VGGT-Omega |
|------|-----------|-----------|
| **分辨率** | 固定 518x518 | 灵活 (256-512+) |
| **推理速度** | 基准 | ⚡ 更快 |
| **参数编码** | 直接矩阵 | FOV 编码 |
| **BA 支持** | ✅ | ✅ (新!) |
| **模型大小** | 1B | 1B |

---

## 🚀 快速开始

### 1. 安装依赖

```bash
# 基础依赖
pip install -r requirements.txt

# VGGT-Omega (可选)
pip install git+https://github.com/facebookresearch/vggt-omega.git

# BA 优化依赖
pip install pypose pycolmap
```

### 2. 准备数据

```
YOUR_SCENE_DIR/
├── images/
│   ├── img_0.jpg
│   ├── img_1.jpg
│   └── ...
└── (sparse/ 会自动创建)
```

### 3. 运行推理

#### 选项 A: 使用原 VGGT + BA (推荐用于精度)

```bash
python demo_colmap_omega.py \
  --model vggt \
  --scene-dir /path/to/scene \
  --use-ba \
  --implementation bae
```

#### 选项 B: 使用 VGGT-Omega + BA (推荐用于速度)

```bash
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint /path/to/vggt_omega_1b_512.pt \
  --image-resolution 512 \
  --scene-dir /path/to/scene \
  --use-ba \
  --implementation bae
```

#### 选项 C: VGGT-Omega 无 BA (最快)

```bash
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint /path/to/vggt_omega_1b_512.pt \
  --image-resolution 512 \
  --scene-dir /path/to/scene
```

---

## 📊 完整参数说明

### 模型选择
```bash
--model {vggt,vggt-omega}    # 选择模型 (默认: vggt)
--omega-checkpoint PATH      # VGGT-Omega 检查点路径 (必需当 model=vggt-omega)
--image-resolution INT       # 输入分辨率 (默认: 512, 仅用于 VGGT-Omega)
```

### 场景输入
```bash
--scene-dir PATH             # 场景目录路径 (必需)
```

### Bundle Adjustment 配置
```bash
--use-ba                     # 启用 BA 优化 (默认: 禁用)
--implementation {pycolmap,bae}  # BA 实现方式 (默认: pycolmap)
--max-reproj-error FLOAT     # 重投影误差阈值像素数 (默认: 8.0)
```

### 轨迹预测配置
```bash
--query-frame-num INT        # 选择多少帧进行追踪 (默认: 8)
--max-query-pts INT          # 每帧最多特征点数 (默认: 4096)
--vis-thresh FLOAT           # 可见性得分阈值 (默认: 0.2)
--fine-tracking              # 启用细粒度追踪,更准但更慢 (默认: True)
```

### 其他参数
```bash
--shared-camera              # 所有图像共用一个相机模型 (默认: False)
--camera-type STR            # 相机类型 (默认: SIMPLE_PINHOLE)
--conf-thres-value FLOAT     # 深度置信度阈值 (默认: 5.0)
--seed INT                   # 随机种子 (默认: 42)
```

---

## 📈 使用场景对比

### 场景 1: 精度优先 (静态扫描)
```bash
python demo_colmap_omega.py \
  --model vggt \
  --scene-dir scene_static \
  --use-ba \
  --implementation bae \
  --query-frame-num 8 \
  --max-query-pts 4096 \
  --max-reproj-error 5.0
```
**特点**: 使用原 VGGT (精度最高) + 完整 BA 优化

### 场景 2: 速度优先 (视频提帧)
```bash
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint ckpt.pt \
  --image-resolution 512 \
  --scene-dir scene_video \
  --use-ba \
  --query-frame-num 5 \
  --max-query-pts 2048
```
**特点**: VGGT-Omega 快速推理 + 轻量级 BA

### 场景 3: 快速预览 (初始检查)
```bash
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint ckpt.pt \
  --image-resolution 256 \
  --scene-dir scene_quick
```
**特点**: 最小分辨率,仅推理,无 BA

---

## 🔧 高级配置示例

### 对比两个模型的结果

```bash
# 用 VGGT + BA
python demo_colmap_omega.py \
  --model vggt \
  --scene-dir /tmp/vggt_result \
  --use-ba \
  --seed 42

# 用 VGGT-Omega + BA
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint ckpt.pt \
  --image-resolution 512 \
  --scene-dir /tmp/omega_result \
  --use-ba \
  --seed 42

# 对比结果
diff /tmp/vggt_result/sparse/points.ply /tmp/omega_result/sparse/points.ply
```

### 调试模式 (降低 BA 复杂度快速测试)

```bash
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint ckpt.pt \
  --scene-dir scene \
  --use-ba \
  --query-frame-num 3 \
  --max-query-pts 1024 \
  --vis-thresh 0.5
```

---

## 📂 输出结构

```
SCENE_DIR/
├── images/                  # 原始图像 (输入)
└── sparse/                  # COLMAP 格式重建结果
    ├── cameras.bin          # 相机参数
    ├── images.bin           # 相机位姿
    ├── points3D.bin         # 3D 点
    └── points.ply           # PLY 可视化点云
```

### 与其他工具集成

```bash
# 用于 Gaussian Splatting (gsplat)
python -m gsplat.train \
  --data-dir SCENE_DIR \
  --result-dir ./results

# 用于 NeRF (nerfstudio)
ns-process-data colmap \
  --data SCENE_DIR \
  --output-dir ./nerfstudio_data

# 用于 3D Gaussian Splatting (3dgs)
python train.py -s SCENE_DIR -m output
```

---

## ⚙️ 实现细节

### 关键流程

```
1. 模型推理
   ├─ VGGT: 直接输出 extrinsic [3x4], intrinsic [3x3]
   └─ VGGT-Omega: FOV 编码 [9D] → 自动转换为标准矩阵

2. 深度反投影
   └─ depth [H,W] → points_3d [N, H, W, 3]

3. 轨迹预测 (BA 前置条件)
   └─ 跨帧特征匹配 → tracks [N, P, 2]

4. Bundle Adjustment
   ├─ BAE: 使用 Levenberg-Marquardt + SE3 参数化
   └─ PyColmap: COLMAP 自己的 BA 求解器

5. 结果导出
   └─ COLMAP 格式 (二进制)
```

### 参数转换 (VGGT-Omega 关键)

```python
# FOV 编码 → 标准矩阵
pose_encoding = [T(3), quat(4), fov_v(1), fov_h(1)]  # 9D
extrinsic, intrinsic = omega_encoding_to_camera(pose_encoding)
# extrinsic: [3x4], intrinsic: [3x3] ✓
```

---

## 🐛 常见问题

### Q1: VGGT-Omega 需要单独安装吗?

**A**: 是的。需要:
```bash
pip install git+https://github.com/facebookresearch/vggt-omega.git
```

不安装的话,脚本会警告并禁用 VGGT-Omega 选项。

### Q2: 为什么 VGGT-Omega + BA 比原始更快?

**A**: 因为:
- VGGT-Omega 原生支持灵活分辨率 (不需要 1024→518 的预处理)
- 可以用 256 或 384 而不是固定 518 分辨率
- 推理本身更快 (模型设计优化)

### Q3: BA 优化能改善多少精度?

**A**: 根据场景而定:
- **好场景** (充足的轨迹): +10-20% 精度
- **普通场景**: +5-10%
- **困难场景** (少量轨迹): 无效或负面

### Q4: 怎么选择 `--max-reproj-error`?

**A**: 建议值:
- `4.0`: 严格 (适合高精度需求)
- `8.0`: 平衡 (推荐)
- `12.0`: 宽松 (容错强)

### Q5: 能否同时优化多个场景?

**A**: 不能直接。但可以:
```bash
for scene in scenes/*/; do
  python demo_colmap_omega.py \
    --model vggt-omega \
    --omega-checkpoint ckpt.pt \
    --scene-dir "$scene" \
    --use-ba
done
```

---

## 📝 测试与验证

### 验证安装

```bash
# 检查 VGGT-Omega
python -c "from vggt_omega.models import VGGTOmega; print('✓ VGGT-Omega OK')"

# 检查 BA 依赖
python -c "import pypose; import pycolmap; print('✓ BA deps OK')"
```

### 最小测试用例

```bash
# 1. 创建测试场景
mkdir -p test_scene/images

# 2. 复制几张图像
cp sample_imgs/*.jpg test_scene/images/

# 3. 运行推理
python demo_colmap_omega.py \
  --model vggt-omega \
  --omega-checkpoint ckpt.pt \
  --scene-dir test_scene \
  --use-ba

# 4. 检查结果
ls test_scene/sparse/
```

---

## 📚 参考文献

- **VGGT**: Wang et al., "VGGT: Visual Geometry Grounded Transformer", CVPR 2025
- **VGGT-Omega**: Wang et al., "VGGT-Ω: Improved Camera and Dense Scene Understanding", CVPR 2026
- **Bundle Adjustment**: Triggs et al., "Bundle Adjustment - A Modern Synthesis"

---

## 📞 获取帮助

如有问题,请:
1. 查看脚本的 `--help` 信息
2. 检查日志输出信息
3. 提交 GitHub Issue

```bash
python demo_colmap_omega.py --help
```
