---
title: "2026年のイベント駆動アーキテクチャ: マイクロサービスがイベントストリーミングに移行する理由"
emoji: "🚀"
type: "tech"
topics: ["event-driven-architecture", "microservices", "kafka", "redpanda", "distributed-systems"]
published: true
---

うちのチームは2024年末まで、8つのマイクロサービスをRESTで繋いでいた。5人チーム、1日あたり約30万リクエスト。規模としては「まあ中小規模」という感じで、当時はこれで十分だと思っていた。

思っていた、が正しかった。しばらくは。

## 金曜の午後4時、4段のREST呼び出しチェーンが94秒で崩壊した

金曜の午後4時にデプロイしたのが失敗の始まりだった。このパターン、心当たりのある人は多いと思う。

注文サービスが在庫サービスを呼び出し、在庫サービスが通知サービスを呼び出し、通知サービスがメール配信サービスを呼び出す——という4段のREST呼び出しチェーンがあった。それぞれのサービスはHTTP/JSONで通信しており、特別なことは何もしていなかった。

メール配信サービスがタイムアウトし始めた。30秒後、通知サービスのスレッドプールが枯渇した。60秒後、在庫サービスが応答不能になった。90秒後、注文サービス全体が停止した。

エラーレートが0%から100%になるまで、正確に94秒。Circuit breakerは各サービスに実装していたが、「隣のサービスが回復するまで待つ」という設定になっていた。チェーンが深いと、circuit breakerが全部連鎖してオープン状態になる——これは当時まったく理解していなかった挙動だった。その夜3時間かけて手動でロールバックしながら、「circuit breakerって本当に意味あんのか」とSlackに書いた記憶がある。

教訓は明確だった。**同期的な依存チェーンは、依存の深さに比例して脆くなる。**

## 問題の本質は「プロトコル」ではなく「時間的結合」だった

最初にチームで出た解決案は「全部gRPCに移行しよう」だった。正直に言うと、私も最初はそれを支持していた。gRPCならタイムアウト制御が細かくなるし、バイナリプロトコルで速いし、双方向ストリーミングもできる。

でも実際に調べると、問題はHTTP対gRPCの話ではなかった。

注文サービスが在庫サービスの「今この瞬間の状態」を知る必要があるのか、と考え直してみた。よく考えると、そうではなかった。注文サービスが必要としていたのは「在庫が引き当てられたという事実」であって、それをリアルタイムで同期的に確認する必要はなかった。「注文が発生した」というイベントを発行し、在庫サービスはそのイベントを非同期で処理して「在庫引き当て完了」というイベントを発行する。通知サービスはそのイベントを拾う。

これが時間的結合（temporal coupling）の問題だ。同期RPCは呼び出し元と呼び出し先が「同時に生きていること」を前提にする。それがカスケード障害の根本原因だった。gRPCに乗り換えても、この前提は変わらない。

ここまで整理できて、初めてイベントブローカーの選定を本格的に始めた。

## Redpandaで非同期パイプラインを構築した: 実際のコードと踏んだ罠

ブローカーの候補はKafka、Redpanda、Apache Pulsarの3つだった。

KafkaはZooKeeper依存がv4.0で正式に除去されKRaftが安定しているが、うちのチームにKafka運用経験がなく、クラスタ管理の学習コストが怖かった。PulsarはKafkaと異なるアーキテクチャで、BookKeeperの概念から学ぶ必要がある。

Redpanda（v24.2）を選んだ決め手は2つ。Kafka互換のAPIを持つのでクライアントライブラリがそのまま使えること、そしてシングルバイナリで動いて運用がシンプルなこと。あと正直に言うと、ドキュメントが読みやすかった——これ、笑い話に聞こえるかもしれないが意外と馬鹿にできない判断基準だと今でも思っている。

Goで書いた基本的な構成はこんな感じだ:

```go
// order-service/events/producer.go
package events

import (
    "context"
    "encoding/json"

    "github.com/twmb/franz-go/pkg/kgo"
)

type OrderCreatedEvent struct {
    OrderID     string  `json:"order_id"`
    CustomerID  string  `json:"customer_id"`
    Items       []Item  `json:"items"`
    TotalAmount float64 `json:"total_amount"`
    OccurredAt  int64   `json:"occurred_at"` // Unix timestamp (ナノ秒)
}

func (p *EventProducer) PublishOrderCreated(ctx context.Context, ev OrderCreatedEvent) error {
    payload, err := json.Marshal(ev)
    if err != nil {
        return err
    }

    // OrderIDをパーティションキーにすることで、同一注文のイベント順序を保証する
    record := &kgo.Record{
        Topic: "order.created",
        Key:   []byte(ev.OrderID),
        Value: payload,
    }

    // ProduceSyncはレイテンシより確実性を優先する場合の選択
    // 高スループットが必要なら非同期Produceに切り替えること
    return p.client.ProduceSync(ctx, record).FirstErr()
}
```

```go
// inventory-service/events/consumer.go
package events

import (
    "context"
    "encoding/json"
    "log/slog"

    "github.com/twmb/franz-go/pkg/kgo"
)

func (c *EventConsumer) Run(ctx context.Context) {
    for {
        fetches := c.client.PollFetches(ctx)
        if ctx.Err() != nil {
            return
        }

        fetches.EachRecord(func(record *kgo.Record) {
            var ev OrderCreatedEvent
            if err := json.Unmarshal(record.Value, &ev); err != nil {
                // デシリアライズ失敗は即DLQへ — リトライしても無意味
                c.sendToDLQ(ctx, record, err)
                return
            }

            // reserveInventoryは冪等性を持たせること
            // 同じOrderIDで2回呼ばれても安全なように設計する
            if err := c.reserveInventory(ctx, ev); err != nil {
                slog.Error("inventory reservation failed",
                    "order_id", ev.OrderID,
                    "err", err,
                )
                // ここでのエラー処理戦略はビジネスロジックに依存する
                // うちはリトライトピックを使った指数バックオフを採用
            }
        })
    }
}
```

最初に踏んだ罠が、Dead Letter Queue（DLQ）のモニタリングを後回しにしたことだった。DLQに送る実装はしていたが、アラートを設定していなかった。本番でデシリアライズに失敗した数百件のメッセージが2週間、誰にも気づかれずに静かに積み上がっていた。

**DLQはアラートなしで運用してはいけない。これは本当に最初からやるべきだった。**

## イベントソーシングと「非同期イベント駆動」を混同して1ヶ月無駄にした

これが一番痛い失敗だった——というか、失敗と気づくまでに時間がかかったのが問題だった。

移行を始めて少し経った頃、チームメンバーの一人が「どうせやるならイベントソーシングにしよう」と言い出した。状態をイベントの累積として管理し、DBには「現在の状態」ではなくイベントログを永続化するパターンだ。「完全な変更履歴が手に入る」「いつでも任意の時点の状態を再構築できる」——たしかに魅力的に聞こえた。私も「それ、いいね」と乗ってしまった。

約1ヶ月後、状況はひどかった。`OrderAggregate`の設計に想定外の時間がかかり、CQRSのreadモデルとwriteモデルの整合性管理が複雑になり、全員がイベントソーシングのメンタルモデルを共有できているわけでもなかった。

一歩引いて考え直した。**うちに本当に必要だったのは、イベントソーシングではなかった。非同期のイベント駆動通信がほしかっただけだ。**

注文サービスに注文テーブルがあって、行が挿入されたらトピックにイベントを発行する——それで十分だった。全サービスのすべての状態変化を累積イベントとして持つ必要はなかった。この区別を最初にきちんと議論していれば1ヶ月は節約できた。

イベントソーシングは確かに強力なパターンだが、それ自体がかなり大きな投資だ。監査ログが法的要件になっているサービスや、複雑なドメインロジックを持つコアサービスには合うと思う。「非同期化したい」という動機だけで全面採用するのは、コスパが合わない。

## 移行から1年経った今、率直に言うと

後悔していない。でも過大評価もしていない。

カスケード障害は実質ゼロになった。在庫サービスが落ちても注文サービスは動き続け、イベントはブローカーに溜まって、復旧後に処理される。後から「分析サービス」を追加したとき、既存サービスのコードを一行も変えずに`order.created`トピックを購読するだけでよかった——これは想定以上に気持ちよかった。Redpandaの運用負荷も思ったより低く、3ノードクラスタをAWS上で動かして月次のメンテナンスがほぼアップグレード作業だけで済んでいる。

一方で、デバッグは確実に難しくなった。RESTなら「このAPIを叩いてこのレスポンスが返った」で追えるが、イベントは複数サービスをまたがるので、OpenTelemetryによる分散トレーシングなしでは何が起きているか全くわからない。最終整合性のメンタルモデルをチーム全員に浸透させるのにも時間がかかった。「注文したのに在庫がまだ減ってない（数ミリ秒後には解消する）」という状況を自然に受け入れられるようになるまで、慣れが必要だった。スキーマ進化の管理も地味に増えた——イベントのペイロード形式を変えるとき、後方互換性を真剣に考える必要がある。

**移行を勧めるかどうかの判断基準はシンプルだ**: マイクロサービスが5〜6以上あって、サービス間の同期呼び出しチェーンが2段以上ある場合、イベント駆動への移行を本気で検討する価値がある。1〜2サービスならREST + Sagaパターンで十分で、ブローカーを持ち込むコストに見合わない可能性が高い。

ブローカーの選択について、2026年現在でも私はまずRedpandaを試してほしいと言う。Kafka互換なので将来的にKafka managed serviceに移行する選択肢を残しつつ、運用のシンプルさを取れる。WarpStreamやConfluentのサーバーレスオファリングも面白いが、ベンダーロックインのリスクとコストをよく計算してほしい。

どのブローカーを選んでも変わらない話を最後に一つ。冪等性のある消費者の実装と、DLQのアラート設定は最初から入れること。私は実体験でそれを学んだ——DLQのアラートを後から追加するのは、思っているより面倒だった。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: fixed Kafka ZooKeeper version claim (v4.0 not v3.7), updated meta_description wording, added personal aside on Redpanda doc rationale, smoothed event sourcing section opener, tightened benefits/drawbacks flow, removed redundant sub-headers in final section -->
