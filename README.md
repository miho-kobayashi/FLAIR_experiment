# FLAIR_experiment
# FLAIR前処理変換比較実験

## 目的
乗法的季節性を持つ時系列に対し、入力変換がFLAIRの予測性能に与える影響を検証。

## 仮説
レベル上昇に伴って振幅も増えるデータでは、rawのままではShapeが歪む可能性がある。
そのため、log変換やEWMA residualにより、Shapeが見えやすい空間へ写像する。

## データ
Electricity Load Diagrams  
train: 2011-2012  
valid: 2013  
test: 2014

## 比較した前処理
- raw
- log1p
- boxcox
- ewma_resid_24 / 72 / 168
- log_ewma_resid_24 / 72 / 168

## 結果
testで最も良かった変換は log_ewma_resid_24。
rawよりMASE24が改善した。

## 考察
Shape推定器を複雑化するのではなく、入力空間を軽く整えることで予測性能が改善した。
これはFLAIRのシンプルな設計思想と整合的である。

