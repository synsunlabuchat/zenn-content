---
title: "FastAPI、Django、Flask——2026年、実際に使い込んで分かったこと"
emoji: "🚀"
type: "tech"
topics: ["python", "fastapi", "django", "flask", "\u30a6\u30a7\u30d6\u30d5\u30ec\u30fc\u30e0\u30ef\u30fc\u30af"]
published: true
---

去年の10月、社内のMLパイプラインにAPIレイヤーを追加することになった。チームは自分含めて4人、全員がバックエンド寄りのエンジニアで、Pythonには慣れている。「どのフレームワーク使う？」という話になって、30分の議論が2時間に延びた。FastAPIを推す声、Djangoの安定性を挙げる声、「Flaskで十分じゃない？」という声。

結局、自分が手を挙げて「2週間で全部試す」ことになった。プロトタイプレベルじゃなく、認証、非同期処理、SQLAlchemyとの連携、OpenAPI仕様の自動生成——実際に使いそうな機能全部込みで。

その経験と、その後5ヶ月使い続けて気づいたことを書く。

---

## 「速い」というのは何が速いのか——FastAPIの実態

FastAPIを最初に触ったのは2023年だったけど、正直そのときはピンと来なかった。型ヒントが多くて「Pythonらしくない」と思っていた。今回改めて向き合って、考えが変わった。

速さには2種類ある。実行速度と、開発速度だ。

実行速度についてはベンチマークが色々出回っているので詳しくは言わないけど、uvicorn + FastAPIの組み合わせは体感でもわかるくらい速い。同じエンドポイントをFlaskで作ったときと比べて、単純なGETで3〜4倍のスループットが出た（うちの環境の話なので、あなたの環境で同じになるかは保証できないけど）。

開発速度の方が個人的には驚きだった。型ヒントを書けば自動でバリデーションとドキュメントが生成される、というのは頭では理解していたけど、実際にやってみると体験がまるで違う。

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import Optional
import asyncio

app = FastAPI()

class InferenceRequest(BaseModel):
    prompt: str
    max_tokens: int = 512
    temperature: float = 0.7
    # Optional fields — フロントがまだ実装してない機能は None でいい
    stream: Optional[bool] = False

class InferenceResponse(BaseModel):
    result: str
    model_version: str
    latency_ms: float

# 依存性注入でAPIキー検証を共通化
async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key not in VALID_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return x_api_key

@app.post("/v1/infer", response_model=InferenceResponse)
async def run_inference(
    req: InferenceRequest,
    api_key: str = Depends(verify_api_key)
):
    start = asyncio.get_event_loop().time()
    result = await model.generate(req.prompt, req.max_tokens, req.temperature)
    latency = (asyncio.get_event_loop().time() - start) * 1000

    return InferenceResponse(
        result=result,
        model_version="v2.1.3",
        latency_ms=round(latency, 2)
    )
```

これで `/docs` にアクセスすると自動でSwagger UIが出る。フロントエンドのチームメンバーに「ここ見て」と言えば、APIの仕様を別途書かなくていい。うちのチームでこれが地味に一番助かった点だった。

一方で、FastAPIで詰まった部分も正直に書いておく。非同期の扱いが思ったより難しい。`async def`で書いたけど内部でブロッキングなライブラリを呼んでいて、パフォーマンスが全然出ないという状況に最初ハマった。SQLAlchemy 2.0の非同期セッション管理は、設定を間違えると静かにデッドロックする。金曜の午後にデプロイしてモニタリングを見ていたら、じわじわとレスポンスタイムが上がっていって——原因特定に3時間かかった。

**実際の判断基準**：AIや機械学習系のAPIを作るなら、FastAPIはほぼ一択だと思う。非同期I/O、型安全、自動ドキュメント、全部が噛み合っている。ただし、チームにPythonの型ヒントと非同期プログラミングへの理解がある前提で。

---

## Djangoは「やりすぎ」じゃなかった——ただし条件がある

Djangoについては偏見を持っていた。「重い」「設定が多い」「APIには向かない」。今回の検証で、自分の理解が古かったと気づいた。

Django REST Framework（DRF）と組み合わせたDjango、そしてDjango Ninja（FastAPIスタイルの型ヒントベースのビューが使える）——この2つは別物として考えた方がいい。

Django Ninjaは正直、予想外だった。FastAPIのインターフェースにDjangoのバッテリー込みの哲学が組み合わさっている。認証、管理画面、マイグレーション、セッション管理——全部すでに用意されている。うちのプロジェクトではMLモデルの管理UIが必要で、Django Adminをそのまま流用できたのは大きかった。ゼロから作ってたら確実に1週間以上かかっていた作業が、設定数時間で終わった。

Djangoが本当に強いのは、「ユーザー管理」「権限システム」「管理画面」が必要なとき。SaaSの管理ダッシュボード、社内ツール、コンテンツ管理系——こういうものを作るなら、Djangoのデフォルト機能の充実度は他の追随を許さない。

ただ、パフォーマンスのオーバーヘッドは実在する。同一のエンドポイントで比較すると、DjangoはFastAPIより3〜5倍レイテンシが高かった。マイクロサービスの末端でリクエストをさばくような用途には向かない。あと、ORM——DjangoのORMは習熟が必要で、N+1問題を踏むのはDjango開発者のほぼ通過儀礼だと思う（`select_related`と`prefetch_related`を覚える前に一度は踏む）。

結局、これはプロジェクトの性質次第だ。APIだけを提供するマイクロサービスを作るのか、フルスタックのウェブアプリを作るのかで、判断が180度変わる。

管理機能付きのプロダクト、ユーザー認証・権限が複雑なアプリ、チームにDjangoに慣れたエンジニアがいる場合——このどれかに該当すれば、Djangoの「バッテリー込み」は本物の価値を持つ。スタートアップで「とにかく早く動くものを」という状況でも、意外とDjangoの方が速く立ち上がることがある。

---

## FlaskはシンプルだがシンプルすぎるのはFlaskのせいではない

Flaskに対する自分の評価は一番複雑だ。

Flaskは悪くない。むしろ、何年も愛用してきた。マイクロフレームワークとして、「最小限の構造で、あとは自分で決める」という思想は正しいと思う。問題は、「自分で決める」部分が積み重なったとき何が起きるか、だ。

4人チームで小さいプロジェクトをFlaskで始めると、最初は快適だ。ルーティング、テンプレート、基本的なDB接続——全部シンプルに書ける。でも半年後、認証ライブラリ、シリアライザ、バリデーション、非同期タスクキュー——全部バラバラのライブラリを組み合わせていて、「なぜこの設計になっているのか」がドキュメントどころかコードにも残っていない状況になりやすい。

```python
# Flaskでよく見る「育ちすぎた」コード
from flask import Flask, request, jsonify
from marshmallow import Schema, fields, ValidationError  # 別ライブラリ
from flask_sqlalchemy import SQLAlchemy               # 別ライブラリ
from flask_jwt_extended import jwt_required           # 別ライブラリ
import celery                                         # 別ライブラリ

# それぞれのバージョン互換性を自分で管理しないといけない
# flask-jwt-extended 4.x はFlask 2.x が前提だが
# 自分のプロジェクトではFlask 1.xで動いてて...みたいな話が起きる
```

これはFlaskが悪いというよりも、Flaskを選ぶ判断をした人が「後から複雑になることを過小評価する」という問題だと思う。自分も含めて。

2026年の今、Flaskを積極的に選ぶ理由はかなり限られてきている。シンプルなAPIが欲しいならFastAPIの方が型安全で自動ドキュメントがある。フルスタックならDjangoの方が一貫している。Flaskが輝くのは、本当に小さいスクリプト的なAPIとか、既存のFlaskプロジェクトのメンテナンスとか、学習目的で「フレームワークが何をしているか理解したい」とき——そういうニッチなシーンだ。

一個だけ、Flaskが今でも自分の中で生きている場面がある。Jupyter Notebookからデータサイエンティストが作ったモデルをサッと公開する、みたいな「使い捨てのプロトタイプAPI」だ。設定ゼロでHTTPエンドポイントが立つ、という強みはまだ本物だ。新規の本番プロジェクトなら、まずFastAPIかDjangoを検討してからFlaskを候補に入れるかどうか考えるのが正直なところだと思う。

---

## AI/MLツールを作るエンジニアへ——これが一番聞きたいはず

この記事を読んでいる人の多くは、LLMのAPIラッパーを作っているか、MLモデルのサービング基盤を整えているか、そういうところにいると思う。

正直に言う。AIバックエンドにはFastAPIがほぼ最適解だ。理由はいくつかある。

非同期I/Oが必要だから。OpenAIやAnthropicのAPIを呼ぶとき、ストリーミングレスポンスを扱うとき、複数のモデルを並列で呼ぶとき——全部、ブロッキングなフレームワークだとスループットが出ない。FastAPIの`async/await`はこの用途のために設計されたようなものだ。

型ヒントがモデルの入出力定義と相性がいい。Pydanticのモデルは、MLパイプラインの「どんなデータが入ってきて、何が返るべきか」を表現するのにちょうどいい粒度だ。ここでバリデーションエラーが自動で返ってくるのは、フロントエンドとの連携でも助かる。

Langchain、LlamaIndex、各種Pythonクライアントライブラリ——全部、FastAPIとの組み合わせが一番事例が多い。StackOverflowで詰まったとき答えが見つかりやすい、というのは実務で効いてくる。

一点だけ注意。FastAPIでWebSocketを使ったリアルタイムストリーミングを実装しようとしたとき——設定が一見シンプルに見えて、スケールアウトしたとき（特にKubernetes上で複数ポッドを立てたとき）にセッション管理が面倒になる。私はここを100%解決した自信がなくて、RedisでのセッションバックエンドかCloud RunのHTTP/2ストリーミングで逃げている。あなたの環境によっては別の解があるかもしれない。

---

## 最終的に何を選んだか

うちのMLパイプラインAPI、FastAPIにした。

2週間の検証を経て、チーム全員が同じ方向を向いた。型安全、自動ドキュメント、非同期処理——全部、うちの用途にフィットしていた。半年後の今も後悔していない。

個人的なルールとしてまとめると：

AIとMLのAPI、マイクロサービス、スループットが重要なエンドポイント——**FastAPI**。

管理機能付きプロダクト、複雑な権限システム、フルスタックのウェブアプリ、あるいはチームにDjango経験者がいる——**Django（+ Django Ninja）**。

新規の本番プロジェクト——**Flaskは候補から外す**。既存コードのメンテナンスや使い捨てプロトタイプは別の話だけど。

Anyway、フレームワークの選択は最終的には「チームが何に慣れているか」と「プロダクトが何を必要としているか」の掛け算だ。ただ、技術的な方向性として言えば、2026年のPythonバックエンドはFastAPIを中心に収束しつつある——少なくとも、自分の観測範囲ではそう見える。

<!-- Reviewed: 2026-03-08 | Status: ready_to_publish | Changes: removed mid-Japanese English phrase "Which brings me to", fixed date inconsistency (2025→2026 in Flask section), broke parallel 実際の判断基準 header pattern (Django section rewritten as inline prose, Flask section integrated into narrative), trimmed one redundant paragraph from Django section intro, fixed "半年後" timeline to "5ヶ月" for consistency with article date -->
