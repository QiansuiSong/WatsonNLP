# スタンドアロン コンテナーを使用して事前トレーニング済みのモデルでセンチメントを分析する
## 前提条件

[Docker デスクトップ](https://docs.docker.com/get-docker/)

[Python 3.9](https://www.python.org/downloads/) 以降

[container software libraryからエンタイトルメント キー](https://myibm.ibm.com/products-services/containerlibrary)を取得

[Watson NLP Runtimeおよび事前トレーニング済みモデルへの Docker アクセス](https://github.com/ibm-build-lab/Watson-NLP/tree/main/MLOps/access#docker)

[Watson NLP Runtime Python client library](https://github.com/ibm-build-lab/Watson-NLP/tree/main/MLOps/access#python)

[gRPC](https://grpc.io/docs/languages/python/quickstart/)
    

## ステップ1. サンプルコードを入手する
サンプルコードをGitHub リポジトリから複製します。

```
git clone https://github.com/ibm-build-labs/Watson-NLP
```

このチュートリアルで使用するサンプル コードは、Watson-NLP/MLOps/Watson-NLP-Container の下にあります。
## ステップ2. サービスを構築する
1. 次のコマンドを実行します。

```
cd Watson-NLP/MLOps/Watson-NLP-Container/Runtime
```

2. このディレクトリ内の Dockerfile はサービスの構築に使用されます。

```ARG WATSON_RUNTIME_BASE="cp.icr.io/cp/ai/watson-nlp-runtime:1.0.18"
ARG SENTIMENT_MODEL="cp.icr.io/cp/ai/watson-nlp_sentiment_aggregated-cnn-workflow_lang_en_stock:1.0.6"
ARG EMOTION_MODEL="cp.icr.io/cp/ai/watson-nlp_classification_ensemble-workflow_lang_en_tone-stock:1.0.6"

FROM ${SENTIMENT_MODEL} as model1
RUN ./unpack_model.sh

FROM ${EMOTION_MODEL} as model2
RUN ./unpack_model.sh

FROM ${WATSON_RUNTIME_BASE} as release

RUN true && \
    mkdir -p /app/models

ENV LOCAL_MODELS_DIR=/app/models
COPY --from=model1 app/models /app/models
COPY --from=model2 app/models /app/models
```
事前トレーニング済みの Watson NLP モデルはコンテナー・イメージとして保管されます。これらのコンテナーが正常に実行されると、unpack_model.sh スクリプトが呼び出されます。スクリプトはモデル ファイルを抽出し、コンテナーのファイル システム内の /app/models ディレクトリにコピーします。

この Dockerfile で unpack_model.sh を実行して、ビルド中に解凍をトリガーします。

生成される最終的なイメージは、Watson NLP ランタイム イメージをベース イメージとして使用します。 LOCAL_MODELS_DIR 環境変数は、Watson NLP ランタイムに、サービスを提供するモデルの場所を伝えるために使用されます。モデル ファイルは、中間イメージから最終イメージの /app/models にコピーされます。

**日本語のモデルを使う場合、ARG SENTIMENT_MODEL変数を「cp.icr.io/cp/ai/watson-nlp_sentiment_aggregated-cnn-workflow_lang_ja_stock:1.0.6」に編集します。**

3. 次のコマンドでイメージをビルドします。

```
docker build . -t watson-nlp-container:v1
```
これにより、watson-nlp-container:v1 という Docker イメージが作成されます。


## ステップ 3. サービスを起動する
次のコマンドを使用して、ローカル マシンでサービスを開始します。
```
docker run -e ACCEPT_LICENSE=true -d -p 8085:8085 watson-nlp-container:v1
```
モデルはポート 8085 の gRPC エンドポイントを介して提供されています。


## ステップ 4. サービスをテストする
Python クライアント プログラムを使用してサービスをテストできます。
1. Watson NLP Python クライアント ライブラリーがインストールされていることが必要です。

```
cd ../Client
```

クライアント コードは、感情分析および感情分類モデルを利用するために作成されています。異なるモデルを利用する場合は、クライアント コードを更新する必要があります。

2. クライアント コマンドを実行します。

```
python3 client.py "Watson NLP is awesome"
```

3. 次の例のような出力が表示されます。

```classes {
  class_name: "joy"
  confidence: 0.9687168002128601
}
classes {
  class_name: "anger"
  confidence: 0.03973544389009476
}
classes {
  class_name: "fear"
  confidence: 0.030667975544929504
}
classes {
  class_name: "sadness"
  confidence: 0.016257189214229584
}
classes {
  class_name: "disgust"
  confidence: 0.0033179237507283688
}
producer_id {
  name: "Voting based Ensemble"
  version: "0.0.1"
}

score: 0.9761080145835876
label: SENT_POSITIVE
sentiment_mentions {
  span {
    end: 21
    text: "Watson NLP is awesome"
  }
  score: 0.9761080145835876
  label: SENT_POSITIVE
}
producer_id {
  name: "Document CNN Sentiment"
  version: "0.0.1"
}
```

ポート 8080 を公開すると、[こちら](https://github.ibm.com/EmbeddableAI-JP/work/blob/main/nlp/client/nlp_sentiment.html)のサンプルアプリを使ってテストできます。

```
docker run -e ACCEPT_LICENSE=true -d -p 8080:8080 watson-nlp-container:v1
```

試す時にSafariを使って、下記のように設定してください。

　・設定>詳細>メニューバーに開発メニューを表示
 
　・開発>クロスオリジンの制限を無効にする にチェック
 
　・Safari で上記に nlp_sentiment.html を開く
