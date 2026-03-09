---
title: "LangChain vs LlamaIndex vs Haystack: 2026年本番RAGで2週間使い比べた"
emoji: "🚀"
type: "tech"
topics: ["rag", "langchain", "llamaindex", "haystack", "llm"]
published: true
---

3つ全部作った。2週間、同じRAGパイプラインを各フレームワークで実装した。

うちのチームは僕を含めて4人で、社内ドキュメント検索システムを本番に載せようとしていた。Confluenceのページが約8,000枚、日本語と英語が混在してるやつ。フレームワーク選定で「LangChainが一番メジャーだから」という声もあったけど、それだけで決めたくなかった。というか、2024年に別のプロジェクトでLangChainを使って、バージョン間の非互換で泣いた経験があったので。

だから比較した。これはその記録。

## セットアップ初日の解像度で見えてくること

LangChain（v0.3.x系）から始めた。ドキュメントは豊富だし、チュートリアルも大量にある。でも——最初にぶつかったのが「どれが今の正解な書き方か」問題だった。検索すると古いChainベースのコードとLCEL（LangChain Expression Language）ベースのコードが混在していて、Stack Overflowで見つけたコードをそのまま実行したら`DeprecationWarning`が5つ出た。それ自体は大した問題じゃないけど、混乱する。v0.1からv0.2、v0.3と続いた移行の傷跡があちこちに残ってる。

LlamaIndex（0.11系）の初日は対照的だった。コンセプトが一貫している——ドキュメント→ノード→インデックス→クエリ、この流れが全体を通してブレない。`VectorStoreIndex.from_documents(docs)`でインデックスを作って、`index.as_query_engine()`でエンジンを取って、`engine.query("質問")`。最初の動くものができるまでが早かった。

Haystackは——正直に言うと——最初「地味だな」と思った。deepset製で、ドキュメントは悪くないんだけど、LangChainのような熱量がない。GitHubのスター数も段違い。コミュニティの大きさでいえば比較にならない。

でも使い始めて数時間後に、この「地味さ」の意味が少しわかってきた。Haystackのパイプラインは、コンポーネントの入出力が型で明示的に定義されている。接続が間違っていたらすぐエラーが出る。曖昧さがない。最初は面倒に感じるけど、後々デバッグするときに効いてくる——これ、最初に見えるコストじゃない。

## 同じ質問20問を3つのフレームワークで投げた結果

検索品質を比較するために、20問を用意した。よくある質問から、複数ページにまたがる情報が必要なものまで。

LangChainのデフォルト設定（RecursiveCharacterTextSplitter、チャンクサイズ1000、オーバーラップ200）は、シンプルなQAには悪くない。でも複数ページにまたがるテーブルや箇条書きが途中で切れる問題が頻発した。特に仕様書系のドキュメントで回答精度が落ちた。改善しようとすると、カスタムのText Splitterを書くか、チャンクサイズをチューニングするしかない——つまり、デフォルトのままでは本番品質に至らない。

LlamaIndexは`SentenceSplitter`がデフォルトで入っていて、文脈をより賢く保持してくれる印象があった。20問の評価では14問でLlamaIndexの回答の方が「具体的だった」。ただ——これは正直なところ——うちのドキュメントが構造化されてるせいかもしれない。マークダウンのテーブルや見出し構造が多い仕様書が中心だったので、ユースケースによって差は変わると思う。

Haystackで予想外だったのが、ハイブリッド検索の実装コストの低さだった。BM25とベクトル検索を組み合わせたい場合、LangChainだと`EnsembleRetriever`を自分で設定する必要がある。LlamaIndexも`CustomRetriever`を実装する必要がある。でもHaystackでは:

```python
from haystack import Pipeline
from haystack.components.retrievers import InMemoryBM25Retriever, InMemoryEmbeddingRetriever
from haystack.components.joiners import DocumentJoiner

pipeline = Pipeline()
pipeline.add_component("bm25", InMemoryBM25Retriever(document_store=doc_store))
pipeline.add_component("embedding", InMemoryEmbeddingRetriever(document_store=doc_store))
pipeline.add_component("joiner", DocumentJoiner(join_mode="reciprocal_rank_fusion"))

pipeline.connect("bm25.documents", "joiner.documents")
pipeline.connect("embedding.documents", "joiner.documents")
# あとはpipeline.run()するだけ
```

これが動いたとき、「あ、考えて設計されてるな」と思った。日本語のように形態素解析が重要な言語では、BM25とベクトルのハイブリッドは特に有効で、その実装がこれだけシンプルなのは正直驚いた。

## 金曜午後にデプロイして月曜朝に壊れた話

これは自分の失敗談として書いておく。

LangChainベースの実装をステージングから本番に昇格させたのが金曜の午後。週末はアクセスが少ないので問題が見えなかった。月曜の朝にSlackで「検索が遅い」という報告が来た。P99レイテンシを見ると8秒を超えていた。平均でも3.5秒。普通に使えるレベルじゃない。

原因を追うと、埋め込みモデルの呼び出しとLLMの呼び出しが直列になっていた。LangChainのLCELは`RunnableParallel`を使えば並列実行できる。でもデフォルトでは直列で、知らないとハマる——というか僕がハマった。

```python
# やりがちな書き方（直列実行になる）
chain = retriever | prompt | llm | output_parser

# 並列で処理したい場合はこう書く
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

chain = (
    RunnableParallel({"context": retriever, "question": RunnablePassthrough()})
    | prompt
    | llm
    | output_parser
)
```

LlamaIndexはこの点でより自然に非同期に対応している。`engine = index.as_query_engine()`で取ったエンジンに対して`await engine.aquery("質問")`で非同期クエリができて、FastAPIのエンドポイントに素直に収まる。うちのバックエンドが全部asyncベースだったので、これが後の決め手のひとつになった。

Haystackのエラーメッセージについては——ここが正直一番言いにくいんだけど——コンポーネントの接続ミスで出るエラーが、原因がどこにあるのかすぐわからないことがある。パイプラインが複雑になってくると特に。`pipeline.draw()`で可視化できるのは便利だけど、Graphvizが必要で環境によってはそのセットアップが地味に面倒だった。

## LangSmithは便利、でもそれで全部解決するわけじゃない

本番RAGで見落とされがちなのが「なぜこの回答が出たのか」を後から追えるかどうかだ。どのチャンクが取得されたか、プロンプトの中身は何だったか、どこでどれだけ時間がかかったか——これが見えないと障害対応がつらい。

LangChainはLangSmithとのインテグレーションが便利だった。環境変数を2つ設定するだけでトレーシングが動く。

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls__your-api-key"
os.environ["LANGCHAIN_PROJECT"] = "rag-prod-2026"

# これだけで全チェーンの実行がトレースされる
result = chain.invoke({"question": "この仕様書の認証フローは?"})
```

UIも見やすいし、各ステップの実行時間やLLMへの入出力が全部確認できる。ただし有料。チームのサイズによっては月額コストが無視できなくなる。

LlamaIndexはArize PhoenixやOpenTelemetryと組み合わせるのが一般的で、ベンダーロックインを避けたいチームには向いてる。うちは最終的にArize Phoenixをセルフホストしてる。S3にトレースを吐いてGrafanaで見る構成にした。初期セットアップにはそれなりに時間がかかったけど、LangSmithのSaaSに依存したくなかった。正直、この選択は正解だったと思ってる。

Haystackは`Pipeline.run()`が詳細な中間結果を返すので、ログに吐けばある程度追える。でもそれだけで本番を見るのは厳しい。OpenTelemetryのサポートはあるけど、「すぐ使えるUI」はLangSmithほど整っていない——これはHaystackの明確な弱点。

## で、実際どれを選んだか

**うちの選択はLlamaIndexだった。**

理由は3点重なった。20問中14問で検索品質が良かった。非同期対応がFastAPIと素直に噛み合った。チームメンバーが読んでコンセプトを理解しやすかった。

LangChainはエコシステムが一番広い。これは事実で、否定しない。統合できるツールの数はダントツだし、エージェント系の機能はLangChainが一番充実してる。でも「RAGをちゃんと本番で動かす」という目的に絞ると、抽象化が多すぎて何が起きてるかわかりにくい場面が何度もあった。バージョン間の非互換の問題も継続してある。2024年に痛い目を見たのが、まだ少し頭に残ってる。

Haystackに関しては——これが一番意外な評価になるんだけど——本番での信頼性という意味では、3つの中で一番真剣に設計されてると思う。パイプラインの明示性、型付きコンポーネント、ハイブリッド検索のしやすさ。deepsetは企業向けSaaSも提供していて、本番で動かすことを前提として考えてるのがコード設計に出てる。チームが10人以上いてMLエンジニアが専任でいて、かつOSSで本番を動かしたいなら、Haystackは真剣に試す価値がある。

ただし——日本語の情報が圧倒的に少ない。詰まったときにすぐ答えが見つからない。4人チームで全員がHaystackに慣れていない状態で進めるのは、現実問題としてリスクだと判断した。これが決め手のひとつだった、というのは正直に言っておきたい。技術的な優劣だけで選べない場面は実際にある。

プロトタイプを早く作ってエコシステムを活用したいならLangChain、バックエンドがasyncでドキュメント検索のRAGを本番品質で動かすことが主目的ならLlamaIndex、大きめのチームで本番信頼性を最優先にするならHaystack——これが2026年3月時点での僕の結論。来月変わるかもしれないけど、今はこれが正直なところ。

<!-- Reviewed: 2026-03-10 | Status: ready_to_publish | Changes: tightened parallel paragraph structures; corrected LlamaIndex async API reference (index→engine.aquery); removed redundant sentence in LlamaIndex quality section; added one opinion line in Haystack conclusion; minor rhythm fixes throughout -->
