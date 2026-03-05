---
title: "AutoGen vs LangGraph vs CrewAI: 2026年、実際に2週間使い倒して分かったこと"
emoji: "🚀"
type: "tech"
topics: ["ai\u30a8\u30fc\u30b8\u30a7\u30f3\u30c8", "autogen", "langgraph", "crewai", "llm"]
published: true
---

去年の11月、10人規模のスタートアップで顧客サポートの自動化システムを作ることになった。要件はシンプルに見えた——ユーザーの問い合わせを分類して、適切なナレッジベースを検索して、必要なら人間にエスカレーションする。ところがこれが思った以上に複雑で、結局3つのフレームワークを本番に近い環境で試す羽目になった。

2週間、AutoGen 0.4.2、LangGraph 0.3.x、CrewAI 0.95（当時の最新）を並べて叩いた記録を書く。ベンチマーク記事みたいなきれいな数字は出てこない。でも「どっちを選ぶべきか」という問いに対して、自分なりの答えは出た。

## AutoGen 0.4の正直な評価——会話は得意、デバッグは地獄

AutoGenはMicrosoftが作っているマルチエージェントフレームワークで、0.4系で内部構造がかなり変わった。旧来の`ConversableAgent`ベースのアーキテクチャから、`AG2`という非同期ランタイムに移行している。この変更でパフォーマンスは上がったけど、ドキュメントとコードの乖離がひどくて最初の3日間は公式ドキュメントとGitHubのIssueを行き来し続けた（[Issue #4821](https://github.com/microsoft/autogen/issues/4821)あたりが参考になった）。

一番気に入ったのは、エージェント同士の会話フローが本当に自然に書けること。

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_ext.models.openai import OpenAIChatCompletionClient

# 実際のプロジェクトで使ったシンプルな構成
async def run_support_pipeline(user_query: str):
    model_client = OpenAIChatCompletionClient(
        model="gpt-4o",
        # コストを抑えるためmax_tokensは意識的に絞る
        max_tokens=1024,
    )

    classifier = AssistantAgent(
        name="classifier",
        model_client=model_client,
        system_message="問い合わせを billing/technical/general に分類してください。JSONで返すこと。",
    )

    resolver = AssistantAgent(
        name="resolver",
        model_client=model_client,
        system_message="分類された問い合わせに対して、ナレッジベースから回答を作成してください。",
    )

    team = RoundRobinGroupChat(
        [classifier, resolver],
        max_turns=4,  # ここを設定しないと無限ループのリスクがある——実際に1回やらかした
    )

    result = await team.run(task=user_query)
    return result

asyncio.run(run_support_pipeline("請求書が届いていません"))
```

ここで一度やらかした話をしておく。`max_turns`を設定し忘れた状態でテストを回したら、エージェントたちが「私はclassifierです、あなたはresolverですね」みたいな自己紹介ループを延々と続けて、気づいたらOpenAIのAPIコストが数ドル飛んでいた。笑えない。

AutoGenが本領を発揮するのは、エージェント間の会話が複雑に絡み合うシナリオ——たとえばコードを書くエージェントとレビューするエージェントが行き来するような場合。でも「この処理を終えたら次のステップに進む」という明確な制御フローが必要な場合、正直に言うと向いていない。非決定的な動きが多くて、同じ入力でも出力がブレる。プロダクション環境でこれが許容できるかどうかは、ユースケース次第だと思う。

**実用的な観点から**: AutoGenはプロトタイプ段階か、会話の自然さが最優先のプロジェクトに向いている。デバッグのしやすさを重視するなら次のLangGraphを見てほしい。

## LangGraphは「めんどくさい」が正義だった

LangGraphはLangChainチームが作っているグラフベースのフレームワークで、正直、最初は過剰エンジニアリングだと思っていた。ノードとエッジを定義して、状態を明示的に管理して……これって普通にコード書けばよくない？

でも2日ほど使ったら考えが変わった。

LangGraphの核心は「状態がすべて明示的」なこと。各ノードが受け取るものと返すものが型で定義されていて、どこで何が起きているか追いやすい。AutoGenで悩んだ「なぜこのエージェントがこの返答をしたのか」という問題が、LangGraphではかなり減った。

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages

# 状態の定義がコードの設計図になる
class SupportState(TypedDict):
    messages: Annotated[list, add_messages]
    category: str | None
    escalate: bool

def classify_node(state: SupportState) -> dict:
    # LLMを呼んで分類する（省略）
    # ここで返す値が次のノードへの入力になる
    return {
        "category": "technical",
        "messages": [{"role": "assistant", "content": "技術的な問い合わせです"}]
    }

def resolve_node(state: SupportState) -> dict:
    # categoryに基づいて回答を生成
    if state["category"] == "billing":
        # 請求系は別のツールを呼ぶ
        pass
    return {"messages": [{"role": "assistant", "content": "回答..."}]}

def should_escalate(state: SupportState) -> str:
    # 条件分岐をここで明示的に書く——これが地味に重要
    if state["escalate"]:
        return "human_handoff"
    return END

# グラフの組み立て
builder = StateGraph(SupportState)
builder.add_node("classify", classify_node)
builder.add_node("resolve", resolve_node)
builder.add_edge("classify", "resolve")
builder.add_conditional_edges("resolve", should_escalate)
builder.set_entry_point("classify")

graph = builder.compile()
```

このコードを見ると「冗長だな」と感じる人もいると思う。でも3ヶ月後に自分が書いたコードを読み返す場面を想像してほしい。どのエージェントがどの条件で何をするのか、LangGraphなら一目で分かる。AutoGenで書いたコードは……正直、2週間後に読んでもよく分からなかった。

LangChainのエコシステムとの統合も強い。LangSmith（トレーシングツール）がそのまま使えるので、本番環境でのデバッグが格段に楽だった。ただLangChainの負債も引き継ぐ——deprecated警告がうるさい、バージョン間の破壊的変更が多い、という問題は依然としてある。

**実用的な観点から**: チームで長期保守するプロジェクト、複雑な条件分岐がある処理フロー、本番環境での可観測性が必要な場合はLangGraph一択だと思った。

## CrewAIは「速さ」に価値がある——ただし条件付きで

CrewAIは三つの中で一番すぐ動かせる。「役割を持つエージェントたちがタスクを協力してこなす」というメンタルモデルが分かりやすくて、ドキュメントを30分読めばとりあえず何か動く。

```python
from crewai import Agent, Task, Crew, Process

# ロールプレイ感覚でエージェントを定義できる
classifier = Agent(
    role="カスタマーサポート分類エージェント",
    goal="問い合わせを正確に分類する",
    backstory="5年のカスタマーサポート経験を持つ分析のプロ",  # backstoryが地味に効く
    verbose=True,
)

resolver = Agent(
    role="テクニカルサポートエージェント",
    goal="技術的な問題を解決する",
    backstory="エンジニアリング出身のサポートスペシャリスト",
    verbose=True,
)

classify_task = Task(
    description="次の問い合わせを分類してください: {query}",
    agent=classifier,
    expected_output="billing/technical/general のいずれか",
)

resolve_task = Task(
    description="分類された問い合わせに回答してください",
    agent=resolver,
    context=[classify_task],  # タスク間の依存関係
    expected_output="ユーザーへの回答文",
)

crew = Crew(
    agents=[classifier, resolver],
    tasks=[classify_task, resolve_task],
    process=Process.sequential,
)

result = crew.kickoff(inputs={"query": "パスワードをリセットしたい"})
```

見た目はきれいだし、エージェントに`backstory`を与えると品質が上がるのも面白い（これは実際に効果があった——なぜかは正直よく分からない）。

ただ、私が不満を感じたのは制御フローの柔軟性が低いこと。CrewAIは内部で何をしているかが分かりにくくて、「なぜこのタスクがこの順番で実行されているのか」を追うのが難しい。階層型プロセス（`Process.hierarchical`）を使おうとしたら、マネージャーエージェントが予想外の判断をして処理が止まる場面が何度かあった。デバッグツールもLangSmithほど充実していない。

あと、LLMプロバイダーへの依存がやや強い。私のプロジェクトはAzure OpenAIを使っていたんだけど、設定まわりで詰まる箇所があって1時間溶けた。

**実用的な観点から**: PoC段階でとにかく速く動かしたい、チームメンバーがLLMに詳しくない、という場合はCrewAIが最短経路。ただし本番に持っていくタイミングで別のフレームワークへの乗り換えを検討することになるかもしれない——実際に私がそうなった。

## コスト・速度・デバッグの現実

2週間のテストで分かったことをまとめると、こんな感じ。

**APIコスト**: 同じタスクに対して、CrewAIが一番トークンを使いがちだった。抽象化の高さと引き換えに、内部でシステムプロンプトが膨らむ。LangGraphは自分でプロンプトを制御できるので、無駄がない。AutoGenは中間くらい——ただしループを適切に制御しないとスパイクする。

**レイテンシ**: 差は少ないが、LangGraphが一番予測しやすい。AutoGenは会話のターン数によって大きくブレる。

**デバッグ**: LangGraph（LangSmith連携あり）> AutoGen（ログは充実してるが読みにくい）> CrewAI（ブラックボックス感がある）の順。

One thing I noticed（これ日本語で言うと変なので英語のまま使う）: フレームワークの抽象度が高いほど、最初の立ち上がりは速いが、想定外の動作に対処する時間が増える傾向がある。CrewAIで「なんか変な動きをする」を解決するのに費やした時間と、LangGraphで最初のグラフを設計する時間は、結局あまり変わらなかった。

100%確信はないが、チームが5人以上でLLMアプリを長期運用するなら、初期のセットアップコストを払ってでも可観測性の高いフレームワークを選ぶ方が、トータルコストが下がると思っている。

## 結局、何を選ぶか——私の答え

曖昧にしたくないので直接書く。

**LangGraphを選ぶ**: 本番運用する、チームで保守する、処理フローが複雑、デバッグのしやすさを重視する——これらのどれか一つでも当てはまるなら。最初は面倒くさく感じるが、その面倒くささがコードベースの堅牢さに直結する。

**AutoGenを選ぶ**: コードを書いて実行して評価するサイクルが中心のユースケース（Microsoftのデモがこれが多い）、あるいはエージェント同士の自由な会話を試したい研究・実験的なプロジェクト。プロダクション前提なら注意が必要。

**CrewAIを選ぶ**: 「とにかく2日でデモを作る」という場合。ロールベースの設計が直感的なので、LLMになじみのないステークホルダーに見せるプロトタイプには向いている。ただし私は本番にCrewAIをそのまま持ち込む判断はしなかった——もしかしたら特定のユースケースではいけるかもしれないが、少なくとも私のケースでは制御フローの予測不能さがリスクだった。

結局、今の本番システムはLangGraphで書き直した。最初のLangGraph実装を作るのに3日かかったけど、その後の改修が楽で、先週新しいエスカレーションルールを追加するのに30分かからなかった。それが答えだと思っている。
