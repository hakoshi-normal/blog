---
title: 複数カメラの入力に対しYOLOv8で並列推論する
# publishdate: 2023-12-01
date: 2023-12-01
tags: [multiprocessing, Python, YOLOv8, 麻雀自動採譜]
toc: true
---

## 実装過程あれこれ１

これまでの作業でmultiprocessingモジュールを利用しプレーヤーの手牌を写す4台の
<a href="https://www.tp-link.com/jp/smart-home/tapo/tapo-c200/" target="_blank">Tapo C200</a>
からフレームを受信することができるようになりました．
<br>
早速各カメラに割り当てたプロセス毎に適当に学習させておいた
<a href="https://docs.ultralytics.com/ja/" target="_blank">YOLOv8</a>
による牌推論を実行し，結果を描画してみました．わくわく！

はい，映像出力が不穏な感じに．
{{< video src="before" >}}

卓の四隅にカメラを配置し，中央でお洒落カバーをバッサバッサしてます．

- プロセス間でまったく映像出力のタイミングがあっていない
- フレーム間隔がまちまち
- 映像の乱れが酷い
- FPSが低い（Tapo入力時15→10弱）

特に映像出力タイミングがあっていない状況だと自動採譜なんてとてもじゃないけど実現できないよね......<br>


フレーム出力がばらつく理由を勝手に予想．こんな感じだろうか

{{< figure src="multiprocess_fig1.jpg" title="">}}

出力結果を見る感じ，どうやらフレームに映り込んでいる牌が多いほど推論時間が増加してるっぽい．次のフレーム入力までに推論が追いつかず，プロセス間でフレーム出力に差が出ているように見える．
<br>  
<span style="font-size: 200%; color: red;">
こんなときこそマルチプロセス処理の出番でしょ↓</span>

{{< figure src="multiprocess_fig2.jpg" title="">}}
（推論プロセスをフレームごとに分散させれば安定した出力が可能だよね）

既にカメラごとにプロセスを分割しているため，その拡張のつもりで実装に取り掛かりました．設計は以下．

1. 映像受信プロセス ×4（カメラの台数）
2. 推論処理プロセス ×n×4（任意で設定可能，カメラごとに割当）
3. 推論結果統合プロセス ×4（カメラの台数）

さあやるぞ！

## 実装過程あれこれ２

ここで問題発生．フラグによるイベント発生伝達や事前に渡してある引数を処理に使うのはできそうだが，起動済みのプロセス1,2,3の間でフレームndarrayを引き渡す方法がわからない．<br>
base64→内部UDPが一瞬頭を過るが無視して
<a href="https://docs.python.org/ja/3.10/library/multiprocessing.html" target="_blank">公式ドキュメント</a>
を漁ってみます．

ドキュメントによると，python標準モジュールmultiprocessingにはメモリ共有やプロキシ経由でのオブジェクト操作をサポートする機能が幾つかあるんだとか．
Value, Array, Queue, Pipe, RawArray, Manager, shared_memory......

多い．更にここ4,5年のPythonアップデートでmultiprocessingの仕様が大幅に変わっており，どれを使えばいいのか分かりづらい．

試行錯誤の後，名前で共有メモリ領域を指定するshared_memoryを利用することに．
フラグと共有メモリ領域名を予め設定しておけば任意のタイミングで他プロセスが処理したndarrayを受け取れる．かなり便利ですね．

そんなこんなで推論のマルチプロセス処理を実装しました．映像出力がこちらになります．
{{< video src="after" >}}

概ね同じタイミングでフレームが描画されているように見えますね．<br>
FPSはいずれのカメラも15程度であり，入力時のFPSを保持したままリアルタイムで推論が実行できました．
<br>  
ちなみに，
- multiprocessingではなくthreadingを使ってYOLOv8のモデルをスレッド間で共有する方法があるが，提供元である
<a href="https://ultralytics.medium.com/object-tracking-across-multiple-streams-using-ultralytics-yolov8-7934618ddd2" target="_blank">UltralyticsのMedium記事</a>
では推奨方法，
<a href="https://docs.ultralytics.com/guides/yolo-thread-safe-inference/#non-thread-safe-example-multiple-model-instances" target="_blank">公式ドキュメント</a>
では非推奨方法として紹介されていたため今回は見送り．
- 試験的に使用してるYOLOv8モデルは，現環境（GeForce RTX 3070）で学習可能な最大サイズのモデル（YOLOv8l）に対して複数の公開データセットを統合した5万枚強の牌画像を学習させたものである．統合作業時にデータセット間でラベルのズレが発生したため，検出結果に誤りがある．ラベルが混合した状態のデータセットではあるが安定した検出ができているため，データセットの整理によって更に高精度な検出の実現が期待できる．
- 背景部分の予期せぬ牌検出は信頼度で棄却できるため問題なし．
- 検証動画は双方画面録画の仕様上FPSが低下している．

## ソースコード

複数台のカメラのフレーム入力に対してYOLOv8による推論を並列実行するPythonプログラム
ソースコード解説は<a href=" /blog/2023_12_26_multiprocessingforrealtimevideo/" target="_blank">こちら</a>からどうぞ．
```python
import time
import msvcrt
from multiprocessing import Process, Event, shared_memory
import numpy as np
import av
import cv2
from ultralytics import YOLO

# RTSPカメラ設定情報
info_list = [["toncamera", "*"],
            ["nancamera", "*"],
            ["shacamera", "*"],
            ["peicamera", "*"]]

password = "*"

# 1:1280x720, 2:640x360
mode = 1

# 推論プロセス数指定
N = 3

def rec_cam(cam_id, username, ipaddress, pre_flags, stop_flag, frame_info):
    url = f"rtsp://{username}:{password}@{ipaddress}:554/stream{mode}"
    container = av.open(url)
    counter = 0
    start = time.time()
    for frame in container.decode(video=0):
        counter+=1
        frame = frame.to_ndarray(format='bgr24')

        # 共有メモリにフレームを書き込み
        idx = (cam_id)*N+counter%N
        mem_name = f'shared{idx+1:02}'
        shm = shared_memory.SharedMemory(name=mem_name)
        frame_sh = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm.buf)
        frame_sh[:] = frame[:]
        # 推論開始
        pre_flags[idx].set()
        if stop_flag.is_set():
            t = time.time() - start
            print(f"{username}, TIME:{round(t, 3)}, FRAME:{counter}, FPS:{round(counter/t, 3)}")
            break
    container.close()
    cv2.destroyAllWindows()
    return

def finish_monitor(stop_flag):
    while True:
        if msvcrt.kbhit() and msvcrt.getch() == b'q':
            print("Finished")
            stop_flag.set()
            break
    return

def predict_frame(pre_flag, pre_fin_flag, mem_name, stop_flag, username, frame_info):
    model = YOLO('best.pt')
    while True:
        if pre_flag.is_set(): # 推論
            pre_flag.clear()
            shm = shared_memory.SharedMemory(name=mem_name)
            frame = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm.buf)
            # YOLOv8による推論実行
            result = model(frame)[0]
            frame = result.plot()
            # 結果統合用共有メモリにのせる
            shm_result = shared_memory.SharedMemory(name=username)
            frame_new = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm_result.buf)
            frame_new[:]=frame[:]
            # 結果統合用
            pre_fin_flag.set()
        if stop_flag.is_set():
            break

def view_frame(pre_fin_flag, username, stop_flag, frame_info):
    while True:
        if pre_fin_flag.is_set():
            pre_fin_flag.clear()
            shm = shared_memory.SharedMemory(name=username)
            frame = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm.buf)
            # Full-HDだと大抵画面が埋まっちゃう
            # frame = cv2.resize(frame, None, None, 0.6, 0.6)
            cv2.imshow(username, frame)
            cv2.waitKey(1)
        if stop_flag.is_set():
            break

if __name__ == '__main__':
    # メモリ確保情報
    url = f"rtsp://{info_list[0][0]}:{password}@{info_list[0][1]}:554/stream{mode}"
    container = av.open(url)
    for i, frame in enumerate(container.decode(video=0)):
        frame = frame.to_ndarray(format='bgr24')
        if frame is not None:
            break
    container.close()
    frame_info = {"nbytes" : frame.nbytes,
                  "shape" : frame.shape,
                  "dtype" : frame.dtype}

    # 停止プロセス
    stop_flag = Event()
    p = Process(target=finish_monitor, args=(stop_flag,))
    p.start()

    # 表示プロセス群（カメラ台数分）
    view_processes = []
    pre_fin_flags = []
    mem_space = []
    for i in range(4):
        pre_fin_flag = Event()
        pre_fin_flags.append(pre_fin_flag)
        username = info_list[i][0]
        # メモリ確保 今回は画像用
        shm = shared_memory.SharedMemory(create=True, size=frame_info["nbytes"], name = username)
        mem_space.append(shm)
        p = Process(target=view_frame, args=(pre_fin_flag, username, stop_flag, frame_info))
        view_processes.append(p)
        p.start()


    # 推論プロセス群（カメラ台数×推論プロセス数）
    pre_processes = []
    pre_flags = []
    mem_names = []
    for i in range(N*len(info_list)):
        pre_flag = Event()
        pre_flags.append(pre_flag)
        mem_name = f'shared{i+1:02}'
        mem_names.append(mem_name)
        # メモリ確保
        shm = shared_memory.SharedMemory(create=True, size=frame_info["nbytes"], name = mem_name)
        mem_space.append(shm)
        # frame_sh = np.ndarray(shape=frame_shape, dtype=frame_dtype, buffer=shm.buf)
        username = info_list[i//N][0]
        pre_fin_flag = pre_fin_flags[i//N]
        p = Process(target=predict_frame, args=(pre_flag, pre_fin_flag, mem_name, stop_flag, username, frame_info))
        pre_processes.append(p)
        p.start()

    # 撮影プロセス群（カメラ台数）
    rec_processes = []
    for cam_id, info in enumerate(info_list):
        p = Process(target=rec_cam, args=(cam_id, info[0], info[1], pre_flags, stop_flag, frame_info,))
        rec_processes.append(p)
        p.start()

    p.join()
    for p in view_processes:
        p.join()
    for p in pre_processes:
        p.join()
    for p in rec_processes:
        p.join()
    # 共有メモリのリソース開放、削除
    for mem_name in mem_names + [info[0] for info in info_list[:4]]:
        shm = shared_memory.SharedMemory(name = mem_name)
        shm.close()
        shm.unlink()

```
