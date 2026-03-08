---
title: "RAGの深掘り: チャンキング戦略・ベクターDB・検索最適化完全ガイド"
emoji: "🚀"
type: "tech"
topics: ["rag", "vector-database", "chunking", "llm", "python"]
published: true
---

半年前、うちのチーム（4人）で社内ナレッジベースのRAGシステムを作っていた。対象はConfluenceのドキュメント数千件とGitHubのREADME群。最初の実装は2日で動いた。精度は悲惨だった。

「チャンク1000トークン、オーバーラップ200、Chromaでいいでしょ」というノリで進めたら、ユーザーから「全然的外れな回答が返ってくる」という報告が山のように来た。そこから2週間、ひたすらチャンキング・ベクターDB・検索パイプラインを試し続けた。その記録を書く。

---

## チャンキングは「大きさ」じゃなくて「意味の境界」が全て

最初に気づいたのは、固定サイズチャンキング（fixed-size chunking）の根本的な問題だった。1000トークンで機械的に切ると、コードブロックの途中で分割されたり、Markdownの表が半分になったりする。LLMに渡されるコンテキストがそもそも欠けているのだから、まともな回答が出るわけがない。

試したアプローチを整理すると——

**Fixed-size chunking**は実装が一番楽。ただ「意味の途中で切れる」問題は根本的に避けられない。500〜1000トークンが一般的だが、技術ドキュメントと社内FAQでは最適値が全然違う——同じサイズで処理しようとしている時点で、設計がずれている。

**Recursive character splitting**（LangChainの`RecursiveCharacterTextSplitter`が代表格）は段落→文→単語の順で分割点を探す。固定サイズより明らかにマシで、最初の改善として試す価値は十分ある。

**Semantic chunking**は文の埋め込みベクトルの距離が急変する地点で分割する手法。LlamaIndexやlangchain-experimentalに実装がある。精度は確かに上がる。ただしドキュメント数が多いと埋め込みコストが跳ね上がるので、試算してから導入を判断すること（うちは一度試算して少し引いた）。

**Hierarchical chunking（Parent-Child）**——今のところ一番気に入っている。

Hierarchicalの考え方はシンプルで、親チャンク（大きい）と子チャンク（小さい）を別々に管理する。検索は子チャンクで行い、LLMには親チャンクを渡す。

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# 子チャンク: 検索精度のために小さく（意味的凝集性を高める）
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

# 親チャンク: LLMに渡すコンテキストのために大きく
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

store = InMemoryStore()
vectorstore = Chroma(
    collection_name="documents",
    embedding_function=OpenAIEmbeddings()
)

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# 追加すると親と子が自動で管理される
retriever.add_documents(docs)

# 検索は子チャンクで行われ、返ってくるのは親チャンク
results = retriever.invoke("AWS S3のバケット命名規則は？")
```

この構造の何がいいかというと、検索は意味的に凝集した小さいチャンクで行えて、LLMに渡すのは文脈が豊富な大きいチャンクになる。Recall（関連ドキュメントをちゃんと拾えるか）とContext Quality（LLMが使えるコンテキストか）のトレードオフを両立できる。

やらかした話をひとつ。オーバーラップを大きくしすぎた。400トークンのチャンクに200トークンのオーバーラップをつけたら、同じ内容が複数チャンクに重複して入り込み、検索結果の上位がほぼ同じ内容のチャンクで埋まるという事態になった。オーバーラップは10〜15%くらいが無難だと思う。

---

## ベクターDB: 「とりあえずPinecone」が正解とは限らない

ベクターDB選択で迷っている人は多いと思う。自分も迷ったし、正直今も完全には決着がついていない部分がある。ただ実際に試した感想をシェアする。

**Chroma**はローカル開発やプロトタイプには最高で、セットアップが一番楽。ただ100万件を超えたあたりから怪しくなるので、スケールが見えているプロジェクトには向かない。

**Qdrant**——個人的に一番気に入っている。RustベースなのでCPUバウンドな処理が速く、ペイロード（メタデータ）に対するフィルタリングが柔軟。v1.8.0からSparse vectorのネイティブサポートが入って、ハイブリッド検索がかなり楽になった。セルフホストもCloud版もある。

**pgvector**はPostgreSQLの拡張で、既存スタックがPostgresなら採用しやすい。v0.5.0からHNSW indexが使えるようになって速度は改善したが、純粋なベクターDBと比べると大量データ時のANN性能は一歩劣る印象がある。「Postgresで完結したい」という強い動機がない限り、専用DBの方が後々楽だと思う。

**Pinecone**はマネージドサービスとしての完成度が高く、運用コストを下げたいチームには向いている。料金は相応にかかるし、ベンダーロックインは覚悟すること。

**Weaviate**はマルチモーダル対応が強み。テキスト以外（画像など）も扱うなら選択肢に入る。うちはテキストのみだったので深く評価しなかった。

うちの場合、最終的にQdrantのセルフホスト（Kubernetes on AWS）を選んだ。理由は3つ——コスト（Pineconeより安い）、ハイブリッド検索サポートの充実、Rustベースの安定性。想定データ量（数百万ドキュメント規模）とコスト感を考えると、セルフホストに振り切った方が合っていた。

---

## ハイブリッド検索とリランキングで精度が跳ね上がった

ここが一番「そういうことか」となったポイント。

純粋なベクター検索（semantic search）は意味的な類似性を捉えるのが得意だが、キーワードの完全一致が弱い。「AWS S3のバケット名の文字数制限」みたいな具体的なクエリで、S3・バケットというキーワードが入っているドキュメントをうまく拾えないことがあった。逆にBM25（キーワードベースの全文検索）は完全一致に強いが、言い回しが違う類義語的なクエリに弱い。

ハイブリッド検索はこの両方を組み合わせる。RRF（Reciprocal Rank Fusion）でスコアをマージするのが一般的なアプローチ。

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("localhost", port=6333)

# Dense embeddingを取得（OpenAI, Cohereなど）
dense_vector = embedding_model.encode(query)

# Sparse vector（BM25相当）を取得
# fastembed の BM25 エンコーダーが使いやすい
sparse_vector = bm25_encoder.encode_queries(query)

results = client.query_points(
    collection_name="documents",
    prefetch=[
        # Dense vectorで上位候補を取得
        models.Prefetch(
            query=dense_vector,
            using="dense",
            limit=100,
        ),
        # Sparse vectorで上位候補を取得
        models.Prefetch(
            query=models.SparseVector(
                indices=sparse_vector.indices.tolist(),
                values=sparse_vector.values.tolist(),
            ),
            using="sparse",
            limit=100,
        ),
    ],
    # RRFで両者のランクをマージ
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=20,
)
```

さらにリランキングを入れると精度がもう一段上がった。Cross-encoderベースのリランカー（Cohereの`rerank-multilingual-v3.0`は日本語にも対応している）を使って、上位20件を5件に絞るパターン。

リランキングのコストとレイテンシを最初は心配していたが、ベクター検索で絞った後の少数ドキュメントに対してだけ動かすので、実際のレイテンシ増加は許容範囲内だった（うちの場合、p99で800msほど追加）。

**ハマったこと**: リランキングモデルの言語対応を確認し忘れた。英語しか対応していないモデルを日本語ドキュメントに使って、スコアが完全にめちゃくちゃになった。日本語コンテンツには`Cohere rerank-multilingual-v3.0`かBGE-Reranker-v2-m3あたりが無難。これは本当に後悔している。

ハイブリッド検索は最初から設計に組み込んでおくべきだった。後から追加しようとすると、コレクションの作り直しとエンべディングの再生成が必要になって、結構大変だった。

---

## 評価なしで改善はできない: RAGASで定量測定する

2週間試行錯誤して気づいたのは、「なんとなく良さそう」で判断していても全く意味がないということ。RAGAS（Retrieval Augmented Generation Assessment）で定量評価を入れてから、ようやく改善が加速した。

RAGASが計測してくれる主要メトリクス——

- **Faithfulness**: 回答がコンテキストに基づいているか（ハルシネーション検出）
- **Answer Relevance**: 回答がクエリに対して関連しているか
- **Context Recall**: 正解に必要なコンテキストを拾えているか
- **Context Precision**: 取得したコンテキストのうち関連するものの割合

最初にRAGASのスコアを出したときは結構ショックだった。Faithfulnessが0.62しかなくて、「これはまずい」と素直に思った。

評価データセットを作るのが地味に大変だが（テスト用のQ&Aペアを100件手で作った）、これなしにチューニングするのは感覚だけでコードを書くようなものだと思う。固定サイズチャンキングからHierarchicalに変えたらContext Recallが0.59→0.78に改善し、ハイブリッド検索を追加したらさらに0.84まで上がった。こういう定量的な変化が見えるようになると、何が効いているか判断できるようになる。

評価セットの作成コストを惜しまないこと。100件のQ&Aを作るのに丸一日かかったが、それがなければ2週間の改善作業は単なる時間の無駄になっていた可能性が高い。

---

## 自分が実際に選ぶなら

「ケースバイケース」と言いたいところだが、あえて推薦を書く。

**チャンキング**: まずHierarchical chunkingを試してほしい。実装コストは少し上がるが、精度改善の効果が大きい。子チャンク300〜500トークン、親チャンク1500〜2000トークンあたりから始めるといい。オーバーラップは10%程度。

**ベクターDB**:
- ローカル/小規模プロトタイプ → **Chroma**（セットアップが楽）
- プロダクション、コスト重視 → **Qdrant**（ハイブリッド検索込みで選ぶならここ）
- 既存がPostgres → **pgvector**（ただしスケールに限界がある）
- マネージドで楽したい → **Pinecone**（料金と相談）

**検索パイプライン**: Dense + Sparse + RRFのハイブリッド検索を最初から設計に入れること。後付けは辛い。リランキングはレイテンシと精度のトレードオフを計測した上で判断する。日本語コンテンツにはmultilingual対応のモデルを使うこと（これは本当に忘れがち）。

**評価**: RAGASを最初から入れる。後回しにすると絶対に後悔する。

RAGは「動くシステム」を作るのは簡単だが、「使えるシステム」を作るのは全然別の話だった。チャンキング・検索・評価の3つを真剣に考えないと、ユーザーの信頼を失うのは早い。うちがそれを体感するのに2週間かかった。

<!-- Reviewed: 2026-03-07 | Status: ready_to_publish | Changes: チャンキング比較リストの並列構造を解体・文長に変化をつけた、ベクターDB各評価に個人的な文脈と判断基準を追加、Semantic chunking費用試算の補足追加、ハイブリッド検索セクションにハマったことの強調と後悔の一文追加、重複した締め段落を削除 -->
