# Sphere-Stereo 元論文実装チュートリアル

## 📋 概要

元論文（CVPR 2021 Oral）のPython実装プログラムです。
4枚の魚眼カメラ画像から360°パノラマ画像と深度マップを生成します。

**論文**: "Real-Time Sphere Sweeping Stereo from Multiview Fisheye Images"  
**著者**: Andreas Meuleman et al., KAIST Visual Computing Lab

---

## 🎯 実行フロー

1. **カメラ撮影**: quad_cam_systemで4枚の静止画を同時撮影
2. **画像配置**: 撮影した画像をresourcesディレクトリに配置
3. **実行**: sphere-stereoで360°パノラマRGB-Dを生成

---

## 📸 ステップ1: 4枚の画像を撮影

### 1-1. ROS2ワークスペースをビルド

```bash
cd /home/motoken/Fisheye-Lens-SLAM/ros2_ws
colcon build --packages-select quad_cam_system
source install/setup.bash
```

### 1-2. 撮影サービスを起動

```bash
# サービスノードを起動
ros2 launch quad_cam_system max_resolution_capture_service.launch.py \
    output_dir:=/home/motoken/Fisheye-Lens-SLAM/sphere-stereo/temp_capture \
    filename_prefix:=sphere_capture
```

### 1-3. 撮影を実行

別のターミナルで：

```bash
# ROS2環境をセットアップ
cd /home/motoken/Fisheye-Lens-SLAM/ros2_ws
source install/setup.bash

# 撮影をトリガー
ros2 service call /capture_max_resolution_images std_srvs/srv/Trigger
```

撮影が成功すると、以下のファイルが生成されます：
```
sphere-stereo/temp_capture/
├── sphere_capture_camera0_YYYYMMDD_HHMMSS_xxx.jpg
├── sphere_capture_camera1_YYYYMMDD_HHMMSS_xxx.jpg
├── sphere_capture_camera2_YYYYMMDD_HHMMSS_xxx.jpg
└── sphere_capture_camera3_YYYYMMDD_HHMMSS_xxx.jpg
```

**解像度**: 1944×1096ピクセル（最大解像度）  
**品質**: JPEG 100%（最高品質）

---

## 📁 ステップ2: 画像をresourcesに配置

### 2-1. ディレクトリ構造を作成

```bash
cd /home/motoken/Fisheye-Lens-SLAM/sphere-stereo

# 既存のresourcesをバックアップ（オプション）
# mv resources resources_backup

# ディレクトリ構造を作成
mkdir -p resources/cam0 resources/cam1 resources/cam2 resources/cam3 resources/output
```

### 2-2. 画像をリネームして配置

撮影した画像を各カメラディレクトリに `0.jpg` としてコピー：

```bash
# 撮影した画像のファイル名を確認
ls temp_capture/

# 各カメラの画像を配置（ファイル名を適宜変更）
cp temp_capture/sphere_capture_camera0_*.jpg resources/cam0/0.jpg
cp temp_capture/sphere_capture_camera1_*.jpg resources/cam1/0.jpg
cp temp_capture/sphere_capture_camera2_*.jpg resources/cam2/0.jpg
cp temp_capture/sphere_capture_camera3_*.jpg resources/cam3/0.jpg
```

**または一括リネーム:**

```bash
cd temp_capture
for i in 0 1 2 3; do
    # 最新のファイルを取得
    latest=$(ls -t sphere_capture_camera${i}_*.jpg | head -1)
    cp "$latest" ../resources/cam${i}/0.jpg
    echo "Copied $latest to cam${i}/0.jpg"
done
cd ..
```

### 2-3. キャリブレーションファイルを配置

Basaltで生成したキャリブレーションファイルをコピー：

```bash
# Basaltのキャリブレーション結果から
cp /path/to/basalt_calibration_result.json resources/calibration.json

# または既存のキャリブレーションを使用（すでに存在する場合）
# ls resources/calibration.json
```

**calibration.json の形式**:
- Double Sphere (ds) カメラモデル
- 各カメラの内部パラメータ (fx, fy, cx, cy, xi, alpha)
- 各カメラの外部パラメータ (quaternion + translation)

### 2-4. マスク画像の配置（オプション）

各カメラの有効領域マスクを配置：

```bash
# マスク画像がある場合
cp /path/to/mask_cam0.png resources/cam0/mask.png
cp /path/to/mask_cam1.png resources/cam1/mask.png
cp /path/to/mask_cam2.png resources/cam2/mask.png
cp /path/to/mask_cam3.png resources/cam3/mask.png
```

マスク画像がない場合は、プログラムが自動的にスキップします。

### 2-5. 最終的なディレクトリ構造

```
sphere-stereo/resources/
├── calibration.json          # カメラキャリブレーションファイル（必須）
├── cam0/
│   ├── 0.jpg                # カメラ0の画像（必須）
│   └── mask.png             # マスク画像（オプション）
├── cam1/
│   ├── 0.jpg                # カメラ1の画像（必須）
│   └── mask.png             # マスク画像（オプション）
├── cam2/
│   ├── 0.jpg                # カメラ2の画像（必須）
│   └── mask.png             # マスク画像（オプション）
├── cam3/
│   ├── 0.jpg                # カメラ3の画像（必須）
│   └── mask.png             # マスク画像（オプション）
└── output/                   # 出力ディレクトリ（自動生成）
```

---

## 🚀 ステップ3: Sphere-Stereoを実行

### 3-1. Python仮想環境をアクティベート

```bash
cd /home/motoken/Fisheye-Lens-SLAM/sphere-stereo
source .venv/bin/activate
```

### 3-2. プログラムを実行

**基本実行:**

```bash
python python/main.py \
    --dataset_path resources \
    --references_indices 0 1 2 3 \
    --visualize True
```

**パラメータ調整例:**

```bash
python python/main.py \
    --dataset_path resources \
    --references_indices 0 1 2 3 \
    --min_dist 0.55 \
    --max_dist 100 \
    --candidate_count 32 \
    --sigma_i 10 \
    --sigma_s 25 \
    --matching_resolution 1024 1024 \
    --panorama_resolution 2048 1024 \
    --visualize True
```

### 3-3. 主要パラメータの説明

| パラメータ | 説明 | デフォルト |
|----------|------|----------|
| `--dataset_path` | 入力データセットのパス | `resources` |
| `--references_indices` | 深度推定を行うカメラのインデックス | `0 2` |
| `--min_dist`, `--max_dist` | 最小・最大探索距離（メートル） | `0.55`, `100` |
| `--candidate_count` | 深度候補数（高いほど精度向上） | `32` |
| `--sigma_i` | エッジ保存パラメータ（低いほど鮮明） | `10` |
| `--sigma_s` | 平滑化パラメータ（高いほど滑らか） | `25` |
| `--matching_resolution` | マッチング解像度 [幅, 高さ] | `[1024, 1024]` |
| `--panorama_resolution` | 出力パノラマ解像度 [幅, 高さ] | `[2048, 1024]` |
| `--visualize` | 結果を可視化 | `False` |

**references_indicesについて:**
- 全方位カバレッジに必要なカメラを指定
- 例: `0 1 2 3` = 全4台のカメラで深度推定（高精度、低速）
- 例: `0 2` = 対角の2台のみ使用（高速、やや低精度）

---

## 📊 ステップ4: 結果の確認

### 4-1. 出力ファイル

実行後、以下のファイルが生成されます：

```
sphere-stereo/resources/output/
├── rgb_0.png                    # RGBパノラマ画像（2048×1024）
├── inv_distance_0.exr           # 距離マップ（float32、EXR形式）
└── inv_distance_0_colored.png   # カラーマップ化された距離画像（可視化用）
```

### 4-2. 結果の表示

**RGB パノラマ:**
```bash
eog resources/output/rgb_0.png
# または
feh resources/output/rgb_0.png
```

**距離マップ（可視化版）:**
```bash
eog resources/output/inv_distance_0_colored.png
```

**EXRファイルの解析（Python）:**
```python
import OpenEXR
import numpy as np
import Imath

# EXRファイルを読み込み
exr_file = OpenEXR.InputFile('resources/output/inv_distance_0.exr')
dw = exr_file.header()['dataWindow']
size = (dw.max.x - dw.min.x + 1, dw.max.y - dw.min.y + 1)

# Float32として読み込み
FLOAT = Imath.PixelType(Imath.PixelType.FLOAT)
inv_distance = np.frombuffer(exr_file.channel('Y', FLOAT), dtype=np.float32)
inv_distance = inv_distance.reshape(size[1], size[0])

# 距離に変換（逆数）
distance_map = 1.0 / (inv_distance + 1e-6)

print(f"距離範囲: {distance_map.min():.2f}m - {distance_map.max():.2f}m")
```

---


## 📝 クイックリファレンス

**完全な実行フロー（コピペ用）:**

```bash
# 1. 撮影
cd /home/motoken/Fisheye-Lens-SLAM/ros2_ws
source install/setup.bash
ros2 launch quad_cam_system max_resolution_capture_service.launch.py \
    output_dir:=/home/motoken/Fisheye-Lens-SLAM/sphere-stereo/temp_capture &
sleep 3
ros2 service call /capture_max_resolution_images std_srvs/srv/Trigger
killall ros2

# 2. 画像配置
cd /home/motoken/Fisheye-Lens-SLAM/sphere-stereo
mkdir -p temp_capture
cd temp_capture
for i in 0 1 2 3; do
    latest=$(ls -t sphere_capture_camera${i}_*.jpg 2>/dev/null | head -1)
    if [ -n "$latest" ]; then
        cp "$latest" ../resources/cam${i}/0.jpg
        echo "✓ Copied to cam${i}/0.jpg"
    fi
done
cd ..

# 3. 実行
source .venv/bin/activate
python python/main.py \
    --dataset_path resources \
    --references_indices 0 1 2 3 \
    --visualize True

# 4. 結果表示
eog resources/output/rgb_0.png
eog resources/output/inv_distance_0_colored.png
```

---

## 🔗 関連ドキュメント

- [quad_cam_system/README_max_resolution_capture.md](../ros2_ws/src/quad_cam_system/README_max_resolution_capture.md) - カメラ撮影詳細
- [README_basalt.md](README_basalt.md) - キャリブレーション手順
- [sphere-stereo/README.md](sphere-stereo/README.md) - 元論文の詳細とデータセット

---

## 📚 参考文献

- **論文**: Andreas Meuleman et al., "Real-Time Sphere Sweeping Stereo from Multiview Fisheye Images", *IEEE CVPR 2021* (Oral)
- **プロジェクトページ**: http://vclab.kaist.ac.kr/cvpr2021p1/
- **GitHub**: https://github.com/KAIST-VCLAB/sphere-stereo