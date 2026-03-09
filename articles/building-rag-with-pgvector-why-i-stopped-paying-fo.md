---
title: "pgvectorでRAGを構築する: Pineconeの課金をやめた理由"
emoji: "🚀"
type: "tech"
topics: ["pgvector", "postgresql", "rag", "pinecone", "\u30d9\u30af\u30c8\u30eb\u691c\u7d22"]
published: true
---

Pineconeの請求書を開いたのは去年の9月で、金額は$283だった。

3人チームで社内向けのドキュメント検索ツールを作っていて、RAGのベクトルストアとしてPineconeのStarter → Standardと乗り継いできた。ベクトル数は当時約40万件。リクエスト数は1日せいぜい500回。それで$280。スタートアップとしては正直しんどい数字だし、何より「もうちょっとスケールしたら倍になる」という見えない圧力がずっとあった。

そこで同僚のTakuが「どうせPostgreSQLもう使ってるんだし、pgvectorでよくない?」と言い出した。最初は「まあそうだけど...」という感じで流していたのだが、ある週末に自分で試してみたら、思った以上に話が早かった。

## 月$280のPineconeと、何に課金していたのかという疑問

Pineconeは確かによくできている。APIはきれいだし、マネージドで運用コストはゼロ、メタデータフィルタリングも直感的に書ける。最初の数ヶ月は「これで正解だ」と思っていた。

問題はコストモデルにある。Pineconeはベクトルの「保存数」と「読み書きのユニット数」で課金される。うちのユースケースは読み書きよりも保存が主体で、40万ベクトルというのはたいして大きくない数字なのに、Standard tierじゃないと複数のnamespaceが使えない。namespaceを使いたかった理由はシンプルで、テナントごとにデータを分離したかったから。それだけのために$200/月のtierに上がるのは、どう考えてもオーバーキルだった（機能の半分も使わない tier に上げるのは、EV充電のために高速道路年間パスを買う感覚に近い）。

加えて、Pineconeはデータが自分のインフラの外にある。これは社内ツールだったので、会社の方針的にもグレーな部分があった。法務から「そのデータ、どこにあるの?」と聞かれたとき、「Pineconeというアメリカのサービスのサーバーに...」という回答は、あまり受け入れられなかった。

So、pgvectorに乗り換えるモチベーションは純粋にコストとデータ主権の2点だった。パフォーマンスが同等なら、の話だが。

## PostgreSQL 16 + pgvector 0.7.3のセットアップ: 実際にやった手順

まずpgvectorのバージョンについて。2025年末時点でv0.7.3が出ていて、HNSW indexingのパフォーマンスが0.6系から大きく改善されている。0.5系を使っているドキュメントがまだ多く残っているので注意が必要で、特にインデックスの作成構文が変わっている。

うちはAWS RDS PostgreSQL 16を使っていたので、pgvectorの有効化は単純だった。正直、拍子抜けするくらいには。

```sql
-- extensionの有効化
CREATE EXTENSION IF NOT EXISTS vector;

-- ドキュメントチャンクのテーブル
CREATE TABLE document_chunks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    doc_id      UUID NOT NULL,
    tenant_id   UUID NOT NULL,           -- テナント分離用
    content     TEXT NOT NULL,
    embedding   vector(1536),            -- text-embedding-3-smallの次元数
    metadata    JSONB,
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- HNSWインデックス (IVFFlatより精度が高く、更新にも強い)
CREATE INDEX ON document_chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- テナントごとのフィルタリングを想定したインデックス
CREATE INDEX ON document_chunks (tenant_id);
```

埋め込みの生成とストアはPythonで書いた。OpenAIのtext-embedding-3-smallを使っていて、1536次元。

```python
import psycopg2
from openai import OpenAI
from pgvector.psycopg2 import register_vector

client = OpenAI()

def store_chunks(conn, chunks: list[dict], tenant_id: str):
    register_vector(conn)
    
    # バッチで埋め込みを取得 (APIコストを減らすため)
    texts = [c["content"] for c in chunks]
    response = client.embeddings.create(
        input=texts,
        model="text-embedding-3-small"
    )
    
    with conn.cursor() as cur:
        for chunk, emb_data in zip(chunks, response.data):
            cur.execute(
                """
                INSERT INTO document_chunks 
                    (doc_id, tenant_id, content, embedding, metadata)
                VALUES (%s, %s, %s, %s, %s)
                """,
                (
                    chunk["doc_id"],
                    tenant_id,
                    chunk["content"],
                    emb_data.embedding,   # pgvector.psycopg2がリストをvector型に変換
                    chunk.get("metadata", {}),
                )
            )
    conn.commit()

def search_similar(conn, query: str, tenant_id: str, limit: int = 5):
    register_vector(conn)
    
    q_emb = client.embeddings.create(
        input=query,
        model="text-embedding-3-small"
    ).data[0].embedding
    
    with conn.cursor() as cur:
        cur.execute(
            """
            SELECT content, metadata, 
                   1 - (embedding <=> %s::vector) AS similarity
            FROM document_chunks
            WHERE tenant_id = %s
            ORDER BY embedding <=> %s::vector
            LIMIT %s
            """,
            (q_emb, tenant_id, q_emb, limit)
        )
        return cur.fetchall()
```

`<=>` がコサイン距離の演算子。`<->` がL2距離。最初これを混同して、なんかちょっと変な結果が返ってくるなと思ったら距離関数が違っていた、ということがあった。埋め込みモデルによってどっちが向くかは変わるので、OpenAIのモデルならコサイン距離を使うのが基本。

## 2週間の並行稼働で見えたこと

移行に際して2週間、PineconeとpgvectorをA/Bで動かした。同じクエリを両方に投げて、返ってくるチャンクの内容を比較するシンプルな検証。

結果として、top-5の一致率は約87%だった。完全一致ではないが、回答品質に影響が出るような差ではなかった。これは正直、思ったより近い数字だった——移行前は「専用ベクトルDBと戦えるわけない」という先入観があったので。

レイテンシは条件による。40万ベクトル、テナントあたり平均1万件という構成で:

- Pinecone (p1.x1): p50で約45ms、p99で約120ms
- pgvector HNSW (RDS db.r6g.large): p50で約38ms、p99で約95ms

pgvectorの方が速い、というのは正確ではなくて、「ネットワークホップが1つ減るから同じAWS VPC内であればRTTが小さくなる」というだけの話だと思う。Pineconeのエンドポイントがus-east-1に固定されていて、うちのアプリサーバーもus-east-1にあるのに、なぜかp99でレイテンシが高かった。

One thing I noticed: テナントフィルタリングの書き方で性能が大きく変わる。`WHERE tenant_id = %s ORDER BY embedding <=> ... LIMIT 5` とすれば、PostgreSQLはまずtenant_idでフィルタしてからベクトル検索をかける。が、テナントあたりのデータが少ない（100件以下）とHNSWインデックスが効かずにシーケンシャルスキャンが走ることがある。`SET enable_seqscan = off;` で強制的にインデックスを使わせることもできるが、本番でそれをやるのは慎重にした方がいい。

## 本番で踏んだ地雷: IVFFlatを最初に選んだ話

ここが一番書きたい部分。

最初のインデックスはHNSWではなくIVFFlatを選んだ。理由は単純で「pgvectorのドキュメントでよく見かけるのがIVFFlat」だったから——今思えば、古いチュートリアルを鵜呑みにした完全な判断ミスだ。

IVFFlatは事前に`LISTS`パラメータ（クラスタ数）を指定する必要があって、このときベクトル数が十分に存在していないとインデックスが使い物にならない。pgvectorのドキュメントには「ベクトル数の平方根程度のLISTS値を推奨」とあるが、インデックス作成後にデータが大幅に増えると再構築が必要になる。

本番に切り替えて3日後の金曜の午後、ドキュメントのバッチインポート機能をリリースした。ユーザーが大量のPDFをアップロードし始めて、ベクトル数が40万から一気に65万に膨らんだ。その瞬間、検索のp99が800msを超えた。

原因は結局、IVFFlatのLISTS値が実際のベクトル数と合わなくなってインデックスの効率が落ちたこと——ではなくて、調べてみたらインデックスがそもそも使われていなかった。work_memが低くてPostgreSQLがインデックスよりシーケンシャルスキャンを選んでいた。`SET work_mem = '256MB'` をセッションレベルで試したらp99が90msに戻ったが、これをインスタンスレベルで設定するには再起動が必要で、その日は結局対応できなかった。金曜の午後だったので、週が明けるまでモニタリングを強化しながらひやひやしていた。

週が明けてHNSWに切り替えたら問題は解消した。HNSWはオンライン更新に強く、データが増えても再構築が不要。インデックス作成自体は遅いが（65万ベクトルで約12分かかった）、一度作ってしまえば安定している。

`m`（各ノードの接続数）と`ef_construction`（構築時のビームサイズ）のデフォルト値（16と64）は、うちのユースケースでは変更しなかった。精度をさらに上げたければmを増やすが、インデックスサイズとのトレードオフになる。100% recallが必要なユースケースでないなら、デフォルトで十分だと思う。

## 結局うちはpgvectorで正解だった、ただし条件付きで

コストは移行後、$283/月 → ほぼゼロ（RDSのインスタンス代はもともとかかっていたので）。

Pineconeを選ぶべきケースは実際にある。ベクトル数が億単位になってきたとき、PostgreSQLのストレージとクエリ性能がどこまで耐えられるかは正直わからない。1000万ベクトルくらいまでは問題ないという事例を見かけるが、それ以上は自分で検証していないので確信を持てない。

あとは、pgvectorはPostgreSQLの運用知識が必要。インデックスチューニング、work_memの設定、VACUUMの挙動——これらを知らないまま使い始めると私みたいにp99が800msになる。Pineconeはそういうことを気にしなくていい。それはちゃんとした価値だ。

ただ、もし今からRAGを構築するなら——3人チームで、ベクトル数が数百万規模以下で、すでにPostgreSQLを使っているなら——最初からpgvectorで始めることを勧める。Pineconeは必要になってから移行を検討すればいい。逆順の方が楽だ。

Pineconeから移行する場合、データのエクスポートはAPIから全ベクトルをフェッチするしかなく、40万件で約1時間かかった。移行スクリプト自体は単純だが、この1時間だけは覚悟しておくといい。あとエクスポート中にPineconeへの書き込みが走ると整合性がズレるので、できれば書き込み停止ウィンドウを取った方がいい——うちは深夜にやった。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: added parenthetical aside on tier upgrade rationale, added "拍子抜け" reaction to setup section, strengthened A/B results with pre-migration expectation, added IVFFlat hindsight aside, added Friday-night detail to incident, added export consistency warning to closing -->
