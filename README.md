# Screentone Raymarching — Unified GLSL for TouchDesigner

TouchDesigner の GLSL TOP 上で動作するレイマーチングシェーダー。
4種類のジオメトリモードと、ハーフトーン／ハッチングによるスクリーントーン表現を統合した VJ 向けシステム。
TouchDesinger2025.32050、GLSL4.30で作成。

---

## ⚠ 利用にあたって

- **理は、本ファイルの使用によって生じたいかなる損害・トラブルについても一切の責任を負いません。** 自己責任でご利用ください。
- **改変 OK・SNS 投稿 OK** — コードを改変し、その映像を SNS 等に投稿することは自由です。クレジット不要。
- **そのまま使った映像を SNS に上げる場合** — 改変なしで TouchDesigner の出力映像をそのまま投稿する場合は、一報いただけると助かります（X / Instagram 等どこでも可）。
- **VJ・映像素材としての利用** — ライブ VJ、映像演出、プロジェクションマッピング等の素材として使う分には何も問題ありません。自由にどうぞ。

---

## 概要

1本の Pixel Shader で **4つの表示モード** を切り替え可能。カメラ・ライト・スクリーントーン・カラーは全モード共通で、`uDisplayMode` uniform の値だけでシーンが切り替わる。

### Display Mode（`uDisplayMode`）

| 値 | モード | 内容 |
|----|--------|------|
| 1 | Multiple Spheres | 複数の球体が円環状に配置され、ノイズで揺らぎながら浮遊する |
| 2 | Cubes | 複数の立方体が回転しながら円環配置。エッジの効いたスクリーントーンが映える |
| 3 | Single Sphere | 単体の球がノイズ駆動でゆっくり漂う。シンプルかつスクリーントーンの質感が際立つ |
| 4 | Tetrahedrons | 正四面体が回転しつつ配置。幾何学的な陰影が特徴的 |

---

## Uniform 一覧

### 基本

| Uniform | 型 | 内容 |
|---------|----|------|
| `iResolution` | `vec4` | 解像度（`.xy` を使用） |
| `iTime` | `float` | 経過時間（秒） |
| `uDisplayMode` | `float` | 表示モード（1–4） |

### シーン制御（`uScene`）

| 成分 | 内容 | 範囲 |
|------|------|------|
| `.x` | オブジェクト数 | 1–12（Mode 3 では未使用） |
| `.y` | カメラモード | 1–4 |
| `.z` | ライトモード | 1–4 |
| `.w` | スタイルモード | 1=ハーフトーン, 2=ハッチング |

### オブジェクト（`uTetra`）

| 成分 | 内容 |
|------|------|
| `.x` | オブジェクト回転（0=固定, 0.5以上=回転。Mode 2, 4 で有効） |
| `.y` | オブジェクトサイズ |
| `.z` | 配置の広がり（spread） |

### カメラ（`uCamera`）

| 成分 | 内容 | 対応カメラモード |
|------|------|-----------------|
| `.x` | 回転速度 | Mode 1: Rotating |
| `.y` | カメラ距離 | Mode 1, 2, 3 共通 |
| `.z` | ステップ間隔 | Mode 2: 45° Step |
| `.w` | Figure-8 速度 | Mode 3: Figure-8 |

### カメラモード詳細

| 値 | 名前 | 動き |
|----|------|------|
| 1 | Rotating | 原点を中心に滑らかに周回。上下にも緩やかに揺れる |
| 2 | 45° Step | 45度ずつステップ回転。イージングで滑り込む動き |
| 3 | Figure-8 | 8の字軌道。距離が周期的に伸縮する |
| 4 | Manual | `uPos.xyz` と `uRad.xyz` で位置・注視点を直接指定 |

### 手動カメラ（カメラモード 4 のみ）

| Uniform | 内容 |
|---------|------|
| `uPos.xyz` | カメラ位置 |
| `uRad.xyz` | カメラ注視点 |

### スクリーントーン（`uScreentone`）

| 成分 | 内容 | スタイルモード |
|------|------|---------------|
| `.x` | ドットスケール | 1: Halftone |
| `.y` | ラインスケール | 2: Hatching |
| `.z` | ライン角度（rad） | 2: Hatching |

### ライトモード（`uScene.z`）

| 値 | 名前 | 動き |
|----|------|------|
| 1 | Rotating | ゆっくり周回する白色光 |
| 2 | Random | Perlin ノイズ駆動でランダムに動く |
| 3 | Fixed | 固定位置（4, 6, 4） |
| 4 | Blue | 固定位置 + 青白い光色 |

### カラー

| Uniform | 内容 |
|---------|------|
| `uAccentColor.xyz` | リムライト・スペキュラに乗るアクセント色 |
| `uBgColor.xyz` | 背景色 |

### レイマーチング制御（`uRay`）

| 成分 | 内容 | 推奨値 |
|------|------|--------|
| `.x` | 最大ステップ数 | 128 |
| `.y` | 最大レイ距離 | 50 |
| `.z` | サーフェス判定閾値 | 0.001 |

---

## TouchDesigner での使い方

1.toxファイルをダウンロードし、touchdesignerにD&D。

2. Parameter COMP で各値をコントロールすれば、MIDI マッピングや UI からリアルタイム制御ができる

---

## シェーダー構造

```
Noise (hash / noise / smoothNoise3)
  │
Rotation (rot2D / rotX / rotY / rotZ)
  │
Camera System (getCamera / setCamera)
  │   4モード: Rotating / 45° Step / Figure-8 / Manual
  │
Light System (getLightPos / getLightColor)
  │   4モード: Rotating / Random / Fixed / Blue
  │
Screentone (halftone / hatching / applyScreentone)
  │   2スタイル: Halftone Dots / Hatching Lines
  │
SDF Primitives (sdSphere / sdBox / sdTetrahedron)
  │
Object Layout (getObjectPos / getObjectSize / getObjectRot)
  │   uDisplayMode で形状分岐
  │
Scene SDF (map / mapObject)
  │
Raymarch → calcNormal → getHitObject
  │
Shading (shadeObject / shadeSingle / shadeGround)
  │   Diffuse + Specular + Rim + Screentone + Accent
  │
Main → Vignette → fragColor
```

---

## 技術メモ

- レイマーチングのループ上限は 256 だが、`uRay.x`（Max Steps）で実質的なステップ数を制御できる。パフォーマンスが厳しい場合は 64–96 程度に下げても十分使える
- 地面は `y = -2.0`（Mode 3 のみ `y = -1.5`）の無限平面
- Mode 1 と Mode 2/4 でスペキュラのパワーやリムの強度が微妙に異なる。形状ごとに映えるバランスに調整済み
- ビネット効果は最終段で一律適用（強度 0.3、半径 1.5）

---

## Author

理　@ri___0_1
