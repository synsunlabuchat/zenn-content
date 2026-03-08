---
title: "GitHub Copilot代替ツール比較2026：Cursor・Codeium・Tabnine・Amazon Qを2週間使い倒した"
emoji: "🚀"
type: "tech"
topics: ["ai-coding-tools", "cursor", "codeium", "tabnine", "amazon-q"]
published: true
---

去年の12月、チームのSlackに「Copilot Businessの月次請求が先月比で跳ね上がった」というメッセージが届いた。チームが7人から11人に増えたせいで、それをきっかけに「そもそも代替ツールってどうなの？」という話になった。私が自ら手を挙げて2週間かけて4つのツールを試した。

これはその記録。

## GitHub Copilotを疑い始めた経緯と評価条件

正直、Copilotに大きな不満があったわけじゃない。でも、チームが成長するほど固定費として重くなるのは事実で、しかも最近のAIコーディングツールの進化は本当に速い。半年前に試して「微妙だったな」と思ってたツールが、今は全然別物になってたりする。

私のセットアップ：TypeScriptとPythonがメイン、Next.js + FastAPIのスタックで、インフラはAWS。チームはフルリモートで、コードベースはモノレポで約15万行。ツールによっては大規模コードベースへの対応に差が出るので、これが重要な前提になる。

テスト期間中は実際の業務タスクで使い続けた。本番のAPIエンドポイントを実装したり、既存コードのリファクタリングをしたりという文脈での評価で、「Hello Worldを書いてみた」みたいな浅いテストはしていない。

## Cursor：エディタごとAI前提で作り直した、という発想

最初にCursorを試した。エンジニアのXのタイムラインに頻繁に出てきてたから、というシンプルな理由。

VSCodeのforkなので移行コストが低い。設定も拡張機能もほぼそのまま引き継げる——私みたいにVSCodeのキーバインドが体に染み込んでる人間には導入摩擦がほぼない。この「乗り換えコストの低さ」は地味に重要で、どれだけ良いツールでも移行が面倒だと試す気にならない。

実際に使って驚いたのはComposerの精度。Cmd+Shift+IのComposerで複数ファイルにまたがる変更を一気に自然言語で指示できる。

```typescript
// api/users/route.ts に対して
// 「ページネーション追加して、limitとpageのバリデーションもつけて、
//  型は lib/types.ts と合わせて」と指示すると…

// route.ts の変更だけじゃなく、
// lib/types.ts に PaginationParams を追加してきた
// lib/validation.ts にバリデーターも生成した
// しかも既存の型定義と矛盾しない形で

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const params = parsePaginationParams(searchParams); // lib/validation から持ってきた
  const { data, total } = await db.user.findManyPaginated(params);
  return Response.json({ data, total, page: params.page });
}
```

Copilotが「次の1〜2行を補完する」という感覚だとすると、CursorのComposerは「この機能を実装する」という粒度で動く。実際、かなり違う体験だった。

一回だけ、Composerが生成したコードが一見正しく見えて後でバグに気づいた、ということがあった。既存の型定義を読んで矛盾なく変更してくれたはずが、optional fieldの扱いが微妙にずれていて、テストを書いて初めて発覚した。精度が高いほどレビューをさぼりやすくなるので、その点は意識的に気をつけてる。

ただ、落とし穴がある。Cursorはデフォルトでコードをクラウドに送る（オプトアウトはできるけど）。うちのチームには社内ポリシー上、ソースコードを外部サービスに送れないプロジェクトがあって、そこではそのまま使えない。Privacy Modeをオンにすれば回避できるんだけど、チーム全員に徹底させるのは正直面倒で、これが個人採用に留まった一番の理由。

料金はPro Planが月$20。Copilot Individualの倍だけど、機能差を考えると個人開発者なら十分ありだと思う。個人で使うなら試す価値は絶対にある。特にフロントエンド中心の開発なら、Composerがかなり仕事のやり方を変える。

## Codeium（Windsurf）：無料で使えるレベルが想定外に高かった

Codeiumは以前試したことがあって、「補完は悪くないけどCopilotには及ばない」という印象を持ってた。でも2025年後半にWindsurfというIDEを出してから話が変わった。

WindsurfもVSCodeベースで、Cursorと直接競合する位置づけ。無料プランが意外と使えて、月あたりのメッセージ上限はあるものの、普通の業務量なら無料で回せる日もあった。

特に印象的だったのは補完の速度。体感でCopilotより反応が速い瞬間があって、最初はプラシーボかと思ったけど、同じファイルをしばらく触ってると確かに速い。ローカルのコンテキストをキャッシュしてるんだろうと思う（確証はないけど）。

Cascadeというエージェントモードも試した。こんなシナリオで使った：

```python
# FastAPIのエンドポイントに対して
# 「このエンドポイントのユニットテストを書いて、
#  モックはpytestのfixtureで管理して、
#  異常系も3パターン入れて」と指示

@pytest.fixture
def mock_db_session(mocker):
    return mocker.patch("app.database.get_session")

@pytest.mark.parametrize("invalid_payload,expected_status", [
    ({"email": "not-an-email"}, 422),
    ({}, 422),
    ({"email": "a@b.com", "role": "superadmin"}, 403),  # 権限エラーも拾ってた
])
def test_create_user_validation_errors(mock_db_session, client, invalid_payload, expected_status):
    response = client.post("/api/users", json=invalid_payload)
    assert response.status_code == expected_status
```

Cascadeの精度はCursorのComposerと比べると一段落ちる感じがした。特に複数ファイルの依存関係を把握した上での変更は、Cursorの方が整合性が高い。ただ、プロンプトの工夫で改善できる余地もある。

まず無料で試せるのが強い。特にスタートアップや個人開発者で、ツールにお金をかけにくい状況なら、Codeiumの無料プランは現実的な出発点になる。

## Tabnine：プライバシー要件がある組織なら実質これ一択

**率直に言う。プライバシー制約がない状況でTabnineをあえてメインツールに選ぶ理由は、今のところ薄い。**

Tabnineの最大の差別化はSelf-hosted deploymentとコードの完全なローカル処理。チームのコードが一切外部サーバーに飛ばない。金融系、医療系、あるいは「OSSじゃないコードを外に出したくない」という会社にとって、これは機能の差じゃなくて要件の差だ。

実際に使うと、補完の質は正直Copilotより一段落ちる感じがする。特にPythonやTypeScriptのモダンなイディオムは、Cursorに慣れた目だと物足りない。ただ、ブロック補完（次の数行をまとめて提案する機能）は地味に便利で、ボイラープレートを大量に書く作業でのストレスは確かに減った。

一つ困ったのがVSCodeとJetBrainsで体験が結構違うこと。チームにIntelliJユーザーが混在してると体験の統一が難しい。これは導入前にベンダーに確認した方がいい。

プライバシー要件がある業務では、ほぼ必然的にTabnineが選択肢になる。それ以外の文脈では、ほかのツールを先に試した方がいい。

## Amazon Q Developer：AWSユーザーへの局所特化型アシスタント

AWSのサービスをどれだけ触るか、によって評価が180度変わるツール。

TypeScriptで普通のWebアプリを書いてる分には、Cursorの方が上。でもAWS CDKやCloudFormationを触り始めると話が変わる：

```typescript
// CDKスタックの記述で
// 「Lambda関数からDynamoDBに書き込むスタックを書いて、
//  IAMは最小権限で」と指示

const writePolicy = new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  actions: [
    'dynamodb:PutItem',
    'dynamodb:UpdateItem',
    // ← GetItem は含まれなかった。ちゃんと最小権限になってた
  ],
  resources: [
    table.tableArn,
    `${table.tableArn}/index/*`,  // GSIのARNも別途追加してた — これは気が利いてる
  ],
});
```

CloudFormationテンプレートの補完、AWS CLIコマンドの提案、CloudWatch Logsのクエリ補助あたりは確かに使える。AWSのドキュメントをかなり深く学習してるのが伝わる。

ただ、AWS以外のコンテキストでは急に普通になる。Next.jsのルーティングについて質問したら、App Routerではなくpagesディレクトリ前提のパターンを提案してきた。汎用的なコーディングアシスタントとしては弱い。

あと、VSCode拡張機能のUIが若干重い印象がある。補完が出るまでのラグが他のツールより目立つ場面があった——M2 MacBook Pro環境での話なので環境差はあると思うけど。

AWSアカウントがあれば無料枠で試せるのは強みで、インフラ中心のエンジニアか、AWSサービスを頻繁に触る人は手元に入れておく価値はある。それ以外のユースケースでメインツールにするには局所的すぎる。

## 結局、私はCursorを選んだ

2週間使い比べて、個人の開発環境ではCursorをメインにした。理由は単純で、Composerで複数ファイルをまたいだ変更をやらせたとき、他のツールとの差が一番はっきり出た。「AIが補完してくれる」じゃなくて「AIと一緒に実装してる」という感覚に最も近かった。

チーム全体の導入については、まだCopilot Businessを使い続けてる。Cursorのチームプランのコスト試算が終わってないこと、Privacy Modeの運用ルール整備が面倒でまだ手が回ってないこと——これが正直な現状。近いうちにちゃんとやる予定（と言いつつ時間が経ちそうな気はしてる）。

用途別でまとめると：プライバシー要件が厳しい業務ならTabnine。AWSを大量に触るならQ Developerを手元に置いておく。コストを抑えつつエージェント機能も使いたいなら、Codeium（Windsurf）の無料プランから始めるのが現実的。

一つだけはっきり言えるのは、「Copilot一択」という時代はもう終わってるということ。半年おきに見直す価値はある。少なくとも私はそう判断してる。

<!-- Reviewed: 2026-03-07 | Status: ready_to_publish | Changes: removed 4x formulaic "このセクションの結論" subheadings, added Composer gotcha (optional field bug), bold opener for Tabnine section, concrete Next.js App Router detail in Amazon Q section, varied paragraph lengths, expanded meta_description to ~148 chars -->
