---
title: "プロダクションAIパイプライン: 1万5千回の実行から学んだこと"
emoji: "🚀"
type: "tech"
topics: ["ai", "llm", "pipeline", "production", "openai"]
published: true
---

去年の11月、深夜2時にSlackの通知で目が覚めた。コンテンツ分類パイプラインが6時間動き続けて、OpenAI APIの費用を$340溶かしていた。結果物の約70%はゴミだった。原因はリトライロジックのバグ——なぜリクエストが失敗しているかを区別できていなかった。`context_length_exceeded`エラーが出るたびに問答無用で3回リトライしていた。朝には手遅れだった。

あの夜から、AIパイプラインを「ちょっとした配管付きAPIコール」として扱うのをやめた。そこから1万5千回以上の処理をこなしてきた——ドキュメント分類、コードレビュー自動化、12人のエンジニアチーム向け社内Q&Aシステム。実際に何が壊れるか、かなり具体的な意見を持てるようになった。

## リトライロジックは一律じゃ絶対ダメ

最初に実装したのはシンプルなものだった。`tenacity`を使って指数バックオフを設定して、「はい、できた」と思っていた。

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
def call_llm(prompt: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content
```

技術的には間違っていない。問題は、これが危険なくらい不完全だということだ。OpenAIのエラーを全部同じように扱ってはいけない。それがまさに$340を一晩で溶かす原因になる。

`RateLimitError`はリトライすべき。レートを超えた、少し待てばいい。でも`InvalidRequestError`は？リトライしても意味がない。プロンプトがコンテキスト長を超えているか、不正なパラメータを渡しているかのどちらかだ。2回目も同じ理由で失敗する。あの深夜2時のインシデントは正確にこれだった——`context_length_exceeded`エラーが繰り返しキューに入り、毎回トークン代を課金されながら失敗し続けた。

ちゃんと分類するとこうなる:

- **リトライあり**: `RateLimitError`、`APITimeoutError`、`APIConnectionError`、`InternalServerError`（503）
- **リトライなし**: `InvalidRequestError`、`AuthenticationError`、`PermissionDeniedError`
- **内容を見てから判断**: `BadRequestError` — エラーメッセージを確認して決める

実際に使っているコードはこれ:

```python
import openai
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

RETRYABLE_EXCEPTIONS = (
    openai.RateLimitError,
    openai.APITimeoutError,
    openai.APIConnectionError,
    openai.InternalServerError,
)

@retry(
    retry=retry_if_exception_type(RETRYABLE_EXCEPTIONS),
    stop=stop_after_attempt(4),
    wait=wait_exponential(multiplier=2, min=5, max=120),
    reraise=True
)
def call_llm(prompt: str, model: str = "gpt-4o-mini") -> str:
    try:
        response = client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            timeout=30.0  # これを省くと無限に待つ羽目になる
        )
        return response.choices[0].message.content
    except openai.BadRequestError as e:
        # context_length_exceededはリトライ不可——即座に諦める
        if "context_length_exceeded" in str(e):
            raise ValueError(f"プロンプトが長すぎます: {len(prompt)} chars") from e
        raise
```

`timeout=30.0`は省略禁止だ。OpenAI APIはたまに——特に閑散時間帯に——90秒以上無音になることがある。タイムアウトなしだとワーカーがそこで詰まって、バッチジョブが数分ではなく数時間止まる。実際にそれで何時間も無駄にしたことがある。

リトライの上限を4回にしているのも理由がある。それ以上は大抵、一時的な問題じゃなく構造的な問題を踏んでいる。リトライを増やしてもレイテンシが伸びるだけで何も解決しない。

## コストが予想外に膨らむ3つのポイント

正直に言う。「トークンをちょっと使うくらいでしょ」と実際に声に出して言っていた。2ヶ月後にそれを言うのをやめた。

**システムプロンプトの重複。** 同じパイプラインで100件のドキュメントを処理するとき、システムプロンプトが500トークンなら、それが100回送られる。OpenAIのPrompt Caching（2024年8月から利用可能）を使えば、キャッシュされた入力トークンが90%割引になる——gpt-4oの場合。ただし1024トークン以上のプレフィックスじゃないとキャッシュが発動しない。短いシステムプロンプトは恩恵がない。安定したコンテンツを先頭に、動的なドキュメントごとのコンテンツをその後に配置する構造にすること。

**モデルの選び方。** デフォルトでgpt-4oを全部の処理に使っていた。プロトタイプで使い慣れていたし、「安全牌」という気持ちがあった。50件のドキュメントで両モデルの出力を手動比較したA/Bテストをやってみたら——単純な分類タスクで、gpt-4o-miniはgpt-4oと精度差が3〜5%に収まりながら、コストは約15〜20分の1だった。分類とキーワード抽出をminiに移し、複雑な推論や長文要約をgpt-4oに残した。その一変更だけで月のコストが約60%減った。

ただし「単純なタスク」かどうかは自分で検証すること。何が単純に当たるかはタスクによってかなり違う。

**出力トークンを制限していない。** gpt-4o-miniでは出力トークンのコストが入力の約4倍。構造化JSONを期待しているのに`max_tokens`を設定しないと、モデルが必要以上に長く応答することがある。`response_format={"type": "json_object"}`を使えば出力は一貫して短くなるが——モデルがスキーマの隙をついて冗長な`reasoning`フィールドを追加することがある。スキーマはできるだけ厳密に定義すること。`max_tokens`の上限設定は安価な保険だ。

## モデルの出力をそのまま信じるな

LLMが幻覚することは知っていた。でもその幻覚した出力が気づく前にデータベースを汚染できる、という事実には十分に対処できていなかった。

うちのパイプラインで起きた事故はこうだ。5つの特定カテゴリ名のひとつでドキュメントを分類するタスクで、モデルが時々微妙に違うカテゴリ名を返していた——類義語だったり大文字・小文字が違ったり、たまに完全に存在しない名前だったり。発生頻度が低かったので手動のスポットチェックでは表面化しなかった。それから下流システムでエラーが出始めて、追跡したら、不正なカテゴリがDBに保存されていた。データのクリーニングに半日かかった。

Pydanticで適切に定義されたenumを使うのが一番シンプルな解決策だった:

```python
from pydantic import BaseModel, field_validator
from enum import Enum
import json

class Category(str, Enum):
    TECHNICAL = "technical"
    BUSINESS = "business"
    LEGAL = "legal"
    MARKETING = "marketing"
    OTHER = "other"

class ClassificationResult(BaseModel):
    category: Category
    confidence: float
    reasoning: str

    @field_validator("confidence")
    @classmethod
    def confidence_range(cls, v: float) -> float:
        if not 0.0 <= v <= 1.0:
            raise ValueError("confidence must be between 0 and 1")
        return v

def classify_document(text: str) -> ClassificationResult:
    response = call_llm(
        f"以下のドキュメントを分類してください。JSONのみで回答してください。\n\n{text}"
    )
    try:
        data = json.loads(response)
        return ClassificationResult(**data)
    except (json.JSONDecodeError, ValueError) as e:
        # パース失敗を専用メトリクスとして追跡する
        metrics.increment("llm.output_parse_failure")
        logger.error(f"出力パース失敗: {e}, 生データ: {response[:200]}")
        raise OutputValidationError("モデル出力が期待するスキーマと一致しません") from e
```

One thing I noticed: パース失敗を専用メトリクスとして追跡することが、思ったより役に立つ。その割合が予想外に上がったとき、プロンプトが変わったか、モデルの挙動が静かに変化したかのどちらかだ。OpenAIはモデルを常にアナウンスせずに更新している——パース失敗率を監視することで、gpt-4o-miniの静かな挙動変化を2回キャッチした。1回は2024年末頃、構造化出力がやや冗長になってJSONの前にラッパーテキストを追加し始めたときだった。

`confidence`フィールドも活きている。0.5未満の結果は自動処理せず手動レビューキューに送っている。モデルが報告するconfidenceが実際の精度とどれだけ一致するかは議論の余地があるが——自分の経験だとあくまで大まかな指標で、キャリブレートされた確率としては信頼できない——それでも不確かなケースを仕分けるのには役立つ。

## 可観測性：一番軽視していた部分

これがこのリストで最も過小評価されているセクションだ。普通のAPIサービスなら計測は比較的シンプルだ——リクエストログ、エラー率、レイテンシ。AIパイプラインはそれに加えて新しい問題が乗っかってくる——非決定的な出力、リクエストごとにばらつくトークンベースのコスト、サイレントなモデル変更、特定の入力タイプにだけ現れる失敗モード。

リクエスト単位で追跡しているのはこれ: 入力トークン数、出力トークン数、レイテンシ、モデル名、成功/失敗、リトライ回数、パース成功/失敗、APIレスポンスの`usage`フィールドから計算した推定コスト。最後のものが特に重要だ。コスト配分をOpenAIのダッシュボードに任せてはいけない——リクエスト単位のデータがあって初めて、どのパイプラインステージや入力タイプが高いかわかる。「今月AIコストが40%上がった」だけでは何も解決できない。

メトリクスとは別に、ランダムサンプリングもやっている。完了したリクエストの1〜2%を人間がチェックする。その人間は自分で、大抵金曜の朝に30分。自動化された検証ではなく——入力と出力を実際に目で見て、それらが筋が通るかどうか確認する。面倒だ。でもこの習慣がプロンプトのデグレードを本番インシデントになる前に2回捕まえてくれた。自動メトリクスは「何かがおかしい」ことを教えてくれる。サンプリングは「何が」「なぜ」おかしいかを教えてくれる。

LangSmithは数週間試した。複数のブランチを持つ複雑なチェーンやマルチステップのエージェントループのデバッグには本当に有用だ。うちのパイプラインはそうじゃない——ほぼ線形だ——ので、オーバーヘッドに見合う効果がなかった。今はOpenTelemetryでGrafanaにトレースを送っている。初期設定は多いが、計測が透明でベンダーロックインもない。

告白すると——最初の3ヶ月間、可観測性をまともに作っていなかった。「とりあえず動かして後から追加しよう」という考えだったが、後から追加する方が最初から作るより何倍もしんどかった。特にレイテンシ分析とコスト追跡は、初期設計に含まれていないと後付けがかなり難しい。可観測性を後回しにした代償は、「後付けする工数」だけじゃない。早めに気づけたはずのインシデントのコストも含まれる。先に作れ、たとえ時期尚早に感じても。

## 今から作り直すなら、こうする

曖昧な「場合による」は言わない。ゼロから始めるなら実際こうする。

**SDKを直接使う。** パイプラインがLLMコールの連続と前後処理なら、LangChainは使わない。抽象化レイヤーがデバッグを難しくするし、メジャーバージョンアップのたびに何かが壊れる（langchain 0.2 → 0.3の移行は辛かった）。LangChainが効果を発揮するのは複雑なエージェントループやマルチステップRAGシステムのとき。絞られた本番パイプラインに対しては、不要な複雑さでしかない。

**リトライ可能なエラーとそうでないエラーを最初から分けること。** リトライ回数、最終結果、具体的な失敗理由をログに残す。深夜11時にデバッグするとき、それを自分に感謝することになる。

**リクエスト単位のコスト追跡を最初から入れること。** OpenAIのダッシュボードに一日あたりの支出アラートも設定する——自分は$50/日に設定している。これが2回発動した。両方ともバグが原因で、正当な負荷ではなかった。

**新しいプロンプトとモデル変更はカナリアデプロイすること。** トラフィックの10%を新バージョンに流して数時間メトリクスを見てから全体に展開する。新しいモデルが特定のタスクで必ずしも良い結果を出すとは限らない。gpt-4oのアップデート後に分類出力のノイズが増えて、以前のバージョンにロールバックしたことが2回ある。

**Pydanticで出力を検証すること。** モデルが指示に従うのが上手いからじゃない——実際かなり上手い——だが「かなり上手い」では、月15,000リクエストで1%の失敗率が150件の破損レコードを意味する場合には不十分だ。それは許容できない。

100%自信があるとは言えない——特にキューイングや並行管理周りは、10倍の負荷になったら設計を見直す必要があるだろう。でも、社内AIパイプラインを本番で動かし始めてちょっとガタが来てきた感じがあるなら、これは6ヶ月前に誰かに渡しておいてほしかったチェックリストだ。深夜2時のアラートは防げる。大抵は、そう。
