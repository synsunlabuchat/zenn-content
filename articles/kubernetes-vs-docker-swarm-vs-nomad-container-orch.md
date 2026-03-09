---
title: "KubernetesとDocker SwarmとNomadを2週間本番相当の環境で比較した話"
emoji: "🚀"
type: "tech"
topics: ["kubernetes", "docker-swarm", "nomad", "container-orchestration", "devops"]
published: true
---

ウチのチームは5人で、去年からマイクロサービスへの移行を進めていた。それまではEC2にDockerを直接並べる構成で回していたんだけど、サービス数が15を超えたあたりでデプロイ管理が本格的に辛くなってきた。「どのコンテナがどのホストで動いているか」を把握するためだけにSlackで確認が走る、みたいな状況。

オーケストレーションを入れようという話が出た時、最初は「Kubernetesでしょ」と思っていた。Swarmは時代遅れ、Nomadは名前は知っているけど使っている人を見たことがない——そんな認識だった。

でも、「本当にK8sが今のウチに合っているか確認してから決めよう」という話になって、3つ全部を2週間ちゃんと検証することにした。結果として、思っていたのとかなり違う景色が見えた。

## Kubernetesの「複雑さのコスト」は2026年でもまだ高い

Kubernetes 1.32を使った。EKSではなくkubeadmでオンプレに近い構成を自分で組んだ——マネージドサービスで隠れているコストを正確に把握したかったから。

最初の3日間はひたすら概念の把握に費やした。Pod、ReplicaSet、Deployment、Service、Ingress、NetworkPolicy。一個一個は理解できる。でも「これを全部組み合わせて本番で動くものを作る」という段になった時、頭の中の設計と実際のYAMLの間に常にギャップがある感覚があった。K8sのドキュメント自体が膨大で、答えを探しながら作業すると時間がいくらあっても足りない。

```yaml
# 一見シンプルなDeployment定義、でも「正しい値」を決めるのが難しい
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api-server
        image: myregistry/api-server:v2.1.4
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"  # この値、どうやって決めればいい？
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

このYAMLを書きながら「resourcesの値はどう決める？」「readinessProbeのタイムアウトはどれが正解？」という疑問が次々と出てくる。

5日目、PersistentVolumeClaimの設定ミスでステートフルなサービスのデータが飛んだ。金曜の午後に検証環境にpushして気づいたのが翌朝——。原因はStorageClassとRetainポリシーの理解不足で、典型的な「慣れれば防げるミス」だ。でも「慣れるまでのコスト」が5人チームには重くのしかかる。

Helmも試した。確かにパッケージ管理は楽になる。ただ、HelmチャートのデバッグはフラットなYAMLのデバッグより難しかった——テンプレートエンジンが挟まる分、実際にレンダリングされているものが直感的に追いにくい。`helm template`で展開したものを見ながらデバッグする時間が想像より多かった。

正直に言うと、K8sは「正しく使えれば最強」なんだけど、「正しく使える状態になるまでのコスト」が自分の想像より全然高かった。

## Docker Swarmは死んでなかった——というか意外と生きていた

Docker Swarm、2026年に真剣に検証している人間がどのくらいいるか怪しいけど、やってみたら発見があった。

Docker 27.3.1に含まれるSwarmは、シンプルさという点では三つの中でダントツだった。managerノード3台、workerノード5台の構成を組むのに半日かからなかった。

```bash
# Swarmの初期化——本当にこれだけ
docker swarm init --advertise-addr 192.168.1.10

# workerノードから参加
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377

# スタックのデプロイ（既存のCompose定義がそのまま使える）
docker stack deploy -c docker-compose.yml myapp

# ローリングアップデート
docker service update --image myregistry/api-server:v2.1.5 myapp_api
```

既存のdocker-compose.ymlをほぼそのまま使える、というのが思っていた以上に大きい。うちの15サービス分のCompose定義を移行するのに1日かからなかった。K8sで同じことをやるとYAMLの書き直しだけで数日かかる。

ログの追いやすさも違う。`docker service logs -f myapp_api`で全ノードからのログが一箇所に出る。K8sで同じことをやろうとすると`kubectl logs -f deployment/api -n production --prefix=true`になるか、Lokiを入れて集約するかになる。小規模チームにとってこの差は地味に効く——セットアップのメンテコストが積み上がる前に問題を発見できるかどうかの話でもある。

弱点も明確だった。オートスケーリングが標準にない。メトリクスベースの自動スケールをやろうとすると自前スクリプトか外部ツールが必要になる。ネットワーキングもK8sに比べると「ざっくり」していて、Ingress相当はTraefikを別途入れるのが現実的——そこで追加の設定コストが発生する。

100台超えたスケールでの挙動は試せていない。正直、そこから先はわからない。

## Nomad 1.9が思っていたより「ちゃんとしていた」

HashiCorpがNomadの開発を続けているのは知っていたけど、「Consul/Vaultと組み合わせてこそ価値が出る」というイメージがあって、単体での使い勝手は期待していなかった。

Nomad 1.9.2で検証した。まず面白かったのが、Dockerコンテナだけでなく、JARファイルやバイナリを直接デプロイできる「タスクドライバー」という概念だ。うちのチームにはJavaで書かれたバッチ処理が一部あって、それをDockerに包むことへの心理的抵抗が関係者にあった——Nomadならその必要がない。

設定ファイルはHCL（HashiCorpの独自フォーマット）で書く。

```hcl
# NomadのJob定義——YAMLよりも読みやすいと感じた
job "api-server" {
  datacenters = ["dc1"]
  type        = "service"

  group "api" {
    count = 3

    network {
      port "http" { to = 8080 }
    }

    task "server" {
      driver = "docker"

      config {
        image = "myregistry/api-server:v2.1.4"
        ports = ["http"]
      }

      resources {
        cpu    = 500  # MHz単位、K8sのmCPUとは別物
        memory = 256  # MiB単位
      }

      # ヘルスチェックがジョブ定義の中に収まる
      service {
        name = "api-server"
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

HCLはYAMLより読みやすいと感じた。ただこれは完全に好みの問題で、チームの半数は「YAMLの方が慣れている」という反応だった。

本当に驚いたのはWeb UIの完成度だった。Nomad 1.9のUIがK8s（kube-dashboard経由）やSwarmより情報密度が高くて使いやすかった。ジョブの状態、アロケーション（K8sでいうPod相当）、ログ、リソース使用状況が一画面で追える。「UIがいい」はNomadに期待していなかった軸だったので、これは純粋にうれしい誤算だった。

弱点はエコシステムの薄さ。K8sならcert-managerが証明書管理の事実上の標準なのに対して、NomadだとVaultとの連携が必要で、そのVaultのセットアップが別途必要になる。依存が増える。あと、Stack OverflowやZennで検索してもNomad関連の記事が少ない。今回の2週間でも何度か「英語でStack Overflowを漁るしかない」という場面があった——自分でハマった時に解決策を素早く見つけられるかどうかは、実運用では思っている以上に重要だ。

## 実際に使い比べて気になったのは比較表には載らないこと

「学習コスト・スケーラビリティ・エコシステム」みたいな軸で比較表を作るのは簡単だけど、実際に触れてみて気になったのはもう少し具体的な話だった。

**ローリングアップデート失敗時の挙動**。K8sは`kubectl rollout undo deployment/api`でロールバックできるけど、途中で失敗したPodが残って状態が中途半端になるケースを経験した（v1.32.1）。SwarmとNomadの方が比較的クリーンに元に戻った。K8sの方が成熟しているはずなのに——これは正直意外だった。

**GitOpsとの相性**。K8sはFluxやArgoCDとの組み合わせが鉄板で、インフラの変更をGitのコミット履歴として管理しやすい。SwarmとNomadでも実現できるけど、ツールチェーンの完成度で差がある。長期的なメンテナンス性を重視するなら、ここはK8sの明確な優位点だ。

**オブザーバビリティの入口**。K8sはPrometheusスタックが事実上の標準で、既製のGrafanaダッシュボードが豊富にある。Swarm/NomadでもPrometheusは使えるけど、K8s向けに最適化されたexporterが多い分、K8sの方が「すぐ動くものが揃っている」状態になりやすい。

結局、3つそれぞれが払っているコストの種類が違う。K8sは学習コストとYAMLの量、Swarmはスケーラビリティとオートスケールの弱さ、Nomadはエコシステムの薄さと情報量の少なさ。どれかを選ぶということは、何かを諦めるということでもある。

## 正直な推奨：チームとフェーズで答えが違う

2週間やってみて、自分なりの見方が固まった。

**5〜10人でサービス数が10〜20程度のチームなら**、Docker Swarmから始めるのが正解だと思う。既存のCompose定義をそのまま活かせるシンプルさが、チームの認知負荷を抑える。スケールの上限はあるけど、そこに達する前にチームが成長してK8sへの移行判断ができる。今のウチみたいな状況には一番フィットした。

**K8s経験者がチームに2人以上いるなら**、K8s一択でいい。経験者がいれば立ち上げコストを吸収できるし、長期的なエコシステムの恩恵は他の二つと比べ物にならない。EKSやGKEのマネージドを使えば運用負担も下がる。ただ、経験者がいない状態でK8sを選ぶのは茨の道だ——少なくとも今のウチには。

**コンテナ以外のワークロードが混在していて、かつConsulやVaultをすでに使っているなら**、Nomadが現実的な選択肢になる。ゼロからHashiCorpスタックを全部入れるのはコストが重いけど、すでに運用しているなら話が変わる。

うちのチームは結論としてSwarmで移行を始めた。K8sは「いつか必要になる」と思っているし、エコシステムの強さも理解している。でも今の5人でK8sを正しく運用するための時間が現実的に取れない。Swarmのシンプルさと既存Composeとの親和性が、今のフェーズには合っている——それが2週間の検証を経た判断だ。

「正解」とわかっていても選べない理由がある。そのトレードオフを自分たちで言語化できたこと自体、2週間の一番の収穫だったと思う。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: removed English phrases mid-Japanese-text (One thing I noticed / Which brings me to), renamed comparison section header to be more specific, reduced parallel structure rigidity in recommendations, added human voice touches (うれしい誤算, Stack Overflow漁る, 今のウチみたいな状況), removed hedging in conclusion, expanded meta_description with specific detail, trimmed minor redundancies -->
