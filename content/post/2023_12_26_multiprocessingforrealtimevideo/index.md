---
title: リアルタイムの映像入力に対してFPSを落とさずに画像処理したい
# publishdate: 2023-12-01
date: 2023-12-26
tags: [multiprocessing, Python, YOLOv8, 並列処理]
toc: true
---

## はじめに

<a href="/blog/2023_12_01_multiprocessingforyolov8/" target="_blank">複数カメラの入力に対しYOLOv8で並列推論する</a>のソースコード解説です．
解説のため，一部表現を変えている部分があります．

大まかな流れは以下の通り．

0. 共有メモリ確保
1. フレーム入力プロセス
    1. カメラからフレームを取得
    2. フレームを共有メモリ1に保存
2. 推論プロセス
    1. 共有フレームを呼び出し
    2. 推論
    3. 推論結果を描画したフレームを共有メモリ2に保存
3. フレーム表示プロセス
    1. 共有フレームを呼び出し
    2. フレームの画面描画

プロセス間の遷移はmultiprocessingモジュールのEvent関数をフラグとして利用します．
カメラの台数が増えた場合はプロセス数を増やせば対応できます．

## 事前準備
- 設定情報
```python
cam_N = 4 # カメラ台数
proccess_N = 3 # カメラ1台あたりのプロセス数
frame_info = {"nbytes" : frame.nbytes,
                "shape" : frame.shape,
                "dtype" : frame.dtype}
```
フレームのサイズや型はメモリ確保，呼び出し時に必要になります．

- 共有メモリの確保
```python
shm = shared_memory.SharedMemory(create=True, size=frame.nbytes, name = mem_name)
```

共有メモリの確保はこの一文でできます．めっちゃ便利ですね．
ただし，mem_nameを動的に作成して複数の共有メモリを確保する場合，注意事項があります．
shared_memory.SharedMemoryの戻り値を格納する変数（上記コードではshm）を設定しなかった場合や上書きした場合，それ以前の共有メモリ領域は参照できなくなります．その場合はshmを格納するリストを用意しておくとアクセス可能になります．

- プロセスの立ち上げ
```python
p = Process(target=関数名, args=(引数,))
p.start() # プロセス開始
p.join() # プロセス終了
```

複数プロセスを立ち上げる場合は p をリストに入れておくと管理が楽です．

## フレーム入力プロセス

```python
    def rec_cam(cam_id, pre_flags, frame_info):
        cap = cv2.VideoCapture(cam_id)
        while True:
            _, frame = cap.read()
            idx = (cam_id)*proccess_N+counter%proccess_N
            mem_name = f'shared{idx+1:02}'
            shm = shared_memory.SharedMemory(name=mem_name)
            frame_sh = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm.buf)
            frame_sh[:] = frame[:]
            # 推論開始
            pre_flags[idx].set()
        cap.close()
```

multiprocessingのSharedMemoryでは共有メモリを名前をつけて呼び出すことができます．便利！
ここでは，"shared1[番号]"という名前で共有メモリを確保している（後述）．カメラ台数✕カメラあたりのプロセス数分の共有メモリ領域が必要になるため，
あるカメラのあるフレームに対する保存領域は，カメラ番号✕カメラ1台あたりのプロセス数＋フレーム番号%カメラ1台あたりのプロセス数で求めることができる．と同時にこの番号は推論プロセスを動作させるフラグ管理のインデックスとしても使う．

## 推論プロセス

```python
def predict_frame(pre_flag, view_flag, mem_name, cam_id, frame_info):
    while True:
        if pre_flag.is_set(): # 推論開始
            pre_flag.clear()
            shm = shared_memory.SharedMemory(name=mem_name)
            frame = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm.buf)

            # 重い推論処理がここに入る（frameに結果を反映する）

            # 結果統合用共有メモリにのせる
            shm_result = shared_memory.SharedMemory(name=f'shared2{cam_id}')
            frame_new = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm_result.buf)
            frame_new[:]=frame[:]

            # 結果表示
            view_flag.set()
```

カメラ台数✕カメラあたりのプロセス数分の推論プロセスを立ち上げます．
推論開始用のフラグや共有メモリの呼び出し名は予め与えた状態でプロセスを立ち上げる．フレーム入力プロセスでの推論開始用フラグの有効化を待ち受けています．
処理済みのフレーを格納する共有メモリ領域名は'shared2[番号]'としており，カメラの台数分事前に確保してあります．
検出結果を描画する必要がない場合は，フレーム表示プロセスを介する必要がないため，別プロセスに推論結果を渡したり，UDP通信などで端末間の結果送信が実装できると思います．

## フレーム表示プロセス

```python
def view_frame(view_flag, cam_id, frame_info):
    while True:
        if view_flag.is_set():
            view_flag.clear()
            shm = shared_memory.SharedMemory(name=f'shared2{cam_id}')
            frame = np.ndarray(shape=frame_info["shape"], dtype=frame_info["dtype"], buffer=shm.buf)
            cv2.imshow(f'window_{cam_id}', frame)
            cv2.waitKey(1)
```

推論プロセスでのフレーム表示用フラグの有効化を待ち受けています．

## おわりに

今回は，以前実装した並列画像処理プログラムの解説でした．コードブロックを断片的に示しただけなので，具体的な実装コードを知りたい方は<a href="/blog/2023_12_01_multiprocessingforyolov8/#ソースコード" target="_blank">ソースコード</a>から参照できます．
