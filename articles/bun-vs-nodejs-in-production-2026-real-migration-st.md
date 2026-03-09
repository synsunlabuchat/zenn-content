---
title: "Bun vs Node.js 本番環境 2026: 2週間の移行実験で分かったこと"
emoji: "🚀"
type: "tech"
topics: ["bun", "nodejs", "javascript", "typescript", "performance"]
published: true
---

うちのチームは4人で、BtoBのSaaSプロダクトを運営している。バックエンドのAPIはずっとNode.js 20 LTSで動かしていて、Bunの存在は2年以上前から知っていたけど「まだ本番には早い」という印象が拭えなくて様子見してた。

転機になったのは去年末。マイクロサービス化を少しずつ進めていて、コンテナのコールドスタートが地味に問題になってきた。個別には130〜180ms程度だけど、それが連鎖するとユーザーに見える形でレイテンシに響いてくる。それで「Bun 1.3、2週間だけ本番相当の環境で検証してみよう」という話になった。

この記事は、その2週間で分かったことをそのまま書いたもの。

---

## ベンチマークの数字: コンテキストなしには意味がない

まず実測値から出しておく。テスト環境はAWS t3.medium、Amazon Linux 2023。対象は社内の軽量なREST API（依存パッケージ数 約40）をベースに作ったテスト用サービス。

**コールドスタート（`/health`への初回応答まで）**
- Node.js 22.x LTS: 平均 **171ms**
- Bun 1.3: 平均 **29ms**

**メモリ使用量（アイドル時）**
- Node.js: 〜**47MB**
- Bun: 〜**20MB**

**スループット（wrk、12スレッド、400コネクション、30秒）**
- Node.js + Express: 〜**27,800 req/s**
- Bun + Hono: 〜**62,400 req/s**

数字だけ見ると「全部Bunに移行しよう」ってなるんだけど、一個重要なことがある。スループットの比較でBun + Honoを使っているのは、「同じものを速くした」のではなく、**別のスタックに乗り換えている**んだよね。Honoはとても速いフレームワークだけど、Expressのミドルウェアエコシステムとは別物。ここを混同すると移行コストを見誤る。

実際のHono + Bun最小構成はこんな感じ。TypeScriptをトランスパイルなしで動かせるのは、地味に体験として良かった。

```typescript
// hono + bun: 最小構成（bun run src/index.ts で直接動く）
import { Hono } from 'hono'
import { logger } from 'hono/logger'

const app = new Hono()

app.use('*', logger())

app.get('/health', (c) => c.json({ status: 'ok', runtime: 'bun' }))

app.get('/users/:id', async (c) => {
  const id = c.req.param('id')
  // Bunのビルトインsqliteが使える場合はここで直接クエリ
  const user = await db.query('SELECT * FROM users WHERE id = ?', [id])
  return c.json(user)
})

export default app
// bun --hot src/index.ts でホットリロードも動く
```

ただ、この構成で「既存のExpressベースのコードをそのまま動かそう」とすると詰まる。次のセクションがその話。

---

## 本番移行で詰まった3つのポイント

**DatadogのAPMが動かなかった**

最初のハマり。`dd-trace`はNode.jsのV8内部フック、特に`--require`フラグとインスペクターAPIの特定の挙動に依存していて、Bunではその初期化シーケンスが違う。起動時のトレースが全く取れなくて30分くらい悩んだ。

調べたら、Datadog公式のBunサポートは2025年夏のGitHub issue段階では「experimental」のままで、1.3時点でも「完全対応」とは言えない状況だった。

解決策はOpenTelemetryベースへの切り替え。Datadog AgentはOTLPを受け付けるので、以下のように書き換えたら動いた。

```typescript
// dd-traceの代替: OpenTelemetry SDKをBunで使う
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
})

sdk.start()

// BunではSIGTERMのハンドリングを明示的にやらないとshutdownが走らないことがある
process.on('SIGTERM', async () => {
  await sdk.shutdown()
  process.exit(0)
})
```

Datadogを使っているなら、OTLPへの移行を先に計画してからBun移行に入ること。これを後回しにすると確実に詰まる。

**`node:crypto` のSubtle APIで金曜の午後にやらかした**

これが一番焦った。金曜の午後にステージングから本番に上げたサービスで、JWTの検証が一部通らなくなった。具体的にはRSA-PSS署名の検証で、Node.jsとBunのWebCrypto実装の挙動が微妙に違ってた。

最初は「Bunのバグか？」と思ったんだけど——調べたら逆で、BunはW3C仕様に沿った実装をしていて、Node.jsの方が独自の挙動をしていた（Bunのissue trackerで同様の議論がいくつか見つかった）。

正直、「どちらが仕様的に正しいか」はどうでもよかった。動いていたものが動かなくなった、それだけが問題で。結局、JWTライブラリ側で`jose`に切り替えて、直接`SubtleCrypto`を触るコードをラップすることで回避した。本番への影響は最小限で済んだけど、crypto周りは特に注意が必要。

**`bun:test`はJestではない**

これは完全に自分の思い込みが原因。BunのテストランナーはJestと似た構文なので「そのまま動くだろ」と思って移行し始めたら、`jest.spyOn`のリストア挙動や、一部のモジュールモック機能が微妙に違った。基本的なテストは動いたけど、既存テストの約20%に修正が必要だった。小さくないコスト。

---

## Node.jsが依然として有利な場面

Bunを推す記事は起動速度とメモリ効率を強調しがちだけど、2026年3月時点でも、Node.jsの方が明確に有利な場面がある。

**ネイティブモジュール**。`sharp`（libvipsのラッパー）、`canvas`、`node-sqlite3`など、node-gypで書かれたモジュールは動かないものがまだある。うちのケースでも、画像処理サービスが`sharp`に依存していて、そこはBunへの移行を断念した。Bunのネイティブモジュールサポートは改善を続けているけど、「全部動く」とは言えない。

安定性と情報量の話もある。Node.jsは10年以上の本番実績があって、LTSサポート、CVE対応の速さ、クラウドプロバイダーのドキュメント——全部Node.jsが一日の長がある。Bunでエラーが出たとき、まだ「Bun Discordで聞く」という体験が数回あった。Stack Overflowで即解決できるNode.jsとは情報の密度が違う。これは経験則だけど、無視できない。3年以上無人で走るバッチ処理や重いETLジョブをBunに突っ込む気にはまだなれないのも、結局ここに帰着する。実績年数がまだ足りない。

---

## 2026年3月時点での俺の結論

実験の結果、うちのチームは**新規のワーカーサービス2本をBun + Honoで書き直して本番投入した**。既存のメインAPIはNode.jsのまま。

バンドラーとしてのBunは、ランタイムより先に試す価値がある。既存のwebpackやesbuildをBunのバンドラーに置き換えたら、ビルド時間が体感で5〜6倍速くなった。ランタイム移行より摩擦が少ないので、まずそこから始めることをすすめる。

Bunが本領を発揮するのは、TypeScriptをそのまま動かしたい軽量な新規サービス、コールドスタートが問題になっているLambda/コンテナ環境、エコシステムの制約を受けない選択ができる新規プロジェクトだ。一方で、既存の重いミドルウェアスタックを抱えているサービス、APMやセキュリティツールがNode.js前提になっている環境、ネイティブモジュール依存が多いサービスは——正直まだNode.jsで行った方が楽。これは「Bunが劣っている」という話ではなく、成熟度とエコシステムの問題。

「結局どっちが勝ち？」という問いへの答えは——Node.jsは成熟した信頼性、Bunはスピードと開発体験、という差は本物だった。ただ、移行コストを甘く見ると痛い目を見る。特にAPM、crypto、テストランナー周りは必ず検証してから進めてほしい。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: removed parallel bold-header in Node.js section; merged stability+batch paragraphs; rewrote conclusion bullet-lists as prose; sharpened APM section closing to actionable advice -->
