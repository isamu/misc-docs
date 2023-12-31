# SlashGPTの使い方

SlashGPTはコマンドラインで使うツールです。
コマンドラインを使っていくつかのツールを使いインストールをします。
SlashGPTの開発者はMacを使っているので、現時点ではMacでの利用を推奨しています

# 事前準備

- gitクライアント
- python
- pip

を入れておきます。

# Install

### ソースをダウンロード

```
git clone https://github.com/snakajima/SlashGPT.git
```

### パッケージのインストール

```
cd SlashGPT
pip install -r requirements.txt
```


### API KEYを環境変数に設定

.envファイルを作成して、OPENAI_API_KEYを設定する


(OPENAI_API_KEYは openaiのサイトから持ってきます。https://platform.openai.com/account/api-keys)

```.env
OPENAI_API_KEY=xxxx
```

### 起動

```
./SlashGPT.py 
```

```
System Slashes: /switch, /bye, /new, /prompt, /sample, /help, ...
Activating: Main Dispatcher
Dispatcher: I am a dispatcher agent. I will find the right agent for your question, and let it answer.
You(dispatcher):
```

と表示されたら、起動は成功です。


```
You(dispatcher): /autotest
```

と、 `/autotest` コマンドで自動テストを実行します。

実行して問題なければグラフが表示されます。

テストの後半は、各種LLMの設定が必要なためエラーとなります

