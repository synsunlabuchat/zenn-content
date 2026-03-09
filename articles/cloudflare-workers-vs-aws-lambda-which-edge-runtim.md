---
title: "Cloudflare Workers vs AWS Lambda: プロダクションで本当に使えるエッジランタイムはどちらか"
emoji: "🚀"
type: "tech"
topics: ["cloudflare-workers", "aws-lambda", "serverless", "edge-computing", "javascript"]
published: true
---

去年の秋、チームの新しいAPIプロジェクトでエッジランタイムの選定をすることになった。メンバーは5人、ユーザーは日本と東南アジアに分散している。「とりあえずLambdaでいいんじゃない？」という声もあったけど、前のプロジェクトでコールドスタートに何度も泣かされていたので、今回はCloudflare Workersも真面目に評価することにした。

2週間、両方を本番に近い環境で動かした。加えてその後数ヶ月の実運用がある。その経験を正直に書く。

## コールドスタート：計測してみると印象が変わる

Lambdaのコールドスタートはずっと開発者コミュニティの不満の種だ。自分の計測では、Node.js 20ランタイム、メモリ512MB、東京リージョンという構成で、コールドスタートは平均800ms〜1.2秒程度かかった。トラフィックが途切れると関数がフリーズして、次のリクエストが来たときに起こされる。そのたびにユーザーが1秒待たされる。

Cloudflare Workersは根本的に異なる仕組みを使っている。V8アイソレートと呼ばれる、NodeのプロセスモデルではなくChromeのJavaScriptエンジンを直接流用したアーキテクチャだ。コールドスタートの計測値は5ms以下のことがほとんどで、自分のテストでも同じ数字が出た。

ここで「じゃあWorkers一択だな」と思った。が、実際はもう少し話が複雑だった。

LambdaにはProvisioned Concurrencyという機能がある。追加コストはかかるが、これを有効にするとコールドスタートがほぼゼロになる。構成によるが月20〜40ドルの追加という感覚で、常にホットな状態を維持できる。本番の重要エンドポイントで「絶対にコールドスタートを出したくない」という要件があるなら、これで解決できる。

つまり「コールドスタートに困っているならWorkers」は正しい。でも「Lambdaではコールドスタートを解決できない」というのは正確ではない。コスト的に許容できるかどうかの話になる。

## デプロイ体験の差が想像より大きかった

正直、ここが一番驚いたポイントかもしれない。技術的なスペックよりも、日常の開発サイクルに直結する部分が全然違う。

Cloudflare Workersのデプロイは速い。`wrangler deploy`を叩いてから本番に反映されるまで10〜15秒。しかもその変更が300以上のグローバルエッジロケーションに一瞬で広がる。

```toml
# wrangler.toml
name = "user-api"
main = "src/index.ts"
compatibility_date = "2025-09-01"

[[kv_namespaces]]
binding = "SESSION_STORE"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

```typescript
// src/index.ts — シンプルな認証ミドルウェアの例
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const token = request.headers.get("Authorization")?.split(" ")[1];
    if (!token) {
      return new Response("Unauthorized", { status: 401 });
    }

    // セッションをエッジのKVから引く（レイテンシ: ~1ms）
    const session = await env.SESSION_STORE.get(`session:${token}`, "json");
    if (!session) {
      return new Response("Invalid token", { status: 403 });
    }

    // リクエストにユーザー情報を付けてオリジンに転送
    const upstreamRequest = new Request(request, {
      headers: { ...Object.fromEntries(request.headers), "X-User-Id": session.userId }
    });
    return fetch(upstreamRequest);
  }
};
```

このコードをデプロイして全世界に反映させるのに15秒。体験としてはgit pushに近い感覚だ。

一方のLambdaはというと。SAMやServerless Frameworkを使えば`sam deploy`で自動化できるが、IAMロール、VPCの設定、CloudWatchのロググループ、エラー通知のSNS設定……初回セットアップの複雑さが桁違いだ。慣れれば問題ないが、新メンバーがオンボードするたびに「CloudFormationのスタックがなぜかROLLBACK_COMPLETEになってて」という事態が普通に起きる。

デプロイ時間も違う。小さな変更でもLambdaは1〜2分かかることがある。毎日50回以上デプロイする開発サイクルでは、この差がじわじわ効いてくる。デプロイ体験の差は技術スペックの話ではなく、一日の作業リズムの話だ。Workersが上——これは体感として揺るがない。

## V8アイソレートが持つ制約と、それが何を意味するか

Workersのアーキテクチャが速い理由と、その代償は同じコインの表裏だ。

V8アイソレートはNodeのプロセスではないため、Node.js固有のAPIが使えない。`fs`、`net`、`child_process`、`crypto`（Node版）——これらが全部動かない。代わりにWeb標準API（`fetch`、`Request`、`Response`、`crypto.subtle`など）が使える。最近は互換性フラグで一部のNode APIがpolyfillされているが、まだ完全ではない。

これが本番で最初に問題になったのはnpmパッケージを使おうとしたときだった。`pg`（PostgreSQLクライアント）は動かない。`mysql2`も動かない。WorkersからRDBMSに直接接続するには、Cloudflare Hyperdrive経由か、HTTP APIを持つマネージドサービス（Supabase、Neon、PlanetScaleなど）を使う必要がある。これはアーキテクチャの選択を強制してくる。

実行時間の上限も違う。WorkersのFree tierはCPUタイム10ms、Paid planでも最大30秒（Workersのcron jobsはもう少し長い）。Lambdaは最大15分。バッチ処理、重い変換処理、長時間のオーケストレーションには構造上Workersは向いていない。これはバグではなく設計思想だが、最初から知っていないと後で痛い目を見る。メモリ制限はWorkersが128MB固定に対して、Lambdaは最大10GBまでスケールできる。機械学習の推論、大きなファイル変換、メモリを食う処理——こういった用途でWorkersを使おうとするのは根本的に間違っている。

一つ誤解していたことを白状すると——最初、WorkersのV8制約は「古いコードを動かすのが難しい」程度の話だと思っていた。実際には「何を使って何を構築するか」というスタック全体の選択に影響する。既存のNode.jsエコシステムに深く依存したプロジェクトでWorkersへ移行しようとすると、想定より大きなリライトが必要になる可能性がある。

## 本番で実際に踏んだ罠

金曜の夕方にWorkersへのマイグレーションをpushして、翌朝Slackを見たら「本番のAPIが断続的に500を返している」というメッセージが来ていた。最初はデプロイが原因だと思わなかった。インフラの問題かと疑って30分無駄にした。

原因はCrypto APIの挙動の違いだった。Lambdaで動いていたコードはNode.jsの`crypto`モジュールを使っていて、WorkersのWeb Crypto APIとはAPIが違う。具体的には：

```typescript
// Node.js (Lambda) — 同期、これがWorkers上では動かない
const hash = crypto.createHash('sha256').update(data).digest('hex');

// Web Crypto API (Workers) — 非同期になる
const encoder = new TextEncoder();
const hashBuffer = await crypto.subtle.digest('SHA-256', encoder.encode(data));
const hash = Array.from(new Uint8Array(hashBuffer))
  .map(b => b.toString(16).padStart(2, '0')).join('');
```

テスト環境では動いていたのに本番で落ちたのは、テストデータが単純すぎて問題のコードパスを踏んでいなかったせいだ。恥ずかしい。

Lambdaで経験した問題は性質が違う。2025年の2月、本番のLambda関数がVPCのENI（Elastic Network Interface）枯渇でタイムアウトを連発するインシデントがあった。コードの問題かインフラの問題かの切り分けに2時間かかって、最終的にはAWSのドキュメントを読み込んでVPC設定を変更して解決した。こういうインフラレイヤーの問題がLambdaでは起きやすい——Workersはインフラ管理をCloudflareに完全に委譲する分、このカテゴリの問題は経験していない。

デバッグのしやすさも違う。LambdaはCloudWatch Logsが充実していて、X-Rayで分散トレーシングも取れる。Workersも`wrangler tail`でリアルタイムにログを見られるが、複雑なマイクロサービス間のトレーシングのエコシステムはまだLambdaに比べると薄い。本番のバグを深く追いかけるときはLambdaのほうが情報が取りやすい——WorkersがダメというよりAWSの観測性エコシステムが単純に成熟している、ということだ。

## どちらをいつ使うか：数ヶ月動かして出た結論

俺の答えはシンプルだ。

Cloudflare Workersが向いているのは、レイテンシが最重要で、処理がシンプルなHTTPリクエスト中心の場合。認証ミドルウェア、APIゲートウェイ、エッジキャッシュロジック、地理的なトラフィックルーティング、A/Bテストのフラグ処理——これらはWorkersが本当に速くてデプロイも楽でコストも安い。リクエスト100万件あたり$0.50で、Lambdaより安くなるケースが多い。新規プロジェクトでAWSの他サービスとの深い連携が不要なら、Workersを最初の選択肢にするのが今は正しいと思う。

逆にLambdaを選ぶべきなのは、既存のNode.jsライブラリへの依存が大きい場合、実行時間が長い処理、メモリを大量に使う処理、AWSの他サービス（RDS、SQS、S3イベント、Step Functions）との連携が中心の場合だ。あとチームがAWSに慣れていてIAMやCloudWatchを日常的に扱えるなら、Lambdaの複雑さはそこまで障壁にならない——慣れの問題が大きい部分もある。

今のうちのチームは、ユーザー向けAPIレイヤーとWebアプリのSSRをWorkersで動かして、バックエンドの重い処理とAWSサービス連携部分はLambdaを使い続けている。この役割分担が今一番しっくりきている。

「新規プロジェクトでどちらか一方を選べ」と言われたら——RDBMSへの直接接続が不要で、チームがWeb標準APIに慣れているか慣れる意欲があるなら、Cloudflare Workersを薦める。コールドスタートを気にしなくていい、デプロイが速い、グローバル配信が設定不要——この快適さは実際に体験すると思っていた以上に大きかった。

Lambdaは「デフォルトで選ばれるもの」から「明確な理由があるときに選ぶもの」に変わりつつあると感じている。特にAWSエコシステムの外に軸足があるプロジェクトではなおさら。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: removed English "Right, so" mid-article; restructured conclusion from parallel bold-header format to flowing prose; varied paragraph length in V8 section; tightened observability sentence; minor voice adjustments throughout -->
