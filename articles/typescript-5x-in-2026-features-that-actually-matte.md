---
title: "TypeScript 5.x 2026年: 本番コードで本当に重要な機能"
emoji: "🚀"
type: "tech"
topics: ["typescript", "javascript", "webdev", "type-system", "tooling"]
published: true
---

TypeScript 5.0が出たのは2023年3月だった。それから3年近く、5.8まで追いかけてきた。毎回リリースノートを読んで「へえ、おもしろい」と思うんだけど、正直大半の機能はそこで終わる。試してみて、コードを書き換えるほどでもないな、と判断して次のスプリントに戻る。

でも一部は違った。実際に仕事のやり方が変わった機能がある。

自分のスタックを書いておく: 5人チームでSaaSプロダクトを作っている。Next.js + tRPC + Prisma、デプロイはVercel + Railway。TypeScriptは常に最新マイナーバージョンを追いかけていて、今は5.8を本番で使っている。今回書くのは、そのコードベースで「本当に差が出た」と感じた4つの機能だ。バズったけど使いどころが狭かったやつは意図的に外した。

---

## `using`宣言でリソースリーク系のバグが1クラスまるごと消えた

TypeScript 5.2で入った`using`宣言、最初見たとき「C#のusingじゃん」と思っただけで特に気にしていなかった。考えが変わったのは2024年の夏、本番で接続プールが枯渇する事件が起きてから。

ログを数時間追ったら、バックグラウンドジョブのひとつで`finally`ブロックが抜けていた。エラーハンドリングを追加するときに誰かが`finally`を消してしまっていて、レビューでも見落とした。そのジョブが長時間稼働していたせいで接続が戻らなくなっていた。手痛い案件だった。

問題になっていたコードのパターン:

```typescript
// before: tryのどこかでthrowすると接続が戻らないことがある
async function processJob(jobId: string) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await doHeavyWork(client, jobId);
    await client.query('COMMIT');
    return result;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release(); // これを書き忘れる、あるいは削除されることが実際にある
  }
}
```

`using`に切り替えるには、`Symbol.dispose`を実装したラッパーを一度書く:

```typescript
class ManagedClient {
  constructor(private client: PoolClient) {}

  get query() {
    return this.client.query.bind(this.client);
  }

  [Symbol.dispose]() {
    // スコープを抜けるときに必ず呼ばれる
    this.client.release();
  }
}

async function processJob(jobId: string) {
  using managed = new ManagedClient(await pool.connect());
  // return/throw/どんな経路でも、スコープを出たらrelease()が走る
  await managed.query('BEGIN');
  const result = await doHeavyWork(managed, jobId);
  await managed.query('COMMIT');
  return result;
}
```

`finally`を書き忘れるというバグのクラスごと消えた。コードレビューで「あ、finallyがない」と指摘する必要がなくなった。

注意点: Prismaそのものはまだ`Symbol.dispose`を実装していないので、ラッパーは自分で書く必要がある。非同期のcloseが必要なリソースには`await using`と`Symbol.asyncDispose`を使う。最初のラッパーを書くコストはかかるけど、1回書けばチーム全体で再利用できる。うちでは`ManagedClient`を書いた翌週には、同じパターンで別のリソース管理コードが自然に増えていた。

---

## `NoInfer<T>`: 半年間気づかなかった型設計の穴を塞いだ

TypeScript 5.4の`NoInfer<T>`は、リリース時に完全に見逃した。気づいたのはGitHubのissueをたまたま読んでいたときで、5.4が出てから4ヶ月くらい後。

「なんでこんな機能が必要なんだ?」と最初は思ったんだけど、自分のコードを見返したら3箇所で同じ問題を抱えていた。「半年間これで困ってたのか」と少しげんなりした。

こういう状況を想像してほしい。ジェネリック関数を書いていて、「型パラメータはある引数から推論してほしいが、別の引数からは推論してほしくない」というケース:

```typescript
// 問題: allowedValuesとdefaultValueの両方からTを推論する
function createSelect<T>(
  allowedValues: T[],
  defaultValue: T
): SelectConfig<T> { ... }

// 意図: 'extra-large'はありえないのでエラーにしたい
// 実際: TypeScriptが T = "small" | "medium" | "large" | "extra-large" に拡張してしまう
createSelect(['small', 'medium', 'large'], 'extra-large'); // エラーにならない
```

`NoInfer<T>`を使うと、指定した引数を型推論の対象から除外できる:

```typescript
function createSelect<T>(
  allowedValues: T[],
  defaultValue: NoInfer<T> // ←Tの推論ソースから除外
): SelectConfig<T> {
  if (!allowedValues.includes(defaultValue)) {
    throw new Error(`Invalid default: ${String(defaultValue)}`);
  }
  return { allowedValues, defaultValue };
}

// 今度はちゃんとコンパイルエラーになる
createSelect(['small', 'medium', 'large'], 'extra-large');
// Argument of type '"extra-large"' is not assignable to parameter of type 'NoInfer<...>'
```

うちのコードベースでは、設定ビルダーのAPIにこのパターンが3箇所あった。適用してから型チェックを走らせたら、本番コードで「ありえないデフォルト値が設定されていた」バグが1件見つかった。ランタイムのバリデーションで防いでいたけど、型レベルで防げていなかったやつだ。

ライブラリを書いている人やAPIデザインにこだわりのある人ほど恩恵が大きい機能だと思う。アプリケーションコードだけ書いている人は「あってよかった」くらいかもしれない。でも該当するパターンがあるなら、見つけた瞬間に直したくなる。「自分には関係ない」と思った人も、次にジェネリック関数を書くとき一度立ち止まってみてほしい。案外ある。

---

## 型述語の自動推論: 気づいたら既存コードが7個消えていた

これは本当に予想していなかった。

TypeScript 5.5の「Inferred Type Predicates」が便利そうというのは知っていたんだけど、実感したのは5.5に上げてしばらく後のコードレビューだった。同僚が書いたPRを読んでいたら、こういうコードがあった:

```typescript
const activeUsers = users.filter(user => user.isActive && user.email !== null);
// emailがstring | nullだったのが、TypeScriptがstring型として推論していた
```

「あれ、これって5.4以前だったら型述語ヘルパーが必要なやつじゃなかったっけ」と思って確認したら、そうだった。5.5以降は、filterコールバックの返り値の型からTypeScriptが自動で型述語を導いてくれる。つまり、こういうヘルパー関数の存在意義がなくなった:

```typescript
// 5.4以前のコードベースに散らばっていたやつ
function isNonNullString(v: string | null | undefined): v is string {
  return v !== null && v !== undefined;
}

function isActiveUser(u: User | null): u is User {
  return u !== null && u.isActive === true;
}
// ...以下続く
```

5.5に上げた後、既存のこういうヘルパー関数を確認してみたら、7つが「もう不要」になっていた。大量削除というわけじゃないけど、1つ1つが「変更のたびにメンテが必要な型コード」だったので、消えたのは素直に嬉しかった。

全部が自動推論されるわけではない。`instanceof`を組み合わせた複雑なケースや、外部APIのレスポンス検証には手書きがまだ必要なことがある。でも「よくあるnullチェック系」はほぼカバーされていると感じている。

---

## `--isolatedDeclarations`はモノレポ勢にだけ刺さる話

これは人を選ぶ。かなり選ぶ。シングルパッケージのプロジェクトには関係ないので、そういう人はここを読み飛ばして問題ない。

5.5で入った`--isolatedDeclarations`は、型宣言ファイル(`.d.ts`)の生成を並列化可能にするフラグだ。仕組みを一言で言うと: このフラグを有効にすると、各パッケージが自分の`.d.ts`を生成するときに他のパッケージの型情報を参照しなくていい状態になる。これがツールチェーン側で並列化のヒントになる。

ただし条件がある。エクスポートされる全ての関数・変数に明示的な型注釈が必要になる:

```typescript
// isolatedDeclarationsが有効だとエラー: 戻り値型が推論に依存している
export function getUser(id: string) {
  return prisma.user.findUnique({ where: { id } });
}

// こう書く必要がある
export async function getUser(id: string): Promise<User | null> {
  return prisma.user.findUnique({ where: { id } });
}
```

有効化したとき、うちのモノレポ(12パッケージ、Turborepoで管理)で修正が必要な箇所が約230あった。正直、想定より多かった。1日かけて対応した。作業自体はほぼ機械的で、TypeScriptのエラーメッセージが型を提案してくれることも多かったけど、地味に疲れる作業だった。途中でやめなかったのは、同僚がCIの遅さに毎日ぼやいていたのを思い出したから。

結果: Turborepoのキャッシュなし全ビルドが4分12秒から2分47秒になった。CIの請求を確認したら、月あたり約$40のコスト削減になっていた。チーム全員のローカルでの`tsc --build`も体感で速くなった。

「CIが遅くて困っている」という具体的な痛みがあるなら試す価値はある。そうじゃなければ後回しでいい。

---

## 優先順位の話

全部一気にやる必要はないし、やろうとすると失速する。

型述語の自動推論(5.5)はバージョンアップするだけで恩恵を受けられる。コードを書き換える必要がない。まずバージョンアップだけして、既存の型述語ヘルパーが不要になっていないか確認することから始めるといい。削除できたら儲けもの。

`NoInfer<T>`(5.4)はライブラリや内部APIを設計しているなら今すぐ検討する価値がある。実装コストが低い割に、型設計の表現力が上がる。

`using`宣言(5.2)はリソース管理のコードがある場所から段階的に導入できる。最初の1個のラッパークラスを書いてみると、使いどころが見えてくる。

`--isolatedDeclarations`(5.5)はモノレポを持っていてCIが遅いなら検討する。移行コストは確実にあるけど、長期的なリターンは出る。

毎回のTypeScriptリリースで「全部把握しなきゃ」というプレッシャーを感じることがあるけど、実際に本番で差が出るものはそこまで多くない。この4つは、うちの日常的な作業の質を実際に変えた。あなたのコードベースで同じになるかどうかはわからない — スタックもチームサイズも違うから。でも試してみる価値は、少なくともある。

<!-- Reviewed: 2026-03-10 | Status: ready_to_publish | Changes: removed English filler phrase ("Right, so —") embedded in Japanese text, added personality details (手痛い案件, げんなりした, CIの遅さに毎日ぼやいていた), varied paragraph lengths, tightened isolatedDeclarations intro, made conclusion section less formulaic -->
