---
title: "Deno 2.0を本番で2ヶ月動かして分かったこと — Node.jsからの移行は思ったより素直で、思ったより面倒だった"
emoji: "🚀"
type: "tech"
topics: ["deno", "nodejs", "typescript", "backend", "migration"]
published: true
---

## 移行前のNode.jsスタック、そして正直うんざりしていた理由

3人チームで管理してる社内APIがある。データパイプラインの管理UI向けのバックエンドで、Express + TypeScript、エンドポイントが30ちょっと、PrismaでPostgreSQLに繋いでた。コードベース自体は大きくない。

でも設定ファイルが多かった。`tsconfig.json`、`.eslintrc`、`.prettierrc`、`nodemon.json`、`jest.config.ts`、`.env.example`... 新人が入るたびに「このファイルは何のため？」という質問が来る。あとCI/CDで`ts-node`のバージョンが微妙にずれてパイプラインが落ちる、みたいなことが月に一度はあった。

Deno 2.0に移行したかった理由はそれだけ。「面白そう」でも「パフォーマンスが上がるはず」でもなく、ただ疲れてた。

Deno 2.0は2024年10月にリリースされて、以来ずっと気になってた。Node.js互換性が大幅に改善されたというのと、`npm:`プレフィックスでnpmパッケージをそのまま使えるというのが決め手だった。2026年1月中旬に移行を決断。ただしこれは勢いで決めた話ではなく、年末年始に一度ブランチで試して「いける」と判断してからだ。移行にかかった実時間は約2週間——1週間が移行作業、1週間がバグ潰し。

---

## npmパッケージ互換性の現実 — 「動く」と「快適に動く」は別の話

互換性は想定よりずっと良かった。ただ「全部そのまま動く」という期待は甘かった。

まず良い方から。`npm:`プレフィックスで大半のパッケージはほぼそのまま動く。移行初日のコードはこんな感じだった:

```json
// deno.json
{
  "imports": {
    "express": "npm:express@4.21.0",
    "zod": "npm:zod@3.23.0",
    "@prisma/client": "npm:@prisma/client@5.22.0"
  },
  "tasks": {
    "dev": "deno run --allow-all --watch src/main.ts",
    "start": "deno run --allow-all src/main.ts",
    "check": "deno check src/main.ts"
  }
}
```

```typescript
// src/main.ts — ほぼ移行前のまま動いた
import express from "express";
import { z } from "zod";

const app = express();
app.use(express.json());

const pipelineSchema = z.object({
  name: z.string().min(1),
  schedule: z.string(),
  enabled: z.boolean().default(true),
});

app.post("/pipelines", async (req, res) => {
  const result = pipelineSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ error: result.error.flatten() });
  }
  // DB処理...
});

app.listen(3000);
```

Express、zod、axios、dotenv — これらはほぼノータッチで動いた。

問題が起きたのはPrismaだった。`prisma generate`がDenoのキャッシュディレクトリと干渉して、最初は起動時にエラーが出まくった。半日溶かした挙句、結局`PRISMA_QUERY_ENGINE_LIBRARY`を環境変数で明示的に設定することで解決したけど、公式ドキュメントの記載はまだ薄い（GitHubのissue #23814が一番参考になった。同じで詰まってる人はそっちを見てほしい）。

Node.js組み込みモジュールを `import fs from "fs"` と書いてる箇所も全部 `import fs from "node:fs"` に書き直しが必要だった。30ファイルくらい。手作業だったけど、`deno check`が未対応箇所を全部洗い出してくれたのは助かった。

一個完全に想定外だったのが `__dirname` と `__filename`。Denoには存在しない。これを使ってたコード（主にパス解決）が軒並み壊れた。`import.meta.dirname` で代替できるんだけど、これに気づくまでの30分間、「なんでこのファイルパスが見つからないんだ」とずっと悩んでた。Denoのエラーメッセージは `__dirname is not defined` とだけ言う。代替方法まで辿り着くのに検索が必要で、移行中の30分は地味に痛かった。

---

## 本番2週間目に起きたこと

Right, so — 移行が終わって、テスト環境で1週間問題なく動いた。金曜の午後にステージングから本番に切り替えた。

最初の4時間は順調だった。

夕方18時ごろ、「WebSocketが途切れる」という報告が来た。パイプラインのリアルタイムログをWebSocketで流してたんだけど、1時間に数回、接続が勝手に閉じてた。ユーザーからすると画面上のログが突然止まる。

調べたところ、原因はDenoのネイティブWebSocket実装がNode.jsの`ws`ライブラリとkeep-aliveの挙動が異なること。Node.jsの`ws`はデフォルトでping/pongを自前管理してたけど、Denoのネイティブ`WebSocket`はそこを自分で実装しないといけない。

```typescript
// 修正後：keep-aliveを明示的に管理する
function setupWebSocketKeepAlive(socket: WebSocket) {
  const interval = setInterval(() => {
    if (socket.readyState === WebSocket.OPEN) {
      socket.send(JSON.stringify({ type: "ping" }));
    } else {
      clearInterval(interval);
    }
  }, 30_000);

  socket.addEventListener("close", () => clearInterval(interval));
}
```

土曜の朝に修正して、以降は問題なく動いてる。ただ、これは自業自得な面もある。`npm:ws`をそのまま使い続ければ起きなかった問題で、「せっかくDenoに移行したんだからネイティブAPIを使いたい」という意地が余計な作業を生んだ。ネイティブAPIに切り替えるなら、挙動の差分は自分で確認しないといけない。分かってたはずのことを、移行の勢いで油断してた。

---

## パフォーマンスの数字と、本当に意味があった変更

2ヶ月運用した結果の数字を書く。比較対象はNode.js 22.x（移行前の最終バージョン）。

**コールドスタート時間**: Node.js + ts-nodeが約2.1秒 → Deno 2.0が約0.8秒。1.3秒の差がデプロイ頻度の高い環境では地味に効く。コンテナを再起動するたびに0.8秒速くなる。

**メモリ使用量**: 正直、ほぼ変わらなかった。アイドル時でNode.jsが約95MB、Denoが約88MB。誤差の範囲。「Denoはメモリ効率がいい」という話を聞いてたけど、このサイズのアプリでは差が出なかった。

**開発体験**: これが一番変わった。`deno fmt`、`deno lint`、`deno check`がビルトインなので、設定ファイルが大幅に減った。

移行前のdevDependencies（抜粋）:
```
typescript, ts-node, nodemon, eslint, @typescript-eslint/parser,
@typescript-eslint/eslint-plugin, prettier, jest, @types/jest, ts-jest...
```

移行後: `deno.json`に本番依存のみ。`package.json`が消えた。

これだけで十分やった価値はあったと思ってる、少なくとも僕の場合は。

One thing I noticed: 移行後に一番喜んでたのは、意外にもPRレビューが楽になったこと。`deno fmt`が強制されるからフォーマットの差分が出ない。誰が書いてもコードが揃う。「インデント4つか2つか」みたいな不毛な議論がなくなった。地味だけど、週に何度もPRを見てるとかなり効く。

あと試験的に使ってる機能として、`deno compile`でシングルバイナリを出力できる。Dockerイメージが230MB→18MBになった（まだ本番には入れてないけど、いずれ移行したい）。

---

## 結局、Deno 2.0に移行すべきか — 正直な答え

「あなたのユースケース次第です」とは言わない。もう少し具体的に言う。

**今すぐ移行する価値があるケース:** Node.jsの設定ファイル地獄に疲れてて、チームが小さくて（5人以下くらい）、コアな依存がExpressやfastify + zodみたいなメジャーどころだけなら、移行コストは低い。僕のケースがまさにこれだった。移行して後悔してない。

**慎重にすべきケース:** PrismaやSequelizeが中心のコードベースで、かつDeno経験者がゼロなら、今すぐ移行するのは割りに合わないかもしれない。Prismaは動くけど、追加の手間がかかる。ネイティブアドオン（`.node`ファイル）を使ってるパッケージが入ってるなら、まだ時期尚早。

大きめのチーム（10人以上）でDeno経験者がいない場合は、学習コストを甘く見ない方がいい。基本的なことは似てるけど、権限モデル（`--allow-net`とか）や`import.meta`の扱い、それにNode.jsとのAPIの微妙な差分は、ある程度慣れが必要。

僕が一番重視してるのは「設定ファイルが減るかどうか」という基準。それが今のプロジェクトの痛点なら、Deno 2.0はほぼ確実に改善をもたらす。逆に、現状のNode.jsスタックで開発体験に不満がないなら、移行コストを払うメリットは薄い。Node.js 22以降の開発体験も実際かなり良くなってるので。

一個だけ強くすすめる: 移行前に、自分のコードベースで一番クリティカルなライブラリ（ORMとかWebSocketとか）を先に単体検証すること。週末の本番障害で調べるより、平日の昼間に落ち着いて調べる方がいい——これは経験から言ってる。

<!-- Reviewed: 2026-03-10 | Status: ready_to_publish | Changes: tightened opening backstory, added context for migration decision timing, punched up Prisma pain paragraph, trimmed generic phrasing in conclusion, sharpened final recommendation to first-person experience, extended meta_description -->
