---
title: "プロダクションAIパイプライン構築: 1万回以上の実行から学んだ教訓"
emoji: "🚀"
type: "tech"
topics: ["ai", "llm", "python", "openai", "\u30d7\u30ed\u30c0\u30af\u30b7\u30e7\u30f3"]
published: true
---

去年の11月、Slackに通知が飛んできた。「ドキュメント要約パイプラインが2時間止まってる」。確認してみると、ログはほぼ空。エラーも出ていない。ただ静かに、何も処理されていなかった。

原因を追うのに3時間かかった。OpenAIのレート制限に静かに引っかかっていて、リトライロジックが事実上の無限ループに近い状態になっていた。コストもその間ずっと積み上がっていた。

あの日以来、私はAIパイプラインの「なんとなく動いてる」という感覚を一切信用しなくなった。

## ログを入れるまで、何も見えていなかった

最初の3ヶ月、私たちのパイプラインはほぼブラックボックスだった。入力を投げて、出力が返ってくる。それだけ。でも1日あたり数百回の実行が数千回になったとき、「なんか遅い」「たまに変な出力が出る」という報告が増え始めた。

再現できない。ログがないから。

LangSmith（0.1系が出た頃）を入れてみて、初めて実際の挙動が見えた。特定のドキュメントタイプでプロンプトのトークン数が跳ね上がっていること、リトライが発生しているのに成功扱いになっているケースがあること、レスポンスタイムの分布が思っていたより二峰性だったこと——全部、ちゃんとしたトレーシングを入れて初めてわかった。

正直、LangSmithは私の環境（小さなスタートアップ、エンジニア5人）ではコストがネックになってきたので、今は[Langfuse](https://langfuse.com)をセルフホストしている。機能的にはLangSmithとほぼ同等で、データが手元に置けるのも安心感がある。ただセルフホストのメンテコストは無視できないので、チームの状況次第だと思う。

ここが重要なんだが: LLMの呼び出しを単なるAPIコールとして扱わないこと。入力トークン数、出力トークン数、レイテンシ、モデルのバージョン、`finish_reason`——これを全部記録する。「後でいいや」と思うかもしれないが、デバッグで絶対後悔する。

実際に私が入れているのはこういうラッパーだ:

```python
import time
import logging
from openai import OpenAI, RateLimitError, APITimeoutError, BadRequestError
from dataclasses import dataclass
from typing import Optional

logger = logging.getLogger(__name__)
client = OpenAI()

@dataclass
class LLMResult:
    content: str
    input_tokens: int
    output_tokens: int
    latency_ms: float
    finish_reason: str
    model: str  # 実際に使われたモデルを記録（自動フォールバックがある場合に重要）

def call_with_observability(
    messages: list,
    model: str = "gpt-4o",
    max_retries: int = 3,
    request_id: Optional[str] = None,
) -> LLMResult:
    """
    エラータイプを区別しながらリトライ。
    RateLimitとTimeoutは再試行、BadRequest（コンテンツポリシー等）は即座に失敗させる。
    """
    attempt = 0
    while attempt < max_retries:
        start = time.monotonic()
        try:
            response = client.chat.completions.create(model=model, messages=messages)
            latency_ms = (time.monotonic() - start) * 1000

            result = LLMResult(
                content=response.choices[0].message.content,
                input_tokens=response.usage.prompt_tokens,
                output_tokens=response.usage.completion_tokens,
                latency_ms=latency_ms,
                finish_reason=response.choices[0].finish_reason,
                model=response.model,
            )
            _log_to_trace(request_id, result, messages)  # 自前のトレーシングに送る
            return result

        except RateLimitError as e:
            wait = 2 ** attempt  # 指数バックオフ
            logger.warning(f"RateLimit hit (attempt {attempt+1}), waiting {wait}s")
            time.sleep(wait)
            attempt += 1

        except APITimeoutError:
            logger.warning(f"Timeout (attempt {attempt+1})")
            attempt += 1

        except BadRequestError as e:
            # コンテンツポリシー違反などはリトライしても無駄——即座に上位に投げる
            logger.error(f"BadRequest — リトライ不可: {e}", extra={"request_id": request_id})
            raise

    raise RuntimeError(f"LLM call failed after {max_retries} attempts")
```

ポイントは`finish_reason`を見ること。`length`で終わっていたら出力が切り捨てられている——これに気づかず「AIの出力がおかしい」と半日悩んだことがある。`stop`以外が返ってきたら、それは正常ではない。

## LLM特有の失敗パターンは、普通のAPIと全然違う

Webアプリを8年作ってきて、API障害への対処には慣れていたつもりだった。でもLLMは別物だと痛感した。

普通のAPIなら「500が返ったらリトライ」でだいたい済む。LLMの場合、エラーの種類によって対処が根本的に違う:

- **レート制限**: 待てば治る。指数バックオフで十分。
- **タイムアウト**: 長いプロンプトやサーバー混雑が原因。リトライしていいが回数に上限を。
- **コンテンツポリシー違反**: 入力に問題がある。何度リトライしても同じ。即座に失敗させてログに残す。
- **コンテキスト長超過**: プロンプトが長すぎる。呼び出す前にトークン数を`tiktoken`でバリデーションする。

で、これを区別しないと何が起きるか。コンテンツポリシー違反でリトライを3回繰り返し、3回分のコストを無駄に払う。実際にやった（後述する）。

もう一つ厄介なのが、エラーではなく「壊れた成功」だ。`finish_reason: "content_filter"`が返ってきているのに、`choices[0].message.content`が`None`になっているケースがある。これを正常レスポンスとして処理してしまうと、`None`がひっそりとデータベースに積み上がっていく。明示的なチェックを必ず入れること。

## レート制限とコストで壁に当たった話

去年の夏、ユーザーが増えてパイプラインの実行回数が1日3000回を超えたあたりから、OpenAIのTPM（tokens-per-minute）制限に定期的に引っかかるようになった。

最初の対策は単純なリトライで、これである程度は解消した。でも根本的な問題は残っていた——同時実行数を制御していなかった。

asyncioで並列実行しているとき、同時に50リクエスト投げれば当然レート制限に引っかかる。`asyncio.Semaphore`で同時実行数を8に絞ったら、スループットはほぼ変わらずにエラーが激減した。この数字は環境によるので、自分のTierの制限値と照らし合わせて調整してほしい。

コスト面で効いたのはキャッシュだった。私たちのユースケースでは、同一ドキュメントを異なるユーザーが要求することが少なくなかった。入力のハッシュをキーにしてRedisに結果をキャッシュしたところ、月のAPIコストが約30%下がった。

ただ、キャッシュには落とし穴がある。プロンプトのバージョンが変わったときにキャッシュを無効化しないと、古いバージョンの出力を返し続ける。キャッシュキーにプロンプトのバージョンハッシュを含めることで解決したが、これに気づくまで「プロンプトを直したはずなのに挙動が変わらない」と1週間近く悩んだ。

## プロンプトをコードとして扱う、という当たり前の話

**やらかした話をする。**

プロンプトをコードベースにべた書きしていた時期、ある「小さな改善」をプロダクションに直接デプロイした。出力フォーマットのインストラクションを少し変えただけのつもりが、下流パーサーが期待するJSONの構造が微妙に変わってしまい、数百件のレコードが壊れたフォーマットで保存された。

バックフィルに2日かかった。コンテンツポリシー違反でリトライを繰り返していたのもこの時期で、なかなかしんどい週だった。

それ以来、プロンプトの管理方針を完全に変えた:

```python
# prompts/v2/document_summary.py

PROMPT_VERSION = "v2.1.0"

SYSTEM_PROMPT = """あなたは技術文書の要約専門家です。
以下のルールに従って要約してください:
- 必ずJSON形式で返す
- "summary"キーに日本語の要約（200字以内）
- "key_points"キーにリスト形式の要点（3〜5個）
- "confidence"キーに要約の確信度（0.0〜1.0）
"""

def build_prompt(document: str) -> list[dict]:
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"以下の文書を要約してください:\n\n{document}"},
    ]

# キャッシュキーやトレーシングに使う
METADATA = {
    "version": PROMPT_VERSION,
    "model": "gpt-4o",
    "created": "2026-02-10",
    "changelog": "confidence fieldを追加、key_pointsの数を3〜5に変更",
}
```

プロンプトをファイルとして管理し、バージョンをセマンティックバージョニングで追う。変更はPRで行い、キャッシュキーにもバージョンを含める。「プロンプトを変えた」という事実がgit logに残るので、何か壊れたときに「あの変更以降だな」とすぐ特定できる。

LangfuseやPromptLayerのようなプロンプト管理ツールも試したが、私のチームには少しオーバーエンジニアリングだった。シンプルにファイル管理 + gitで、今のところ十分に機能している。チームが大きくなったら再検討するかもしれない。

いいか、A/Bテストをどうするかという問題もある。今のところ、ユーザーIDのハッシュで振り分けて、下流タスクの成功率で評価している。自動評価はまだうまく機能しておらず、最終的には人間のレビューが必要なケースが多い——ここは正直、100%解決できているとは言えない。

## 結局、私が実際に使っている構成

1万回以上の実行を経た今のスタックはこうなっている。

**オーケストレーション**: LangChainは最初入れたが、バージョン間の破壊的変更に何度も踏まされて（0.1から0.2の移行が特につらかった）、今は薄いラッパーを自作している。チェーンが複雑になるとデバッグも難しくなる。ただ、シンプルなユースケースには今でも悪くないと思う。

**可観測性**: Langfuseをセルフホスト（Docker Compose）。コスト的にLangSmithより現実的だった。

**エラーハンドリング**: エラータイプごとに明示的に分岐。リトライは`tenacity`ライブラリが便利——自前で書くと細かいバグが出る。

**コスト管理**: 月次でモデル別・ユースケース別のコストを集計し、予算超過アラートをSlackに流している。毎月見ると、思わぬ箇所でトークンを食っていることがよくある。

**プロンプト管理**: Gitで管理、セマンティックバージョニング、キャッシュキーにバージョンハッシュを含める。

これが全員に合う構成だとは思わない。でも「なんとなく動いてる気がする」から「ちゃんと動いていることが確認できる」状態に移行したことで、夜中に叩き起こされる回数は明らかに減った。

一つだけ言えること: 可観測性は後回しにしない。最初の100回の実行からログを取れ。そうしないと、1万回目にやっと問題に気づくことになる。
