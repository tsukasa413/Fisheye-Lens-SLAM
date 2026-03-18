# Fisheye-Lens-SLAM

## 📋 概要

本プロジェクトの中核は **my_stereo_pkg** です。これは4台の魚眼カメラから360°パノラマRGB-D画像（カラー画像 + 深度マップ）をリアルタイムに生成するROS2パッケージです。

KAIST Visual Computing Labが開発した**Sphere Sweeping Stereoアルゴリズム（CVPR 2021 Oral）の元論文Python実装をC++で再実装**し、CUDA加速により約272-308ms/フレームでの高速処理を実現しています。

### プロジェクトの全体像

```
【データ収集】          【前処理】                    【実装・検証】
quad_cam_system  ──→   Basalt         ──→  ┌→ my_stereo_pkg (C++高速実装)
(4台のカメラ制御)      (キャリブレーション)     │   ↓ 出力: RGB-D
                              ↓                │   ↓
                       calibration.json        └→ sphere-stereo (元論文Python実装)
                       (両方で使用)                ↓ 出力: RGB-D
                                                   ↓
                                              【精度検証・比較】
                                              (元論文の精度が再現できているか確認)
```

### 各コンポーネントの役割

- **my_stereo_pkg**: 元論文のC++再実装（本プロジェクトの中核・実運用向け）
- **sphere-stereo**: 元論文のPython実装（精度検証・パラメータ調整用ベースライン）
- **quad_cam_system**: 4台のカメラ画像を取得・配信
- **Basalt**: 両実装が必要とするキャリブレーションファイル（calibration.json）を生成

---

## 🗂️ プロジェクト構成

```
Fisheye-Lens-SLAM/
├── ros2_ws/                        # ROS2ワークスペース
│   ├── src/
│   │   ├── my_stereo_pkg/         # ★C++再実装: 360°パノラマRGB-D生成
│   │   └── quad_cam_system/       # カメラ制御・画像配信
│   └── output/                     # my_stereo_pkgの出力先
├── basalt/                         # キャリブレーション生成ツール
├── sphere-stereo/                  # ★元論文Python実装（検証ベースライン）
├── Pangolin/                       # 可視化ライブラリ（依存）
└── librealsense/                   # RealSenseドライバ（オプション）
```

---
🎯 中核システム: my_stereo_pkg

### ★ my_stereo_pkg - 360°パノラマRGB-D生成エンジン

**本プロジェクトのメインプログラム**。4台の魚眼カメラ画像から360°全方位のRGBパノラマ（2048×1024ピクセル）と対応する深度マップをリアルタイムに
複数の魚眼カメラから取得した画像を用いて、全方位RGBDパノラマ（2048×1024ピクセル）を生成します。

**特徴:**
- **完全C++実装**: Pythonへの依存なし、高速処理
- **CUDA加速**: GPU処理により約272-308ms/フレームで動作
- **リアルタイム処理**: ROS2ノードとして動作し、カメラトピックから直接画像を受信
- **スタンドアロン実行**: 保存済みデータセットからのバッチ処理にも対応

**使用技術:**
- **Sphere Sweeping Stereo**: KAIST Visual Computing Labで開発された魚眼カメラ向けステレオマッチングアルゴリズム
- **Double Sphere カメラモデル**: 広角魚眼レンズの歪みを正確にモデル化
- **PyTorch C++ API (LibTorch)**: CUDA加速による高速画像処理

**出力:**
- RGB パノラマ画像 (2048×1024 pixels)
- 距離マップ (.exr形式、float32、メートル単位)
- カラーマップ化された距離画像 (可視化用)

**入力要件:**
- 4台のカメラ画像（ROS2トピック or 保存済みファイル）
- キャリブレーションファイル calibration.json（Basaltで生成）

**詳細ドキュメント:**
- [README_standalone.md](ros2_ws/src/my_stereo_pkg/README_standalone.md) - スタンドアロン実行方法
- [README_REALTIME.md](ros2_ws/src/my_stereo_pkg/README_REALTIME.md) - リアルタイムROS2ノード実行方法

---

## 📊 元論文実装: sphere-stereo

### ★ sphere-stereo - 元論文Python実装（検証ベースライン）

**役割**: my_stereo_pkgの精度を検証するためのベースライン実装

### 1. quad_cam_system - カメラデータ収集

**役割**: my_stereo_pkgに必要な4台のカメラ画像を同期取得・配信

**機能:**
- 4台のIMX477魚眼カメラの同時制御
- ROS2トピックとして画像配信（`/camera_0/image_raw` など）
- 最大解像度（1944×1096）での静止画撮影
- ROS2バッグ形式での記録

**使用タイミング:**
- my_stereo_pkgをリアルタイムで実行する時（カメラ画像配信）
- sphere-stereo検証用の静止画を撮影する時
- データセット作成・記録時
my_stereo_pkgとの関係:**
- **独立動作**: my_stereo_pkgとは別々に動作可能
- **統合可能**: オドメトリ情報と360°RGB-Dを組み合わせて3Dマッピングに発展可能
- **センサー種別**: LiDAR+IMU（カメラ不要）

**詳細ドキュメント:**
- [FAST_LIO_ROS2/README.md](ros2_ws/src/FAST_LIO_ROS2/README.md)
- [FAST_LIO/README.md](ros2_ws/src/FAST_LIO_ROS2/FAST_LIO/README.md) - アルゴリズム詳細

---

## 📚 依存ライブラリ

### Pangolin
3D可視化ライブラリ。my_stereo_pkgとBasaltの結果表示に使用。

### librealsense
Intel RealSenseカメラドライバ。追加のセンサー統合時に使用可能（オプション）
**役割**: ロボットの位置・姿勢をリアルタイムに推定（my_stereo_pkgとは独立）

LiDARとIMUを統合した高速かつロバストなオドメトリシステムです。

**特徴:**
- **高速処理**: ikd-Treeを用いたインクリメンタルマッピングにより100Hz以上の処理速度
- **Direct Odometry**: 特徴点抽出不要、生のLiDAR点群を直接使用
- **多様なLiDAR対応**: 回転式（Velodyne、Ouster）および固体式（Livox）LiDARに対応
- **外部IMU対応**: 様々なIMUセンサーと統合可能
- **ARM対応**: Jetson TX2、Raspberry Pi 4など組込みプラットフォームで動作

**使用技術:**
- **iterated Extended Kalman Filter (iEKF)**: LiDARとIMUデータの密結合融合
- **ikd-Tree**: 3D k近傍探索のための動的KDツリーデータ構造
- **並列KD-Tree探索**: 計算負荷の削減

**出力:**
- リアルタイムオドメトリ（位置・姿勢推定）
- 3Dポイントクラウドマップ
- 軌跡データ（保存機能付き）

**詳細ドキュメント:**
- [FAST_LIO_ROS2/README.md](ros2_ws/src/FAST_LIO_ROS2/README.md)
- [FAST_LIO/README.md](ros2_ws/src/FAST_LIO_ROS2/FAST_LIO/README.md) - アルゴリズム詳細

---

## 📐 カメラキャリブレーション - Basalt

魚眼カメラの内部・外部パラメータの高精度キャリブレーションに使用します。

**特徴:**
- **Double Sphere カメラモデル**: 広角魚眼レンズ（180°以上）の歪みを正確にモデル化
- **AprilGridベース**: チェッカーボードよりも高精度な検出が可能
- **複数カメラ対応**: 4台の魚眼カメラの相対姿勢を同時にキャリブレーション
- **VIO/SLAM対応**: Visual-Inertial OdometryおよびMappingも可能（本プロジェクトではキャリブレーションのみ使用）

**キャリブレーション出力:**
- 各カメラの内部パラメータ (fx, fy, cx, cy, xi, alpha)
- 各カメラの外部パラメータ (rotation quaternion + translation vector)
- JSON形式のキャリブレーションファイル（my_stereo_pkgで使用）

**使用方法:**
```bash
# Docker環境で実行
docker run -it --rm --runtime nvidia --volume ~/docker_sync:/data basalt:jetson /bin/bash

# AprilGridキャリブレーション実行
/opt/basalt/build/basalt_calibrate \
    --dataset-path /data/mono_camera.bag \
    --dataset-type bag \
    --aprilgrid /opt/basalt/data/aprilgrid_6x6.json \
    --result-path /data/calib_results/ \
    --cam-types ds ds ds ds
```

**関連論文:**
- "The Double Sphere Camera Model" (3DV 2018) - 魚眼カメラモデルの理論
- "Visual-Inertial Mapping with Non-Linear Factor Recovery" (RA-L 2020)

**詳細ドキュメント:**
- [README_basalt.md](README_basalt.md) - 完全なキャリブレーション手順
- [basalt/README.md](basalt/README.md) - Basaltシステム概要

---

## 📚 参考実装・ライブラリ

### sphere-stereo
KAIST Visual Computing Labで開発されたSphere Sweeping Stereoアルゴリズムの参照実装（Python版）。本プロジェクトのmy_stereo_pkgは、このアルゴリズムのC++実装版です。

- **論文**: "Real-Time Sphere Sweeping Stereo from Multiview Fisheye Images" (CVPR 2021 Oral)
- **用途**: アルゴリズム検証、パラメータチューニング

### Pangolin
3D可視化ライブラリ。Basaltおよびmy_stereo_pkgの結果表示に使用。

### librealsense
Intel RealSenseカメラドライバ。追加のセンサー統合時に使用可能。

---

## 🔧 環境要件

- **OS**: Ubuntu 20.04 / 22.04（推奨）
- **ROS2**: Humble（推奨）またはFoxy以上
- **CUDA**: 10.2以上（GPU加速に必要）
- **依存ライブラリ**: 
  - OpenCV >= 4.0
  - Eigen >= 3.3.4
  - PCL >= 1.8
  - LibTorch (PyTorch C++ API)
  - nlohmann/json

---

## 📖 詳細ドキュメント

各コンポーネントの詳細な使用方法については、以下のドキュメントを参照してください:

- [README_basalt.md](README_basalt.md) - Basaltによるカメラキャリブレーション
- [README_camera.md](README_camera.md) - カメラハードウェア設定
- [README_kalibr.md](README_kalibr.md) - Kalibrキャリブレーションツール
- [README_sphere-stereo.md](README_sphere-stereo.md) - Sphere Stereoアルゴリズム
- [ros2_ws/src/my_stereo_pkg/](ros2_ws/src/my_stereo_pkg/) - ステレオ推定パッケージドキュメント

---

## 🎯 使用の流れ

1. **カメラキャリブレーション**: Basaltを使用して4台の魚眼カメラをキャリブレーション
2. **データ収集**: ROS2でカメラ画像・LiDARデータを記録
3. **ステレオ深度推定**: my_stereo_pkgで360°パノラマRGB-Dを生成
4. **オドメトリ推定**: FAST_LIO_ROS2でロボットの位置姿勢を推定
5. **結果統合**: パノラマ画像とオドメトリを組み合わせて3Dマップを構築

---

## 📝 ライセンス

各コンポーネントは独自のライセンスに従います:
- Basalt: BSD 3-Clause License
- sphere-stereo: 研究・評価目的の使用のみ

---

## 🔗 関連リンク

- [Basalt Project Page](https://gitlab.com/VladyslavUsenko/basalt)
- [Sphere Sweeping Stereo Project Page](https://github.com/KAIST-VCLAB/sphere-stereo#)
🎯 使用の流れ（my_stereo_pkg中心）

### 🔰 初回セットアップ

```
1. Basaltでキャリブレーション実行
   ↓ (calibration.json生成)
2. my_stereo_pkgにキャリブレーションファイルを配置
   ↓
3. 準備完了！
```

### 📸 リアルタイム実行

```
1. quad_cam_system起動（カメラ画像配信）
   ↓
2. my_stereo_pkg起動（リアルタイムRGB-D生成）
   ↓
3. 360°パノラマRGB-D出力
```

### 🧪 静止画テスト（sphere-stereo）

```
1. quad_cam_systemで4枚撮影
   ↓
2. sphere-stereo/resourcesに配置
   ↓� クイックスタート

### my_stereo_pkgをスタンドアロン実行（最も簡単）

```bash
cd /home/motoken/Fisheye-Lens-SLAM/ros2_ws
colcon build --packages-select my_stereo_pkg
source install/setup.bash

# 実行（デフォルトデータセット使用）
./src/my_stereo_pkg/run_standalone_estimator.sh
```

出力: `ros2_ws/output/standalone/` に360°パノラマRGB-Dが生成されます。

### sphere-stereoで静止画テスト

```bash
cd /home/motoken/Fisheye-Lens-SLAM/sphere-stereo
source .venv/bin/activate
python python/main.py --dataset_path resources --references_indices 0 1 2 3 --visualize True
```

詳細は [README_sphere-stereo.md](README_sphere-stereo.md) を参照。

---

## 📝 ライセンス

各コンポーネントは独自のライセンスに従います:
- **my_stereo_pkg**: （ライセンス記載予定）
- **sphere-stereo**: 研究・評価目的の使用のみ（KAIST Visual Computing Lab）
- **Basalt**: BSD 3-Clause License

---

## 🔗 関連リンク

### 元論文
- [Sphere Sweeping Stereo Project Page](http://vclab.kaist.ac.kr/cvpr2021p1/) - CVPR 2021 Oral
- [Sphere Sweeping Stereo GitHub](https://github.com/KAIST-VCLAB/sphere-stereo) - 元論文Python実装
- [論文PDF](http://vclab.kaist.ac.kr/cvpr2021p1/cvpr21-sphere-sweeping.pdf)

### ツール
- [Basalt Project Page](https://gitlab.com/VladyslavUsenko/basalt) - カメラキャリブレーション
```

---

## 📖 詳細ドキュメント

### my_stereo_pkg関連
- **[ros2_ws/src/my_stereo_pkg/README_standalone.md](ros2_ws/src/my_stereo_pkg/README_standalone.md)** - スタンドアロン実行
- **[ros2_ws/src/my_stereo_pkg/README_REALTIME.md](ros2_ws/src/my_stereo_pkg/README_REALTIME.md)** - リアルタイムROS2実行

### 元論文・検証
- **[README_sphere-stereo.md](README_sphere-stereo.md)** - 元論文Python実装（精度検証ベースライン）

### サポートツール
- **[README_basalt.md](README_basalt.md)** - カメラキャリブレーション手順
- **[README_camera.md](README_camera.md)** - カメラハードウェア設定
- **[ros2_ws/src/quad_cam_system/](ros2_ws/src/quad_cam_system/)** - カメラ制御・撮影