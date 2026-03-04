---
title: "AutoGen vs LangGraph vs CrewAI: 2026年に実務で使えるのはどれか"
emoji: "🚀"
type: "tech"
topics: ["autogen", "langgraph", "crewai", "ai\u30a8\u30fc\u30b8\u30a7\u30f3\u30c8", "python"]
published: true
---

先月、社内のデータレポート生成フローを自動化するタスクを引き受けた。毎週月曜日に複数のデータソースを集約して、SlackにKPIサマリーを投稿するやつだ。手動でやると毎週1〜2時間かかっていて、チーム全員が嫌がっていた。

この手の仕事はAIエージェントにうってつけに見える。でも実装に入る前に「どのフレームワークを使うか」で詰まった。AutoGen（Microsoft）、LangGraph（LangChain）、CrewAI——この3つをそれぞれ実際のプロジェクトに当てはめながら約2週間テストした。その結果をまとめる。

結論から言う。**LangGraphを選んだ**。ただし、それには理由があるし、ユースケースによってはCrewAIの方がはるかに合理的な選択になる。AutoGenは——正直言って——使うシーンをかなり選ぶ。

## AutoGen v0.4: APIが激変して最初の半日を溶かした

GitHubのスター数が多くドキュメントも充実していたので、最初はAutoGenから触り始めた。ただ、すぐに壁にぶつかった。

v0.4（2025年初頭リリース）でAPIが根本から書き直されている。v0.2以前のコードはほぼ動かない。Stack OverflowやQiitaの記事の多くがまだ古いAPIで書かれていて、最初の半日を「なぜimportが通らないのか」のデバッグに費やした。`autogen-agentchat`と`autogen-core`を両方インストールしないといけないとか、公式ドキュメントのどこに書いてあるの、という話だ（GitHub Issue #4782あたりで同じ悩みを持つ人たちが大量にいる）。

v0.4の設計思想はマルチエージェントの「会話」を中心に置いている。`AssistantAgent`同士がメッセージをやり取りして、`RoundRobinGroupChat`や`SelectorGroupChat`でターンを管理する。

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main():
    model_client = OpenAIChatCompletionClient(model="gpt-4o")

    # データ収集担当エージェント
    collector = AssistantAgent(
        name="DataCollector",
        model_client=model_client,
        system_message=(
            "あなたはデータ収集の専門家です。"
            "与えられたデータソースから必要な情報を抽出してください。"
        ),
    )

    # レポート作成担当エージェント
    reporter = AssistantAgent(
        name="Reporter",
        model_client=model_client,
        system_message=(
            "あなたはビジネスレポートの専門家です。"
            "DataCollectorから受け取った情報を元に簡潔なKPIサマリーを作成してください。"
        ),
    )

    team = RoundRobinGroupChat(
        [collector, reporter],
        max_turns=6,  # 必ず設定すること——ないと普通に無限ループする
    )
    result = await team.run(
        task="先週の売上データを分析してSlack投稿用のサマリーを作成してください"
    )
    print(result.messages[-1].content)

asyncio.run(main())
```

このパターン自体は直感的で、複数のエージェントが「議論」しながら答えに近づくのは面白い。研究用途や複雑な推論が必要なタスクには向いている。

本番で使うとき致命的になり得るのが、**実行フローの制御が難しい**点だ。エージェントが自律的に会話するのはいいが、「このステップが終わったら必ずここに来る」という保証がない。デバッグ用のログも大量すぎて構造化されておらず、何が起きているか追うのが辛い。自分のユースケース（決まったパイプラインを確実に実行する）には合わなかった。

一方で、AutoGenが明らかに輝くのは**探索的なタスク**だ。「このデータセットから面白いパターンを見つけてほしい」みたいな、正解が事前に定義できないケース。エージェント間の自律的な対話でアイデアが広がる感覚は、他の2つには出せない。

## LangGraph: 最初は「過剰設計じゃないか」と思った

正直、LangGraphに最初に触れたとき、「なんでこんな複雑なの」と思った。グラフを定義して、ノードを追加して、エッジを張って——シンプルなタスクに対してボイラープレートが多すぎる印象だった。

でも2〜3日使い込んだら、この設計の意味がわかった。

LangGraphの本質は**明示的な状態管理**だ。エージェントの実行状態をすべて`State`（TypedDictで定義）に持ち、ノード間のデータの流れが全部見える。条件分岐も`add_conditional_edges`で明示的に定義する。

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class PipelineState(TypedDict):
    raw_data: str
    analysis: str
    report: str
    retry_count: int
    messages: Annotated[list, operator.add]

def collect_data(state: PipelineState) -> dict:
    # 実際にはAPIを叩いてデータを取得する
    return {"raw_data": "Q4売上: ¥120M, 前年比+15%..."}

def analyze(state: PipelineState) -> dict:
    result = llm.invoke(f"以下を分析してください: {state['raw_data']}")
    return {"analysis": result.content}

def generate_report(state: PipelineState) -> dict:
    report = llm.invoke(f"分析結果からレポートを作成: {state['analysis']}")
    return {
        "report": report.content,
        "retry_count": state.get("retry_count", 0),
    }

def quality_check(state: PipelineState) -> str:
    # レポートが短すぎたら最大2回リトライ
    if len(state["report"]) < 200 and state.get("retry_count", 0) < 2:
        return "retry"
    return "publish"

builder = StateGraph(PipelineState)
builder.add_node("collect", collect_data)
builder.add_node("analyze", analyze)
builder.add_node("report", generate_report)

builder.set_entry_point("collect")
builder.add_edge("collect", "analyze")
builder.add_edge("analyze", "report")
builder.add_conditional_edges(
    "report",
    quality_check,
    {"retry": "analyze", "publish": END},
)

graph = builder.compile()
result = graph.invoke({"raw_data": "", "retry_count": 0})
```

このコードを見れば、実行フローが一目でわかる。`collect → analyze → report`、品質が低ければ`analyze`に戻る。デバッグのときに「いまどのノードにいるのか」が常に明確なのが助かる。

一つ気になったのが、LangSmithとの連携だ。LangChainの有料サービスだけど、デバッグには本当に助かった。エージェントの各ステップで何が起きているかがブラウザ上のUIで追えるので、ローカルのログを必死に読む必要がない。無料プランでも月2,000トレースまで使えるので個人プロジェクトなら十分だと思う。

本番運用で特に評価が高かったのが、**ヒューマン・イン・ザ・ループ**の設定のしやすさだ。`interrupt_before`や`interrupt_after`を使えば、特定のノードの前後でグラフを一時停止して人間のレビューを挟める。「完全自律エージェント」ではなく人間の承認フローが必要な業務アプリでは、このコントロールが効く。

## CrewAI: プロトタイプを一番速く作れたが、本番は要注意

CrewAIは3つの中で一番「書いていて楽しい」フレームワークだった。役割（Role）を持ったエージェントをチームとして編成し、タスクを自然言語で定義するだけで動く。

```python
from crewai import Agent, Task, Crew

analyst = Agent(
    role="データアナリスト",
    goal="売上データを分析して重要なインサイトを抽出する",
    backstory="5年以上のデータ分析経験を持つエキスパート",
    verbose=True,
)

writer = Agent(
    role="ビジネスライター",
    goal="分析結果を分かりやすいレポートにまとめる",
    backstory="経営陣向けレポート作成が得意なコミュニケーター",
    verbose=True,
)

analysis_task = Task(
    description="Q4売上データを分析し、前四半期比・前年比・主要KPIの変化を特定してください",
    expected_output="箇条書きで5〜7個のインサイト",
    agent=analyst,
)

report_task = Task(
    description="分析インサイトを元に、Slack投稿用の200字以内のサマリーを作成してください",
    expected_output="Slack投稿用テキスト",
    agent=writer,
    context=[analysis_task],  # 前のタスクの出力を参照——これを忘れると詰む
)

crew = Crew(agents=[analyst, writer], tasks=[analysis_task, report_task])
result = crew.kickoff()
```

コード量が少なく、役割が自然言語で定義されているので、非エンジニアにも見せやすい。チームのプロダクトマネージャーに見せたとき「これならエージェントの動きがイメージできる」と言っていた。

で——ここが肝心なのだけど——CrewAIには**実行フローのコントロールが根本的に弱い**という問題がある。エージェントが「そのタスクを完了した」と判断するのが内部ロジックに依存していて、外からハンドリングしにくい。タスクが複雑になると、エージェントが予期しない行動をとることがある。本番で「なぜこの出力になったのか」を追うのが辛い。

CrewAI 0.80以降（2025年末リリース）でFlowsという機能が追加されて状態管理が多少改善されたが、LangGraphの明示性には追いついていない印象だ。100%確信があるわけではないが、CrewAI Flowsは根本的な設計思想を変えるものではなく、既存のアーキテクチャに後付けした感が否めない。

## やらかし話と、実際のベンチマーク結果

やらかした話をひとつ。CrewAIで最初に作ったとき、`context`パラメーターを渡し忘れていて、`writer`エージェントが`analyst`の出力を参照できずに動いていた。出力はそれっぽく見えたのに、実は前のタスクの結果を全く使っていなかった。LangGraphなら状態の型定義でコンパイル時に気づけるケースだった。それに気づくまで30分くらいかかった。

2週間の実装テストで見えた比較を率直に書く。

**デバッグしやすさ**: LangGraph > AutoGen > CrewAI。LangGraphは状態が全部見えるしLangSmithがある。AutoGenはログが多いが構造化されていない。CrewAIは何が起きているか一番追いにくかった。

**初期開発速度**: CrewAI > AutoGen > LangGraph。CrewAIで最初の動くプロトタイプができたのは2時間足らず。LangGraphは状態の設計とグラフの接続で丸一日かかった。

**本番での予測可能性**: 1週間稼働させた感触では、LangGraphが一番安定している。CrewAIは稀に予期しない出力を返した——特にデータが空の場合の処理が弱かった。AutoGenはエラー回復が意外と自然で、エージェント間で問題を「相談」して解決しようとする挙動がある（これが良いかどうかはユースケース次第だが）。

## 結局どれを選ぶか

ポジションを明確にする。

**本番運用するエージェントパイプラインを作るなら、LangGraphを使う**。初期のセットアップコストは高いが、状態管理の明示性とデバッグのしやすさは他の2つにない強みだ。条件分岐が複雑なワークフロー、監査ログが必要な業務アプリ、ヒューマン・イン・ザ・ループが必要なシステム——こういった場面では迷わずLangGraph。

**素早くプロトタイプを作って検証したいなら、CrewAIから始める**のが合理的だ。コンセプト検証、ステークホルダーへのデモ、PoC——CrewAIなら数時間で動くものができる。本番移行を見据えているなら、その時点でLangGraphへの書き直しを検討すればいい。

**AutoGenは、複数のLLMが協調してより良い答えを導き出す探索的なユースケース向け**。コードレビューを複数の視点で評価させる、複雑なリサーチクエスチョンをエージェントが協調して解く——こういった場面なら、AutoGenのアーキテクチャが一番しっくりくる。

自分のデータパイプラインはLangGraphで本番に上げた。今は毎週月曜朝8時にエージェントが自動でレポートを生成してSlackに投稿している。最初の1週間は状態の型定義でハマりまくったが、今は安定して動いている。チームの月曜朝の作業が1〜2時間削減されたので、結果的には正解だったと思っている。

この3つのフレームワークは今後も動き続ける。AutoGenはv0.5の開発が進んでいるし、LangGraphもLangChain本体との統合が深まっている。半年後にはまた状況が変わっているかもしれない。ただ、設計思想の軸——「自律的な会話」「明示的なグラフ」「役割ベースのチーム」——は当面変わらないはずなので、フレームワークを選ぶ基準として使えると思う。
