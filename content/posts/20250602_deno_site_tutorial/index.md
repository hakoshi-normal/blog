---
title: Denoを使ったWebサイト作成入門
# publishdate: 2023-12-01
date: 2025-05-06
tags: [Deno]
UseHugoToc: true
ShowToc: true
tocopen: true
---

## Denoのローカルインストール

1. DenoのGitHub[リリースページ](https://github.com/denoland/deno/releases)からインストーラの圧縮ファイル```deno-x86_64-pc-windows-msvc.zip```をダウンロード
2. ZIPファイルを解凍し，```deno.exe```を確認
3. ```C:\Program Files```に```Deno```ディレクトリを作成
4. ```C:\Program Files\Deno```に```deno.exe```を移動
5. ```Win + S```→「環境変数を編集」を開く
6. 「〇◯のユーザー環境変数」から```Path```を選択し，編集をクリック
7. 「新規」をクリックし，```C:\Program Files\Deno```を追加する．「OK」で終了
8. コマンドプロンプトを開き```deno -V```と入力しEnterキーを押す．バージョンが確認できたらインストール成功


## 最小構成でサイトを作る！

Denoを使ってHTMLを配信する場合，最小構成は次のようになります．

```plane
.
└── my_site/
    ├── server.ts
    └── public/
        └── index.html
```

server.tsとindex.htmlには以下のように記述してください．

```server.ts```
```TypeScript
import { serveDir } from "https://deno.land/std@0.224.0/http/file_server.ts";

Deno.serve(async (req) => {

  return serveDir(req, {
    fsRoot: "public",
    urlRoot: "",
    showDirListing: true,
    enableCors: true,
  });

});
```

```index.html```
```html
Hello World!
```

## サーバーを立ち上げる！

1. コマンドプロンプトを起動し，カレントディレクトリを```my_site```に移動
   （```cd my_siteまでのパス```でも移動できる他，エクスプローラで```my_site```を開きアドレスバーに```cmd```+Enterで開くことができる）
2. ```deno run server.ts```と入力しEnter
3．ネットワークアクセスの許可が確認されるので，```y```を入力して許可
4. 表示されるURL（```http://localhost:8000```など）にブラウザからアクセスし，Hello World!ページを確認

Denoにはサーバ起動時に便利なコマンドフラグが用意されています．
- ```--watch```：ファイル変更時のオートリロード
- ```--allow-net```：ネットワークアクセスの許可
- ```--allow-read```：Webからのプログラムのインポート許可
- ```--allow-env```：環境変数へのアクセス許可

以下のように実行すると許可入力の手間を省けます．便利！！

```deno run --watch --allow-net --allow-read --allow-env server.ts```



## 公開してみる！！

### GitHub，リポジトリの準備

1. [GitHub](https://github.co.jp/)のアカウントを作成，ログインする
2. 新しいリポジトリを作る
　1. [ここ](https://github.com/new)にアクセス
  2. ```Repository name```にリポジトリ名をつける（今回はMySiteとか）
  3. ```Create repository```でリポジトリを作成

### GitHub Desktopで公開してみる











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

## 参考
[https://docs.ultralytics.com/ja/quickstart/#understanding-settings](https://docs.ultralytics.com/ja/quickstart/#understanding-settings)
