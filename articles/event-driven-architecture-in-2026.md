---
title: "2026年のイベント駆動アーキテクチャ: マイクロサービスがイベントストリーミングに移行する理由"
emoji: "🚀"
type: "tech"
topics: ["event-driven-architecture", "microservices", "kafka", "event-streaming", "distributed-systems"]
published: true
---

## なぜ今さらイベント駆動の話をするのか

正直に言う。3年前、僕はイベント駆動アーキテクチャについてのブログ記事を読むたびに「理論はわかるけど、うちのシステムには大げさすぎる」と思っていた。

それが間違いだったと気づいたのは、昨年の夏、本番環境のマイクロサービスが連鎖的にタイムアウトし始めたときだ。注文サービスが在庫サービスを呼び出し、在庫サービスが通知サービスを呼び出し、通知サービスがユーザーサービスを呼び出す。その連鎖のどこか一点が詰まると、全体が止まる。REST APIで同期的に繋いだシステムの脆さを、その日初めて体で理解した。

2026年現在、僕が関わるプロジェクトの大半はイベントストリーミングを中心に設計されている。KafkaやRedpandaを使うものもあれば、マネージドサービスに乗っかるものもある。移行の過程でかなりの失敗もした。この記事では、その失敗も含めて、なぜマイクロサービスがイベントストリーミングに向かっているのかを書いていく。

---

## 同期REST APIの限界は「遅さ」じゃない

よく誤解されているのだが、REST APIからイベント駆動に移行する理由は「パフォーマンスが悪いから」ではない。少なくとも、それだけじゃない。

本当の問題は**時間的結合（temporal coupling）**だ。サービスAがサービスBを呼び出すとき、AはBが今この瞬間に生きていることを前提としている。Bがデプロイ中でも、スケールアウト中でも、一時的にスローダウンしていても、Aは待つか失敗するかしかない。

これを解消しようとして、みんなリトライとサーキットブレーカーを実装する。僕もそうした。でも気づいたのは、リトライを増やすほどシステムの挙動が予測しにくくなるということだ。サービスが「たぶん成功した」「たぶん失敗した」という中間状態に入り込む。冪等性の実装を忘れると、注文が二重登録される。

もう一つの問題が**ファンアウト**だ。例えば「ユーザーが購入完了」というイベントが発生したとき、それに反応すべきサービスが10個あるとする。注文サービスの実装者が10個のサービスを全部知っていて、全部呼び出す責任を負うのか？　そんなのは明らかにおかしい。

イベントストリーミングはこの問題をきれいに逆転させる。「購入完了」というイベントをトピックにパブリッシュし、反応したいサービスが各自でサブスクライブする。注文サービスは他のサービスの存在を知らなくていい。新しいサービスを追加しても、既存のコードは変わらない。

---

## Kafkaに移行して最初にやった3つの失敗

実際に移行を始めてから、教科書には書いていないことをいくつか学んだ。恥を忍んで書く。

**失敗1: スキーマを軽く見た**

最初のプロジェクトでは、イベントのペイロードをJSONの生文字列でKafkaに流していた。「後でスキーマ決めよう」という典型的な先送りだ。3ヶ月後、コンシューマーが6個に増えた段階で、あるフィールドの名前を変えようとしたときに地獄を見た。どのコンシューマーが古いフィールド名に依存しているか、誰も把握していなかった。

今はAvroかProtobufを使い、Schema Registryを必ず立てる。最初は面倒に感じるが、これをやらないと後で絶対に後悔する。

**失敗2: コンシューマーグループの設計をミスった**

Kafkaはコンシューマーグループ単位でオフセットを管理する。これを理解せずに設計したせいで、開発環境と本番環境が同じトピックを読んでいる状況が発生した。開発環境のコンシューマーが本番のイベントを横取りして処理する、という恐ろしい状態だ。本番で不思議な挙動が起きて、調査に2日かかった。あのときの胃の痛さは忘れられない。

**失敗3: デッドレターキューを後回しにした**

コンシューマーが処理に失敗したとき、何が起きるか。デフォルトではリトライし続ける。毒入りメッセージ（poison pill）が一つあると、そのパーティション全体が詰まる。

デッドレターキュー（DLQ）が必要なのはわかっていたが「あとで実装しよう」と後回しにして、本番でメッセージが詰まってから慌てて実装した。最初から入れておくべきだった。これは本当に。

---

## 実際のイベントストリーミング設計: 注文処理の例

コードを見たほうが早い。注文が完了したときに、複数のサービスが並行して動く設計だ。

```python
# order_service/events.py
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID

@dataclass
class OrderCompletedEvent:
    event_id: UUID
    order_id: UUID
    user_id: UUID
    total_amount: float
    items: list[dict]
    occurred_at: datetime
    schema_version: str = "1.0"

# プロデューサー側
class OrderEventProducer:
    def __init__(self, kafka_producer, topic: str):
        self.producer = kafka_producer
        self.topic = topic

    def publish_order_completed(self, event: OrderCompletedEvent):
        self.producer.produce(
            topic=self.topic,
            key=str(event.order_id),  # order_idをキーにすることで同一注文の順序保証
            value=event.to_avro(),
            on_delivery=self._delivery_callback,
        )
        self.producer.flush()

    def _delivery_callback(self, err, msg):
        if err:
            # ここでアラートを飛ばす。サイレントに失敗させない
            logger.error(f"Message delivery failed: {err}")
            raise EventPublishError(f"Failed to publish event: {err}")
```

```python
# inventory_service/consumer.py
class InventoryEventConsumer:
    def __init__(self, kafka_consumer, dlq_producer):
        self.consumer = kafka_consumer
        self.dlq = dlq_producer
        self.MAX_RETRIES = 3

    def process_events(self):
        while True:
            msg = self.consumer.poll(timeout=1.0)
            if msg is None:
                continue

            try:
                event = OrderCompletedEvent.from_avro(msg.value())
                self._handle_order_completed(event)
                self.consumer.commit(msg)  # 処理成功後にコミット

            except RetryableError as e:
                # リトライ可能なエラーはリトライカウントを見て判断
                retry_count = self._get_retry_count(msg)
                if retry_count < self.MAX_RETRIES:
                    self._requeue_with_backoff(msg, retry_count)
                else:
                    self.dlq.send(msg, reason=str(e))
                    self.consumer.commit(msg)  # DLQに送ったらコミット

            except NonRetryableError as e:
                # 即座にDLQへ
                self.dlq.send(msg, reason=str(e))
                self.consumer.commit(msg)
```

ポイントは2点。プロデューサー側でメッセージのキーを`order_id`にしている。Kafkaは同じキーを持つメッセージを同じパーティションに送るので、同一注文に関するイベントの順序が保証される。

次に、コンシューマー側でエラーを「リトライ可能」と「リトライ不可能」に分類している。ネットワークエラーはリトライできるが、スキーマエラーはリトライしても無駄だ。この分類を最初から設計に組み込んでおくと、後の運用がずっと楽になる——実際にこれをやってから、夜中のアラートが明らかに減った。

---

## 2026年のエコシステム: 結局何を選べばいいのか

3年前と比べて、イベントストリーミングの選択肢は大幅に増えた。正直、選ぶのが難しくなっている。

**Apache Kafka** は依然として業界標準だ。成熟しており、ドキュメントも豊富で、困ったときにStack Overflowで答えが見つかる確率が高い。ただし、運用コストは低くない。KRaftへの移行が完了して安定したとはいえ、クラスタ管理は専門知識が要る。「Kafkaを本番で使っている」と言えると採用市場での信頼感が上がる、という副次効果もある（これは半分冗談、半分本気だ）。

**Redpanda** は最近のプロジェクトで使い始めた。Kafka互換APIを持ちながらC++で書かれており、JVMのオーバーヘッドがない。小〜中規模のチームには向いていると思う。個人的にはローカル開発のセットアップのシンプルさが気に入っている——Dockerで立ち上げてすぐ動く。ただ、エコシステムの成熟度はまだKafkaに及ばない。

**マネージドサービス**（Confluent Cloud、AWS MSK、Google Cloud Pub/Sub）は、インフラ管理のコストを払いたくないチームには現実的な選択肢だ。ただし、クラウドへのロックインとコストの上昇は意識しておく必要がある。月次のメッセージ数が増えると、想定外の請求が来ることがある。実際に一度やらかした。あの月の請求書を見たときの顔を、チームメンバーに見せたくなかった。

僕の現在の判断基準はシンプルだ。チームが5人以下で、インフラ専任エンジニアがいないなら迷わずマネージド。10人以上で、データ量がある程度予測できるならKafkaかRedpandaのセルフホスト。その中間は...ケースバイケースとしか言えない。

一つ付け加えておくと、2026年のトレンドとしてWASMベースのエッジでのイベント処理が注目を集めている。Cloudflare WorkersやFastlyのエッジで軽量なイベントフィルタリングをして、コアシステムへの負荷を減らすアーキテクチャだ。まだ実験的なフェーズだが、興味深い方向性だと思っている。

---

## 移行するときに最初にやるべきこと

「わかった、やってみよう」と思った人に向けて、実践的なアドバイスを書く。

**既存システムの全置き換えは絶対にやめろ。** 強調しておきたい。僕が見てきた失敗の大半は「全部まとめてイベント駆動に移行しよう」というアプローチから始まっている。システムが大きくなるほど、移行中の中間状態が複雑になり、どこかで詰まる。

代わりに「ストラングラーフィグパターン」を使う。既存のREST APIを残しながら、新しい機能はイベントドリブンで実装する。徐々に古い部分を置き換えていく。地味だが、これが一番確実だ。

**イベントの命名規則を最初に決める。** `UserUpdated`なのか`user.updated`なのか`UserAccountUpdated`なのか。後から変えると全コンシューマーに影響する。ドメインイベントの命名は過去形を使うのが一般的だ（`OrderCompleted`、`PaymentProcessed`）。これはコマンド（`ProcessPayment`）との区別を明確にするため。

**オブザーバビリティを最初から入れる。** 分散システムでは、何かおかしいときの原因特定が難しい。トレーシングIDをイベントに含め、各ステップでログを残す設計を最初からやっておく。OpenTelemetryをKafkaのプロデューサー・コンシューマーに組み込むのは今や定番だ。

---

最後に、個人的な意見を一つ。イベント駆動アーキテクチャは銀の弾丸じゃない。小さなCRUDアプリをKafkaで動かす必要はないし、チームがこの概念に慣れていなければ複雑さが増すだけだ。「本当にこれが必要か？」という問いを忘れないでほしい。

でも、複数のサービスが複雑に絡み合い、スケールが求められるシステムで、この設計を採用してから運用が楽になったのは事実だ。サービス間の依存が疎になり、個々のサービスを独立してデプロイできるようになった。障害が連鎖しなくなった。

あの夏の本番障害を経験した僕には、それが何より大きかった。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: meta_description expanded to ~65 chars for JP SEO, added personal asides in ecosystem section (Kafka副次効果, Redpandaローカル開発, 請求書エピソード), punched up failure-2 closing line, removed redundant section divider before conclusion, minor wording tightened throughout -->
