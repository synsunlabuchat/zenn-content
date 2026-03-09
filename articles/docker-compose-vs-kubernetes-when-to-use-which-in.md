---
title: "Docker ComposeとKubernetes、2026年にどっちを選ぶか正直に話す"
emoji: "🚀"
type: "tech"
topics: ["docker", "kubernetes", "devops", "container-orchestration", "infrastructure"]
published: true
---

去年の9月、チームで「そろそろKubernetes移行するか」という話が出た。3人のエンジニアで動かしているB2B SaaSで、当時はDocker Compose + EC2で全然普通に動いていたんだが、競合他社がブログで「うちはGKEで動いてます」と書いていて、なんとなく焦った感じになったのがきっかけだ。完全にそれだけ。

2週間、本番相当の環境を両方で構築して検証した。チームの3人が全員「K8sよくわからん」という状態から始めたので、相当しんどかった。最終的に出した結論は「うちにはまだ要らない」だったんだが、その判断に至るプロセスが一番価値があったと思っている。その中で見えてきたトレードオフを、できるだけ具体的に書く。

## Docker Composeが「崩れ始める」のはどの瞬間か

正直、最初は「Docker Composeで何が悪いんだ」という立場だった。

うちのスタック：Next.jsフロント、FastAPIバックエンド、PostgreSQL、Redis、それから最近追加したLLM呼び出し用の非同期ワーカー（Celery）。全部Composeで動いていて、デプロイはGitHub ActionsからSSHでEC2に入って `docker compose up -d` するだけ。シンプル極まりない。

```yaml
# docker-compose.prod.yml（当時の実際のやつ、少し省略）
version: "3.9"
services:
  api:
    image: ${ECR_REGISTRY}/myapp-api:${IMAGE_TAG}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  worker:
    image: ${ECR_REGISTRY}/myapp-api:${IMAGE_TAG}
    command: celery -A app.worker worker --loglevel=info -c 4
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379
    restart: unless-stopped

  redis:
    image: redis:7.2-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  db:
    image: postgres:16
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
```

1年半、特に大きな問題はなかった。

崩れ始めたのはワーカーの負荷が急増したタイミングだった。LLM処理ジョブが増えて、コンカレントワーカーを増やしたい場面が出てきた。でもEC2は1台しかない。`--scale worker=2` はできるが、それは同じEC2の中での話で、別のインスタンスに分散させようとすると途端に複雑になる——Composeのネットワーク、ボリュームの共有、デプロイの順番、全部手動で面倒を見ることになる。

もう一個困ったのがゼロダウンタイムデプロイ。`docker compose up -d` は既存コンテナを止めてから新しいのを起動するので、数秒ダウンする。深夜なら許容できるけど、昼間に「ちょっと修正入れます」とSlackに書いてからデプロイするたびに影響が出る。SaaSでそれは地味にしんどい。

ただ、これを「K8sが必要なサイン」と即断するのは早い、というのが今の俺の見解だ。実はこの2つの問題、K8sなしでも解決できる。それは後で話す。

**このセクションの教訓：** 「Composeが限界」に見える問題の多くは、Compose固有の問題じゃなく「EC2 1台 + 手動デプロイ」の問題だ。切り分けてから判断した方がいい。

## Kubernetesが価値を出すのは「スケール」じゃなく「運用自動化の組み合わせ」

K8sを検討し始めて最初に期待していたのは、「スケールアウトが楽になる」ことだった。それは半分しか正しくなかった。

正確に言うと、K8sの強さはHPA（Horizontal Pod Autoscaler）によるオートスケールもそうだが、それ以上に「自己修復」「ローリングアップデート」「リソース制限」が組み合わさったときに出てくると思う。どれか1つだけ取ると「ECSでもできるじゃん」という話になる。

```yaml
# K8sのworker deployment（検証で作ったやつ）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: celery-worker
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # ここがゼロダウンタイムのキモ
  template:
    metadata:
      labels:
        app: celery-worker
    spec:
      containers:
      - name: worker
        image: myapp-api:latest
        command: ["celery", "-A", "app.worker", "worker", "--loglevel=info"]
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: celery-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: celery-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

これを見て「便利じゃないか」と思ったのは正直だ。CPU使用率が70%を超えたらワーカーが自動で増える。夜中に急にジョブが積まれても対応できる——はずだった。

ここで予想外のことが起きた。

HPAが反応するのは「CPUが上がってから」なので、Redisのジョブキューにタスクが積まれてからワーカーが増えるまで2〜4分かかる。うちのLLM処理ジョブは1件あたり30〜60秒かかるので、この遅延が思ったより痛かった。KEDA（Kubernetes Event-Driven Autoscaler）を使えばRedisのキュー長でスケールできるんだが、それはまた別のコンポーネントの設定が必要で——「K8sで楽になる」という話がどんどん複雑な方向に広がっていく感じ。バッチ処理系のJob APIは最近のK8sでだいぶ改善されているが、それでもKEDA等の追加ツールが必要な場面は依然として多い。

So、K8sが本当に光るのは：複数チームが複数サービスを独立してデプロイしている環境、SLAが厳しくて常時可用性が求められる場合、インフラエンジニアが専任でいる場合——この条件が揃ったときだと思う。1〜2個だと「その分のコストに見合うか」という話になる。

## 誰も正直に話さないKubernetesの「隠れたコスト」

K8sの技術的なメリットを書いた記事は山ほどあるが、コストについて具体的に書いてあるものは少ない。俺が実際に計算した数字を書く。

GKE（Google Kubernetes Engine）でn2-standard-4のノードを3台（最小構成）動かすと、月約$450。EKSだとコントロールプレーンだけで月$73、ノード込みで似たような金額になる。うちが当時払っていたEC2（m5.xlarge 1台 + RDS t3.medium）は月約$185。差額で月$265、年間$3,180。3人チームのスタートアップにとって、これは無視できない数字だ。

金額だけじゃなくて、K8sを「ちゃんと動かす」ために必要な周辺ツールの量が想像以上だった。Helm、cert-manager、ingress-nginx、Prometheus + Grafana、ArgoCD——これらは「K8sを使う」なら事実上必須に近い。全部セットアップするのに2週間かかったし、何かが壊れたときに原因を特定するのが本当に難しい。

実際にやらかした話をする。金曜の午後にingress-nginxのバージョンを上げたら（1.11.x → 1.12.x）、TLS terminationの挙動が変わっていて本番のHTTPSが突然503を返し始めた。気づいたのが夕方5時で、原因特定に2時間かかった。Composeだったら設定ファイルを1つ見れば終わる話が、K8sだとConfigMap、Secret、Ingress、Service、Deployment全部を追っていく必要がある。それ自体は仕方ないんだが、「何が変わったか」を把握するための認知負荷が、Composeの比じゃない。

一方で、Composeのコストにも隠れたものがある。EC2が1台だとSPOF（単一障害点）になる。可用性を上げるためにALB + Auto Scaling Groupを追加すると、それなりの複雑さは出てくる。Composeが「シンプル」でいられるのは、可用性の要件をある程度妥協しているからでもある——これは最初に考えていなかったトレードオフだった。

人月で考えると：K8sの学習と日常的な運用にエンジニア1人が月2〜3日取られるなら、その分の人件費は相当大きい。3人チームで誰かが2〜3日潰れるのは、スタートアップには本当にきつい。

## 3人チームで「Kubernetes移行しない」と決めた理由

2週間の検証が終わって、チームで話し合った。結論は「今じゃない」だった。

決め手は3つある。

まず、うちの一番の痛みは「ゼロダウンタイムデプロイができない」こと——これはComposeのままでも解決できる。EC2をALBの裏に2台置いて、Blue-Greenデプロイをシェルスクリプトで実装した。格好良くはないけど動く。コストは月$60増えるだけで、K8sに移行するより安い。

ワーカーのスケール問題については、SQS + ECS Fargate Spotを組み合わせると、K8sなしで解決できると気づいた。Celeryをそのまま動かしたいなら、ECS単体でもHPAに近いことはできる。2026年時点でECSがかなり成熟していて、「ComposeとK8sの中間」的な選択肢として真剣に検討する価値がある——3ヶ月前には俺の選択肢に入っていなかった視点だ。

最後が一番正直な理由で、チームの認知負荷の話だ。俺は個人プロジェクトでK8sを使っていて、正直好きだ。でもチーム全員がデバッグできる技術じゃないと、障害対応が俺一人に集中する。それは持続可能じゃない——どんなに技術的に優れていても。

Anyway、チームが10人を超えて、マイクロサービスが5個以上になったらK8sの話をまた真剣にするつもりだ。でも今じゃない。「いつか使うかもしれないから今覚えておく」という発想でK8sを本番に入れるのは、運用コストを自ら高くするだけだと思っている。100% sure じゃないけど、少なくともうちのフェーズではそうだ。

## 結論：チームの「最初の痛み」から逆算して選べ

曖昧な答えは出したくないので、はっきり書く。

**Docker Composeを選ぶべき状況：**
- エンジニアが1〜5人で、インフラ専任がいない
- サービスが1つのリポジトリ内に収まっている
- 月間アクティブユーザーが数万人規模以下
- 今の痛みが「スケール」より「デプロイの手間」や「開発速度」にある

**Kubernetesを選ぶべき状況：**
- 複数チームが独立してデプロイする必要がある（これが一番重要）
- サービスが5個以上で、それぞれ異なるスケール要件がある
- SREかインフラエンジニアが専任でいる、または採用予定
- PCI DSSやSOC2等のコンプライアンス要件でK8sエコシステムが前提になっている

「うちは成長する予定だからK8sにしとこう」という判断は、俺はお勧めしない。K8sの運用コストを払い始めると、それを捨てるのが難しくなる。その投資が回収できる規模になってから移行する方が、実際には速い。技術的負債のように聞こえるかもしれないが、現実的な話として、Composeからの移行は「辛い」けど「できない」ではない。

うちは今もComposeで動かし続けているが、後悔はない。むしろ2週間の検証で「なぜComposeで十分なのか」を言語化できたことの方が、チームにとって価値があった。次に誰かが「K8sにすべきでは」と言ったとき、感覚じゃなく根拠で答えられる。それで十分だ。

<!-- Reviewed: 2026-03-10 | Status: ready_to_publish | Changes: expanded meta_description, sharpened section 2 heading, removed version-specific K8s release claim, moved ingress-nginx story to clearer transition, varied section 4 paragraph openings to break parallel structure, tightened closing sentence -->
