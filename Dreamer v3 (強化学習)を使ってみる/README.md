https://qiita.com/0108mitaka/items/5209514a08bd0c71ad79

# Dreamer v3 (強化学習)を使ってみる

# はじめに

この記事は強化学習の初心者が、とりあえず Dreamer-v3 を回せるようにしたことの備忘録です。理論部分は書きません。

dreamer v3 に関する日本語記事が全然なかったので、とりあえず記事を書くことにします。英語が読める人は本家の github に行ってください

> https://github.com/danijar/dreamerv3

筆者は機械学習初心者なので説明が間違っている可能性があります……

## dreamer v3 とは？

世界モデルの新しめ(2023 年初頭現在)のものです。
マイクラをプレイしてダイヤモンドを取得できる AI としても紹介されている。

### 世界モデルとは

世界モデルでは強化学習をするにあたって、環境をまず学習し、学習した環境の中で(ちょうど頭の中で夢を見るように)操作の強化学習をする。その後実際の環境でその操作を実行する。

基本的な理論は dreamer や dreamer-v2 などの記事に書かれているものと同じなので、そちらをググれば良いと思います。

# 概要(まとめ)

`example.py`を実行するだけで、好みの`gym`環境に dreamer-v3 を適用できます
GPU 開発環境周りの構築をして、ライブラリのインストール周りも少し変更しています
crafter を実行した結果、12 時間程度で有名な手法 PPO のスコアとして知られているスコア 5 を超えました。

# 本編

筆者は Python 3.10 を使用しますが、これは 3.11 ではエラーが出たためです。他のバージョンは知りません。

> https://github.com/danijar/dreamerv3

これが本家リポジトリ。
とりあえず動かすのに必要なのは`example.py`です。

必要なライブラリは

```
pip install dreamerv3
```

でインストールできます。(後述する手順を踏まないとインストールできないかもしれません)

`jax`ライブラリが Windows で使用できないようですが、windows ユーザーでも`jax`を使えるようになる神プロジェクトがありました。ありがとうございます有志の人。

> https://github.com/cloudhan/jax-windows-builder

エラーが出た場合は Python のバージョンを確認して、エラーメッセージと照らし合わせてください。機械学習系のライブラリは古いことが多いので、最新版では実行できないことが多いです。

## インストール

さて、jax ではインストール時に CUDA(GPU の開発環境)のバージョンを指定する必要があるので、先に GPU の開発環境を構築します。

### GPU 環境のインストール

他のライブラリ(jax)がついてこれないので、11.8 をインストールします(11.x ならいいはず)

> https://developer.nvidia.com/cuda-11-8-0-download-archive?target_os=Windows&target_arch=x86_64&target_version=10&target_type=exe_local

バージョンを指定すると、windows ならインストーラーを用意してくれます。やさしい
Linux 版でもコマンドを見せてくれるので従うだけ

`nvcc -V`でインストールが成功しているか確認しましょう

お次は cuDNN

> https://developer.nvidia.com/rdp/cudnn-download

`nvcc -V`で確認したバージョンにあったものをインストールします
zip を解凍したら、CUDA のディレクトリ
(デフォルト`C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8`)
にコピペ(bin が上書きされるように)

PATH も`CUDNN_PATH`という名前で`C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8`に通します

### ライブラリのインストール

では、`pip install dreamerv3`でインストールしてみましょう……

```
Collecting gym==0.19.0
  Using cached gym-0.19.0.tar.gz (1.6 MB)
  Preparing metadata (setup.py) ... error
  error: subprocess-exited-with-error

  × python setup.py egg_info did not run successfully.
  │ exit code: 1
  ╰─> [1 lines of output]
      error in gym setup command: 'extras_require' must be a dictionary whose values are strings or lists of strings containing valid project/version requirement specifiers.
      [end of output]
```

gym のインストールでエラーが発生しました。バージョンが指定されているようですが、バージョンが古すぎます。
ここは悪いことが起こらないことを祈って、gym のバージョン指定を消しちゃいましょう。

dreamer v3 から gym のバージョン指定を抜いて、ローカルでビルドします。

> https://github.com/danijar/dreamerv3

を clone して、`requirements.txt`内の`gym==0.19.0`を`gym`に書き換えてしまいます。
その後

```
python setup.py sdist
pip install dist/dreamerv3-1.5.0.tar.gz
```

をして、ローカル版 dreamerv3 をインストールします。
……

```
ERROR: Could not find a version that satisfies the requirement jaxlib (from dreamerv3) (from versions: none)
ERROR: No matching distribution found for jaxlib
```

そうです、windows に jax は対応していません。(今のところは。)

jax を windows に対応させてくれるプロジェクトを使い、jax は別で先に入れてしまいます。

> https://github.com/cloudhan/jax-windows-builder

`nvcc -V`で出てくる CUDA のバージョンを確認します。
そして、次のコマンドで jax, jaxlib をインストールします。(CUDA が 11.x なら cuda11.cudnn82 を入力)
jax が最新版だと対応する jaxlib がないので、バージョンも指定。

```
pip install jax==0.3.25 jaxlib==0.3.25+cuda11.cudnn82 -f https://whls.blob.core.windows.net/unstable/index.html --use-deprecated legacy-resolver
```

では、お待ちかねの dreamerv3 を入れます。

```
python setup.py sdist
pip install dist/dreamerv3-1.5.0.tar.gz
```

通った！

# 実行

> https://github.com/danijar/dreamerv3

から`example.py`と`configs.yaml`を取ってきて、同じディレクトリに配置します。

```
python example.py
```

`--jax.platform cpu`など、オプションも色々あるようです。これは CPU で実行するオプションですが、CPU 使用率が無事 100%になったのでおすすめしません。(それはそう)

```
Could not locate zlibwapi.dll. Please make sure it is in your library path!
```

と怒られたので、

> https://docs.nvidia.com/deeplearning/cudnn/install-guide/index.html#install-zlib-windows

から`ZLIB DLL`で zip を落としてきて、`zlib123dllx64\dll_x64`にある`zlibwapi.dll`を PATH が通っている所に置きます。

今度は問題なく動いた。(なおメモリが足りなくて warning が出た)

途中経過の保存ファイルなど、色々 config で指定できる。configs.yaml に色々なタスク用の設定も書いてあるので面白い(minecraft もある)

# 結果

100 万ステップほど回した結果、crafter というデフォルト設定されているタスク(2D のマイクラのようなもの)で大体 6±1 くらいのスコア(実績解除)を取るようになりました。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/31bd8d82-03c9-32e9-882c-0641ef43dc55.png)
https://github.com/danijar/crafter より

スコア 6 は PPO(優れたパフォーマンスと実装の簡単さが売り)が頑張っても取れないレベルのスコアなので、これはすごい。

私の PC のメモリは 16GB ですが、他の作業をしていたのでコマンドプロセッサは平均 9GB ほどのメモリしか使えていませんでした。100 万ステップに 12 時間ほどかかったのでこれ以上は回しませんでしたが、開発元によると 2000 万ステップほど回すことで 12~15 ほどのスコアを出すことができるようです。

学習を中断して終了してしまったので、スコア-ステップのグラフのようなものは出せません。申し訳ありません。

# さいご

論文を読む限りでは、dreamer v2 との相違点は最適化やチューニング周りのようで、アルゴリズム的に大きな違いはないそうです。(それでも性能は大きく向上しているのですごい)

minecraft でダイヤを採掘するような複雑なタスクを 0 から学習することができるほどの性能らしいので、ちょっとした市販ゲームなら学習させられそうです。個人の PC では minecraft ほどのものは厳しそうですが……。
