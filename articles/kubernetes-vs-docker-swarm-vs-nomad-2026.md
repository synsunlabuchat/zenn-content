---
title: "Kubernetes vs Docker Swarm vs Nomad: 2026年のコンテナオーケストレーション比較"
emoji: "🚀"
type: "tech"
topics: ["kubernetes", "docker-swarm", "nomad", "container-orchestration", "devops"]
published: true
---

去年の11月、チームのインフラを刷新しようとして、約2週間かけてこの3つを真剣に比較した。結果から言うと、最初に「当然Kubernetesでしょ」と思っていた自分の判断は、かなりズレていた。

うちのチームは3人。フロントエンド、バックエンド、インフラ（つまり自分）という構成で、月次アクティブユーザーが3万人程度のSaaSを運用している。コンテナ化自体はすでにしていたが、オーケストレーションはずっとECS + Fargateに依存していて、そこから自前管理に移行しようとしていた。理由はコストとベンダーロックインへの不満。

この比較を通じて学んだことを、できるだけ素直に書く。

## Kubernetesの現実 — 「業界標準」の重さ

Kubernetesはv1.32を使った。公式ドキュメントの充実度は圧倒的で、Stack Overflowで質問したら10分以内に回答が来る。これは本当に助かる。

ただ、セットアップに正直3日かかった。

最初にk3sで軽量版を試したが、本番運用を想定するとやっぱりkubeadmでフルのクラスタを組みたくなる。3ノードのクラスタを組んだら、今度はCNIプラグイン（Flannelを選択）の設定でハマった。PodのネットワークがNodeをまたいで疎通しなくて、原因がFirewallのポート設定だったんだけど、そのデバッグに半日費やした。ドキュメントに書いてあることをちゃんと読んでいれば防げた類の問題なので、反省しかない。

```yaml
# これが意外とつまずくポイント
# kubeadm init時にpodネットワークのCIDRを明示しないとFlannelが迷子になる
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"  # Flannel用。変えると後で詰む
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd  # containerd使うならsystemdに揃える
```

Kubernetesで好きなのはHPAとVPAの組み合わせ。先月、APIエンドポイントに予期しないスパイクが来たとき（原因はいまだ不明、たぶんクローラー）、HPAが自動でPodを6から18に増やして、レスポンスタイムは300ms以内に収まった。この動きを初めて見たときは素直に感動した。

一方で、リソース消費が重い。コントロールプレーンだけで最低2GBのRAMを食う。3人チームで月$200くらいのVPSを使っているような環境だと、これがじわじわ効いてくる。

あと、RBAC。最初は「認証・認可をちゃんとやれてる感」があって気持ちよかったんだけど、複数のnamespaceで権限管理し始めると、RoleとClusterRoleとBindingの関係で混乱してくる。今でも完全に把握できているかというと、正直微妙なところがある。

## Docker Swarmは2026年でも使えるのか

結論から言うと、「使えるが、積極的には選ばない」が正直な感想。

Docker Swarm（Engine v27.x）のセットアップは驚くほど速かった。`docker swarm init`して、ワーカーノードで`docker swarm join`するだけ。設定ファイルもDocker Composeとほぼ同じ書き方ができる。

```yaml
# docker-compose.yml から swarm用に変えるとき、
# 変更点はほとんど deploy セクションだけで済む
version: '3.8'
services:
  api:
    image: myapp/api:v2.1.3
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback  # これ忘れると更新失敗時に詰む
      resources:
        limits:
          memory: 512M
    networks:
      - app-net

networks:
  app-net:
    driver: overlay
```

ある金曜の午後に試しに本番相当の環境を作ってみたら、1時間で動くものができた。Kubernetesの3日と比べると、この差は無視できない。既存のCompose資産をほぼそのまま使い回せる点も、現実的な移行コストとして評価に値する。

ただ、機能の限界がある。

カスタムスケジューリング、細かいリソース管理、複雑なネットワークポリシー、サービスメッシュ連携 — このあたりに踏み込もうとするたびにDockerのissueページを掘ることになる。「3年前から要望あるけどクローズ」みたいなissueがいくつかあって、開発が止まりかけているシグナルだと思った。

Docker社がKubernetes連携（Docker Desktop + k8s）に力を入れていることは明らかで、Swarm自体の機能開発は鈍化している。「今使えるが、2年後も使えるか」という問いに対して、安心して「はい」と言えない。

小規模なサイドプロジェクトや、チームにkube経験者がいなくてとにかく動かしたい場面では今でも選択肢に入る。でもそれ以外だと、わざわざSwarmを選ぶ理由を見つけにくい。

## Nomadという第三の選択肢

これが一番驚いた。比較前は「HashiCorpのマイナーなやつ」くらいの認識だったので。

Nomad（v1.9.x）はJobというコンセプトでワークロードを管理する。コンテナだけじゃなく、バッチジョブ、Javaアプリ、Raw execも扱える。うちの場合はバックグラウンドのデータ処理ジョブとAPIサーバーを同じクラスタで管理できた。KubernetesではCronJobとDeploymentで別々のマニフェストを管理する必要があるが、Nomadは一つのJob定義で済む分だけシンプルだった。

```hcl
# Nomadのjobファイル。HCLで書くのが最初は戸惑う
# でも慣れると可読性がかなり高い
job "api-server" {
  datacenters = ["dc1"]
  type        = "service"

  group "web" {
    count = 3

    network {
      port "http" {
        to = 8080
      }
    }

    task "api" {
      driver = "docker"

      config {
        image = "myapp/api:v2.1.3"
        ports = ["http"]
      }

      resources {
        cpu    = 500  # MHz
        memory = 256  # MB
      }

      # Consulとのネイティブ連携 — これが思ったより強力だった
      service {
        name = "api"
        port = "http"
        check {
          type     = "http"
          path     = "/health"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

Consulとのネイティブ統合が思ったより強力で、サービスディスカバリーがほぼ設定なしで動く。KubernetesだとCoreDNSとService定義をきっちり書く必要があるところが、Nomad + Consulだとほぼ自動でやってくれる。

ただ、エコシステムが薄い。Prometheus連携のドキュメントが古かったり、Helmチャート相当のものが少なかったり、ベストプラクティスを探そうとするとKubernetesの10分の1くらいの情報しかない。

自分が一番困ったのはUIだった。管理UIは正直、k9sやLensと比べると見劣りする。複数タスクのログをまとめて見る方法がすぐ分からなくて、30分くらい彷徨った。いやもしかしたら普通に見る方法があったかもしれないが、少なくともすぐ見つからなかった。

エンタープライズ向けの有料機能があるのも気になるポイントだった。一部のスケジューリングポリシーがEnterprise版専用だという記述を見つけたとき、「将来的に機能が有料に移行していかないか」という不安は正直あった。HashiCorpがIBMに買収された後のロードマップは、今後注意して見ていく必要がある。

## 3つを並べてみたときに見えてくること

実際のスループットを系統的に計測したわけではないが、セットアップコスト、運用負荷、機能の充実度、コミュニティの厚みで整理するとこうなる。

Kubernetesは投資対効果が最も高い — ただし、スケールする前提で。コントロールプレーンに専用のリソースを割けるなら、エコシステムの厚みが全部の不満を上回る。モニタリング（Prometheus + Grafana）、ロギング（Loki）、GitOps（ArgoCD）— これ全部Helmで10分以内に入る。うちのチームが最終的にKubernetesを選んだのは、この「後付けで何でも足せる感」が決め手だった。

Docker Swarmは「今月中に動かしたい、チームにkube経験者がいない」という状況に限って有効。ただし、一回作ったSwarmのスタックを後でKubernetesに移すのは実質書き直しに近い。それを踏まえると、「どうせ後で移行するなら最初から」という発想も合理的だと思う。

Nomadはもっと評価されていい。特に、コンテナ以外のワークロードも混在する環境（バッチ処理、Javaレガシー、スクリプト実行など）では、Kubernetesより設計がシンプルだと感じた。HashiCorpスタック（Vault、Consul、Terraform）を既に使っているなら、相乗効果は本物。

## 正直な推奨と、自分が選んだもの

うちのチームはKubernetesにした。EKSじゃなく自前のkubeadmクラスタで、コントロールプレーンは3ノード、ワーカーは2ノードからスタート。先週ArgoCDのパイプラインが完成して、プルリクエストマージ→自動デプロイのフローが動いてから、デプロイへの心理的ハードルが確実に下がった。

3人チームにKubernetesはオーバースペックだという意見は分かる。実際に最初の1週間は設定に追われて、「Swarmにしておけばよかった」と思った瞬間が何度かあった。でも今は、その初期コストを払って正解だったと思っている。

Right, so — 選ぶ基準をシンプルにするなら：

チームに3人以上のエンジニアがいて、2年以上運用する予定なら **Kubernetes**。一人か二人で、とにかく早く動かしたいなら **Docker Swarm**（ただし後の移行コストを覚悟して）。HashiCorpスタックを使っていて、コンテナ以外のワークロードも混在するなら **Nomad**。

「状況による」と言いたいところだが、2026年の時点では特別な理由がない限りKubernetesが正直な第一選択だと思う。エコシステムの差が開きすぎていて、他を選ぶためには明確な理由が要る。

Anyway、もし同じ検討をしているなら、試す順序は Swarm → Nomad → Kubernetes の順がいいと思う。シンプルなものから入って、Kubernetesの複雑さが何を解決しているかを体感してから移ると、設定の意味が分かりやすい。自分は最初からKubernetesに飛び込んで、後からSwarmを触って「なるほど、これを抽象化するためにこの設定が要るのか」と理解した感じがあった。順序、大事だと思う。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: fixed "ネイティブI統合" typo, removed redundant hedge phrase in RBAC section, tightened Swarm summary paragraph, minor voice tweaks throughout -->
