---
title: 【入門3】YOLOアノテーションと学習
# publishdate: 2023-12-01
date: 2025-05-06
tags: [YOLO, アノテーション, labelme, YOLO入門]
toc: true
---

## アノテーションとは？

機械学習の分類のひとつである**教師あり学習**では，まず入力データと出力データの組を複数用意し，それらを使って機械学習モデルを訓練します．

入力データと出力データの組は，タスクによって様々です．例えば，

- 入力：画像データ，出力：それぞれが犬or猫
- 入力：家の情報（間取り，立地，築年数など），出力：家の価格
- 入力：これまでの気温，出力：明日の気温
- 入力：画像データ，出力：各物体の座標

どんなタスクを前提とする場合でも，これら**入力データと出力データの組を作る作業**が必要となります．これが**アノテーション**です．

## 環境構築

今回はYOLOによる画像物体認識を目標とします．そこで，アノテーションで作るべきデータは以下になります．

- 入力：画像データ，出力：各物体の座標

頑張れば用意した画像の座標を数えてアノテーションできそうですが，今回はより簡単に作業を進めるために**Labelme**というツールを使います．GUIが整備されたソフトウェアですが，Pythonライブラリとして無償で提供されています！（ありがたい！！）

1. Anaconda Promptを開く
2. ```conda create -n annot_env python=3.12```で仮想環境を作成
3. ```conda activate annot_env```で仮想環境を有効化
4. ```pip install --upgrade labelme```と入力しLabelmeをインストール

これでインストールは完了です．そのまま```annot_env```環境内で```labelme```コマンドを実行してみてください！

{{< figure src="labelme.png" class="center">}}

この画面が表示されたら成功です！

## いざ，アノテーション

### 画像読み込み

今回は画像から麻雀牌を物体認識するタスクに挑戦してみます！サンプル画像は以下からダウンロード，解凍してください．

- [サンプル画像](https://github.com/hakoshi-normal/yolo_sample/raw/refs/heads/main/images/images.zip)

Zipファイルの解凍が済んだらLabelmeの画面左側にある```Open Dir```から，画像のディレクトリを開いてください．

{{< figure src="labelme_start.png" class="center">}}

これでアノテーションの準備は完了です！

### 範囲選択，ラベル付け

アノテーションの流れは以下になります．

1. 画像を右クリック，```Create Rectangle```をクリック
2. 物体の範囲を四角形で囲んで選択，間違えたら```Esc```キーで戻る
3. 出てきたポップアップの上部```Enter object label```にラベルを入力して```OK```をクリック
4. 2と3を物体の数だけひたすら繰り返す
5. 画像内のアノテーションが終わったら画面左部```Save```から画像と同名のJSONファイルを保存

ちょっとやってみますね～

{{< figure src="annotated.png" class="center">}}

はい，できました．ラベルは牌の種類（萬子筒子索子字牌）の略称です．

......

............

..................

<span style="font-size: 150%; color: red;">めんどくさい</span>

機械学習では一般的にデータの数が大きいほど性能が上がりやすいとされています．つまり，（要件の要求レベルにもよりますが）精度を上げるためにはたくさんアノテーションする必要があるんです．

そこで，特定のタスクの機械学習を考える場合はアノテーションに取り掛かる前に，データセットが利用可能なライセンスで公開されていないかどうか調べるところからスタートします．例えばYOLOの場合，[Roboflow](https://universe.roboflow.com/)，[VisualData](https://visualdata.io/discovery)，[Google Open Images Dataset](https://storage.googleapis.com/openimages/web/index.html)などが画像データセットを公開している他，タスクに応じて論文と併せて公開されているデータセットも多数あります．

また今回はアノテーション入門ということで手作業での方法を紹介していますが，画像認識AIによる半自動ツールもあるので興味があれば調べてみてください．

**今回は，なんと，特別に，アノテーション済みファイルを用意してあります！**

- [アノテーション結果（JSON）](https://github.com/hakoshi-normal/yolo_sample/raw/refs/heads/main/images/labels.zip)

ZIPファイルをダウンロード＆解凍したら，```datasets```という名前のディレクトリを作成し，先程の画像ファイルとJSONファイルを以下の構造で配置してください．（```image_1.json```は自作のファイルで構いません！）

```planetext
.
└── datasets/
    ├── image_1.jpg
    ├── image_1.json
    ├── image_2.jpg
    ├── image_2.json
    ├── 〜省略〜
    ├── image_10.jpg
    └── image_10.json
```

### データセットフォーマットの変換

では，学習させるデータの中身を一度見てみましょう！```image_1.json```を開いてみます．

{{< figure src="label_1.png" class="center">}}

JSONはデータ記述形式のひとつで，キーと値を並べて書くのが特徴です．Pythonの辞書型そっくりですね．
ラベルや座標がファイルの中に複数記述されていることがわかります．

{{< figure src="label_2.png" class="center">}}

少しスクロールすると，よくわからない文字列が大量に記述されています．実はこれ，Base64と呼ばれるエンコード形式で画像そのものが書きつけられているんです．これにより，このJSONファイル単体で入力データを表現することが出来ます．

さて，ひととおり学習データの中身を見てもらいましたが，実はこのままではYOLOの学習が出来ないんです．

YOLOに学習させるためには，画像のアノテーションを以下の形式で記述する必要があります．

```画像ファイル名.txt```

```planetext
label_id X Y Width Height
```

例えば，```image_1.json```をこの形式で記述すると以下のようになります．

```image_1.txt```

```planetext
4 0.171569 0.375817 0.142157 0.183007
6 0.286765 0.382353 0.124183 0.183007
0 0.401144 0.398693 0.127451 0.185185
5 0.517157 0.408497 0.117647 0.178649
10 0.638889 0.418301 0.125817 0.189542
7 0.767157 0.436819 0.127451 0.174292
```

YOLOのデータセットでは，ラベルの番号，X座標，Y座標，幅，高さを列挙して記述します．

これを手作業で変換するのはかなり面倒です．

そこで今回は，Labelmeで保存したJSONファイルをYOLOのデータセット形式に変換するツール，**labelme2yolo**を使います．condaの```annot_env```環境下で以下の手順で変換します．

1. ```pip install labelme2yolo```でlabelme2yoloをインストール
2. ```dataset```ディレクトリの一つ上の階層で以下のコマンドを実行
   ```labelme2yolo --json_dir ./datasets/ --val_size 0.10 --output_format "bbox"```

これが済んだら，```datasets```ディレクトリの中に```YOLODataset```が生成されているはずです．構造は以下のようになっています．

```planetext
.
└── YOLODatasets/
    ├── images/
    │   ├── train/
    │   │   ├── image_1.jpg
    │   │   └── ～省略～
    │   └── val/
    │       └── image_7.jpg
    ├── labels/
    │   ├── train/
    │   │   ├── image_1.txt
    │   │   └── ～省略～
    │   └── val/
    │       └── image_7.txt
    └── dataset.yaml
```

先ほどのデータセット構造より複雑になっていますね．画像とラベルが別のフォルダに格納され，更にその中で訓練用(```train```)と評価用(```val```)に分かれています．

```labels```ディレクトリのテキストファイルを見てみましょう．YOLOのフォーマットに変換され，ラベルIDや座標が記録されているはずです．

```dataset.yaml```の中身を見てみましょう．

{{< figure src="datasetsetting.png" class="center">}}

データセットまでの絶対パス，訓練評価用ディレクトリのパス，ラベルIDとラベル名の対応が記載されています．YOLOの学習時にはこの```dataset.yaml```を利用してデータセットを読み込みます．

## YOLOの学習

それでは，いよいよ作ったデータセットを使ってYOLOの学習をしてみましょう！！

まず，テスト用の画像を用意します．以下のリンクから画像ファイル```image_11.jpg```をダウンロードして```dataset.yaml```と同階層に配置します．

- [テスト画像image_11.jpg](https://github.com/hakoshi-normal/yolo_sample/raw/refs/heads/main/images/image_11.jpg)

```dataset.yaml```と同階層に```.ipynb```ファイルを作成して，以下のPythonプログラムを実行してみましょう．

```python
from ultralytics import YOLO

model = YOLO("yolo11n.pt")
model.train(data='dataset.yaml', epochs=300, verbose=False)
results = model("image_11.jpg")
results[0].show()
```

CPUのみの実行環境ではかなり時間がかかるので気長に待ちましょう～

学習が終了すると，```image_11.jpg```に対する推論結果が表示されます．

{{< figure src="detectresult.png" class="center">}}

未検出や誤検出がちらほらありますが，部分的に検出が成功していることがわかります．データセットサイズやエポック数を変えることで性能の改善が期待できます．

また，データセットディレクトリの```runs```内を見ると，```weight```ディレクトリに```best.pt```が保存されていることがわかります．
これは今回学習したモデルファイルで，実際に物体検出をシステム開発に組み込むときに使用します．

{{< figure src="ptfile.png" class="center">}}
