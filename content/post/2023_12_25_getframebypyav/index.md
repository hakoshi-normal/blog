---
title: "映像入力の遅延をPyAVで解決してみる"
# publishdate: 2023-12-01
date: 2023-12-25
tags: [Python, OpenCV, PyAV, HDMI, 麻雀自動採譜]
toc: true
---

## ビデオカメラからOpenCVで映像取得（低品質）

<a href="/blog/tags/麻雀自動採譜/" target="_blank">麻雀自動採譜</a>の実装において，卓上を撮影する天カメとしてFull HDで映像記録が可能な家庭用ビデオカメラを取り付けた．

ビデオカメラにはHDMIの出力端子があり，HDMI to USB Cに変換できる[ビデオキャプチャカード](https://amzn.asia/d/jclJASJ)を導入．
これでビデオカメラをWebカメラみたく使えるぞ！やった！！

と思った矢先，

**なんか遅い気がする！！**

よくあるOpenCVでWebカメラの映像を取得するプログラムを走らせると，**1秒遅延＋低FPS＋低画質**という散々な結果に

お前 Full HD で 30FPS 出るって言ってたじゃないか......

キャプチャカードが悪いのか，もともとそんな高品質は無理なのか，いろいろ考えた挙げ句，似た症状を以前どっかで見たのを思い出しました．

[RTSP通信でカメラの映像を受信してみる](/blog/2023_12_01_getframertsp/)

このときもRTSP通信による映像入力がOpenCVでは遅延＆低FPSだった．
今回も<a href="https://pyav.org/docs/develop/index.html" target="_blank">PyAV</a>を使えば解決するかもしれない．

## ビデオカメラからPyAVで映像取得

PyAVでPCに有線接続されたカメラから映像を取得する．
```python
import cv2
import av

con_options  =  dict(
    video_size='1920x1080',
    vcodec='mjpeg',
    framerate='30',
    rtbufsize='1',
)

device_name  =  "USB Video"

con_def  =  dict(
    format='dshow', 
    file=f'video={device_name}', 
    options=dict(con_options, video_device_number='0') )

container = av.open(**con_def)

for frame in container.decode(video=0):
    frame = frame.to_ndarray(format='bgr24')
    # cv2.imshow("test_window", frame)
    # cv2.waitKey(1)
```

ちなみにカメラデバイス名はffmpegをインストールしてるなら以下のコマンドで確認できる．
```console
ffmpeg -list_devices true -f dshow -i dummy
```
PyAVはFFmpegのPythonバインディングらしいので，デバイス名の自動取得も可能かもしれない．subprocessモジュールを使えば上のコマンドでも可能．

これで遅延ほぼなし，30FPSでの映像受信ができた．
