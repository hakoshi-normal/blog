---
title: 【入門2】YOLO，Jupyterのセットアップ
# publishdate: 2023-12-01
date: 2025-05-06
tags: [YOLO, JupyterLab, Python, YOLO入門]
toc: true
---

## JupyterLab+Visual Studio Codeを使ったPython実行

1. Visual Studio CodeにJupyter拡張機能をインストール
2. 空のファイルを作成し，拡張子を```.ipynb```とする
3. ```＋コード```ボタンからコードブロックを作成し，Pythonプログラムを記述
4. 右上```カーネルの選択```から実行したいPython環境を選択し
5. コードブロック左側の実行ボタン```▷```を押下
6. ポップアップが出てきたら```インストール```をクリックして```ipykernel```をインストール
   （```pip install ipykernel```を当該環境で実行してインストールも出来ます）

{{< figure src="vsjupyter.png" class="center">}}

## YOLOのセットアップ

今回は最新のYOLOを簡単に扱うことができるUltralyticsパッケージを利用します．

1. ```conda create --name yolo-env python=3.11```を実行して仮想環境を作成
2. ```conda activate yolo-env```で仮想環境を有効化
3. ```pip install ultralytics```でUltralyticsパッケージをインストール

ここまで済んだら，テストコードを動かしてみましょう！

```python
from ultralytics import YOLO

model = YOLO("yolo11n.pt")
results = model("https://ultralytics.com/images/bus.jpg")
results[0].show()
```

バスを背景にした画像の物体検出が表示されたはずです！

作業ディレクトリを開くと，```yolo11n.pt```というファイルがダウンロードされています．これが，YOLOのモデルファイルです．これはYOLOのデフォルトの訓練済みモデルで，COCO Datasetに収録された80種類の物体を識別することが出来ます．
