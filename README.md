# FLAIR 入力変換比較実験

## 概要

本実験では、FLAIR の時系列予測に対して、

- 入力変換
- trend除去
- スケール補正

がどのような影響を与えるかを検証した。

中心となる仮説は以下である。

> Shape推定器そのものを複雑化するよりも、  
> Shapeが見えやすい空間へ入力を変換した方が、予測性能が改善するのではないか。

対象データには電力需要データを使用した。

---

## 実験背景

FLAIR は概念的に、

```text
Level + Shape + Noise
```

というシンプルな構造を持つ。

しかし電力需要データでは、

```text
需要レベルが大きくなると、
日内変動の振幅も同時に大きくなる
```

現象が確認できる。

例えば：

- 夏は需要そのものが高い
- 同時に昼夜差も大きくなる

これは典型的な：

```text
乗法的季節性
```

を示している。

このとき raw データのまま Shape 推定を行うと、

```text
本来同じ形状であるはずの波形が、
振幅の違いだけで別Shapeとして扱われる
```

可能性がある。

本実験では、

```text
Shapeが見えやすい座標系
```

を探索することを目的とした。

---

## 使用データ

### Dataset

Electricity Load Diagrams Dataset

### 元データ

15分足

### 使用時

1時間足へ resample

---

## データ分割

| split | period |
|---|---|
| train | 2011-2012 |
| valid | 2013 |
| test | 2014 |

---

## Forecast設定

| parameter | value |
|---|---|
| Horizon | 24 |
| Eval Step | 24 |
| Context Bars | 2160 |
| Samples | 100 |

---

# 比較した前処理

---

## 1. raw

### 数式

```math
x_t
```

### 内容

生データをそのまま入力。

### 特徴

- Level情報を完全保持
- 振幅もそのまま保持
- 最も自然なShape

一方で、

```text
レベル依存振幅
```

もそのまま残る。

---

## 2. log1p

### 数式

```math
\log(1+x_t)
```

### 目的

大きな値を圧縮し、振幅を安定化する。

### 仮説

乗法的季節性：

```math
x_t = Level_t \times Shape_t \times Noise_t
```

を、

```text
より加法的な構造
```

へ近づける。

### 期待効果

- Shape整列
- 分散安定化
- Shape stability改善

### 結果

Shape stability は最良。

これは、

```text
日ごとのShapeばらつきが減少
```

したことを示唆する。

---

## 3. Box-Cox

### 数式

```math
\frac{x_t^\lambda - 1}{\lambda}
```

### 内容

log変換を一般化した変換。

### λ の意味

| λ | 意味 |
|---|---|
| 1 | raw |
| 0 | log |
| 0.5 | sqrt |

### 仮説

```text
最適な変換強度
```

が存在する可能性。

### 結果

性能は大きく悪化。

特に逆変換時に不安定性が発生。

### 考察

```text
自由度が高すぎる
```

可能性。

これは FLAIR の：

```text
シンプルなShape推定
```

思想とも整合的。

---

## 4. EWMA residual

### 数式

```math
x_t - EWMA(x_t)
```

### 内容

指数移動平均を引くことで、

```text
長期Level
```

を除去する。

### 仮説

trend / level を除去することで、

```text
純粋なShape
```

が見えやすくなる可能性。

### 結果

単体では不安定。

確認された問題：

- residualノイズ増幅
- Level復元不安定
- 過剰変動

### 考察

trend除去単体では、

```text
乗法的季節性
```

を扱えない可能性。

---

## 5. log + EWMA residual

### 数式

```math
\log(1+x_t)-EWMA(\log(1+x_t))
```

### 内容

#### Step1

log変換で：

```text
レベル依存振幅
```

を補正。

#### Step2

EWMA residual で：

```text
長期Level
```

を除去。

### 仮説

```text
Shapeだけが残る空間
```

へ近づける。

### 結果

test において最良性能。

特に：

```text
log_ewma_resid_24
```

が最良の MASE24 を記録。

---

# 評価指標

## MASE24

Mean Absolute Scaled Error

低いほど良い。

MASE は、

```text
「昨日の同時刻と同じ」
```

という季節性ナイーブ予測と比較した誤差指標である。

---

# 実験結果

## 最良変換

```text
log_ewma_resid_24
```

---

## test結果の傾向

| transform | 傾向 |
|---|---|
| log_ewma_resid_24 | 最良 |
| log_ewma_resid_72 | 良好 |
| raw | 想像以上に強い |
| log1p | 安定 |
| EWMA residual系 | 不安定 |
| Box-Cox | × |

---

# 重要な観察結果

## 1. raw が非常に強い

これは、

```text
FLAIRのShape推定器そのものが既に強力
```

であることを示唆する。

つまり問題は：

```text
Shape推定器不足
```

ではなく、

```text
入力空間の歪み
```

にある可能性。

---

## 2. 軽い前処理は有効

最良結果は：

```text
軽量な変換
```

から得られた。

複雑な変換ほど良いわけではない。

---

## 3. log変換でShape stability改善

log変換後、

```text
Shape stability
```

が大きく改善。

これは：

```text
振幅差による擬似Shape差
```

が減少した可能性を示唆する。

---

## 4. Shape stabilityだけでは不十分

最も安定だった：

```text
log1p
```

が、

最良予測ではなかった。

### 意味

```text
Shape整列だけでは不十分
```

であり、

```text
Level除去とのバランス
```

も重要。

---

# 可視化結果の考察

## Actual vs Prediction

観察された特徴：

- 多くの変換でピーク時刻は捉えられている
- 一方でピーク高さは過小推定されやすい
  <img width="2700" height="1200" alt="all_transforms_overlay" src="https://github.com/user-attachments/assets/13f323b8-9b28-4854-8e4d-82892f916797" />


### 解釈

FLAIR は：

```text
「いつピークが来るか」
```

には強い。

一方で：

```text
「どれくらい高くなるか」
```

は弱い。

---

## Shape stability

log変換後、

```text
日ごとのShapeばらつき
```

が大幅に減少。

これは：

```text
同じShapeとして認識しやすくなった
```

可能性を示す。

---

# 結論

本実験から、

```text
Shape推定器を複雑化するよりも、
入力空間を軽く整える方が有効
```

である可能性が示唆された。

特に：

- 乗法的季節性
- レベル依存振幅
- trend混入

が Shape 推定を阻害する可能性がある。

---

# 最良手法

```text
log変換 + 短期EWMA residual
```

---

# 意義

本実験は、

```text
FLAIRのシンプルなShape推定思想
```

を壊さず、

```text
入力側のみを整える
```

ことで改善できる可能性を示した。

---

# 今後の課題

- seasonal differencing
- STL decomposition
- λグリッド探索
- adaptive transform selection
- MDLベース変換選択
- Level専用モデル分離

---

# Repository

GitHub：

```text
https://github.com/miho-kobayashi
```

---

# QR Code

<img width="444" height="444" alt="github_qr_miho_kobayashi" src="https://github.com/user-attachments/assets/547220c9-9f84-4b99-8f70-0c3c03ad33ac" />


---
