---
title: "RTSP通信でカメラの映像を受信してみる"
# publishdate: 2023-12-01
date: 2023-12-01
tags: [RTSP, Python, OpenCV, PyAV, 麻雀自動採譜]
toc: true
---

## RTSPでフレーム取得（OpenCV）

RTSP通信でカメラの映像を受け取る際，OpenCVを使って記述できる．

```python
import cv2

url = "rtsp://username:password@ipaddress"
# Tapo C200は末尾に画質を指定
url = "rtsp://username:password@ipaddress/stream1"
cap = cv2.VideoCapture(url)

while True:
    ret, frame = cap.read()
    pass
```

ただこれだとTapo C200では**1秒程度遅延**が発生してしまう．
<a href="/blog/tags/麻雀自動採譜/" target="_blank">麻雀自動採譜</a>
の実装上，天井カメラを含めたカメラ間での同期が必須なんですよね．

## RTSPでフレーム取得（PyAV）

RTSP映像の受信にはFFmpegのPythonバインディングである
<a href="https://pyav.org/docs/develop/index.html" target="_blank">PyAV</a>
を利用した方法もあり、以下で映像受信が可能でした．

```python:
import av

url = "rtsp://username:password@ipaddress"
container = av.open(url)
for frame in container.decode(video=0):
    frame = frame.to_ndarray(format='bgr24')
    pass
```

PyAVを利用した場合は遅延が大幅に減少した．やったね．

## RTSPで一定時間毎にフレーム取得（PyAV）

<a href="/blog/tags/麻雀自動採譜/" target="_blank">麻雀自動採譜</a>
牌認識に向けて，学習画像収集のためn秒おきに画像を撮影してみた．いつものOpenCVと同じ感覚で`time.sleep(n)`を使用したところ以下のエラーが複数出現．

```raw:
max delay reached. need to consume packet
RTP: missed 6066 packets
RTP: PT=60: bad cseq a657 expected=8ea5
max delay reached. need to consume packet
```

sleepで無理に処理を中断させたため接続が不安定になったのかも．
通常の受信が安定していたことを踏まえ，とりあえず以下のコードで対応してみる．

```python
import av
interval = 10 # 10秒ごとに撮影
url = "rtsp://username:password@ipaddress"
container = av.open(url)
counter = 0
for frame in container.decode(video=0):
    if frame is not None and frame.time//interval>counter:
        counter+=1
        frame = frame.to_ndarray(format='bgr24')
        cv2.imwrite(f'save_{counter:04}.png', frame)
```

PyAVのcontainerから呼び出したフレームは撮影開始を0としたときの撮影時刻をtimeで取得できる．
撮影開始後数フレームはNoneが返ることがあるため注意．
これで安定したn秒毎の撮影が実行できる．
