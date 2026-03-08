---
title: "ArgoCD で GitOps を始めた話：2週間で学んだことと、正直ちょっとハマった箇所"
emoji: "🚀"
type: "tech"
topics: ["kubernetes", "argocd", "gitops", "devops", "helm"]
published: true
---

金曜の午後4時に本番へのデプロイを手動で実行して、翌週月曜の朝にSlackで「ステージングと本番でAPIのレスポンスが違う」というメッセージが飛んできた。

確認してみたら、Helmのvaluesファイルをステージングにしかapplyしていなかった。本番は2週間前の設定のまま動いていた。チームは当時4人で、「誰かがデプロイしたはず」という暗黙の了解で運用していた。まずい。

これが私がArgoCDを本格的に調べ始めたきっかけだ。「GitOpsはなんとなく知っている、いつかやる」くらいの温度感だったけど、あの月曜の朝で決意が固まった。

## Flux か ArgoCD か — 1週間触って決めた理由

GitOpsツールの選択肢として真っ先に上がるのはFluxとArgoCDの2つだと思う。どちらかに決める前に、両方を1週間ほど触ってみた。

FluxはArgoCDより「Kubernetesネイティブ」な設計で、コントローラーが軽量だし、CRDの設計も洗練されている。特にFlux v2（現在のFluxCD）はOCIレジストリからHelmチャートを直接pullできたり、Kustomizeとの統合が深かったりと、機能面では引けを取らない。正直なところ、技術的な洗練度ではFluxのほうが上かもしれない、と思うこともあった。

でも最終的にArgoCDにした。理由はシンプルで — UIがある。

最初は「UIなんてオプション機能でしょ」と思っていた。でも4人チームでGitOpsを始めるとき、「今何がデプロイされているか」を全員がブラウザで確認できる状態は思った以上に価値があった。特に、Kubernetesにあまり慣れていないメンバーが「本番のPodのステータスを見たい」というとき、ターミナルを開かなくていい。

ArgoCDのリソースグラフは視覚的に分かりやすくて、Syncが失敗したときのデバッグエントリーポイントになる。「どのリソースが OutOfSync か」「エラーの詳細は何か」がブラウザで完結できる。Kubernetes操作に慣れていないメンバーを抱えているチームには特に刺さると思う。

あと、ArgoCDはv2.9あたりからApp of Appsパターンのサポートが安定してきて、マルチテナント構成も扱いやすくなった。私が使い始めたのはv2.10.3で、特に大きな不具合には当たっていない。

## インストールは5分、その後の設定で2時間溶けた

インストール自体は公式ドキュメント通りでほぼ詰まらない:

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.3/manifests/install.yaml
```

最初のパスワード取得もすぐできる:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

問題はここから先だった。

**やらかしポイントその1。** 上のコマンドでログインして「よし動いた」と思ってそのまま進めたんだけど、後から気づいたのは — `argocd-initial-admin-secret` はログイン後に削除することが推奨されている、ということ。ドキュメントに書いてある。でも初回は読み飛ばした。本番で使うなら、早めにSSO連携に移行するのが絶対に正解で、うちはGitHub OAuth を設定した。設定自体は `argocd-cm` ConfigMapに数行追加するだけで、15分もかからない。

**やらかしポイントその2 — Ingressの設定。** ArgoCDはデフォルトでgRPCとHTTPSを同じポートで扱うので、NginxのIngress Controllerと組み合わせるときに一手間いる。素のIngress設定を書いたら、WebUIはアクセスできるのにCLI（`argocd` コマンド）がgRPC接続エラーを吐き続けた。試行錯誤の末、`nginx.ingress.kubernetes.io/backend-protocol: "GRPC"` アノテーションを設定することで解決したが、環境によってはSSLパススルーの構成が必要になるケースもある。これで30分くらい溶けた。

もう一点、デフォルトのポーリング間隔は最大3分。つまり `git push` してから最大3分待たないとSyncが始まらない。「あれ、反映されてない？」と混乱したので、GitHubのWebhookを早めに設定することを強くすすめる。ArgoCDのWebhook用エンドポイントは `/api/webhook` で、GitHub側のSecret設定と合わせて30分あれば完了する。

## Git リポジトリの構成 — モノレポで始めた理由

「アプリリポジトリとインフラリポジトリを分けるべきか」という問題、ここが正直一番悩んだポイントかもしれない。

調べると「分けるべき」という意見が多い。セキュリティ的にも、アプリの変更とインフラの変更が混在しないほうがレビューしやすい。理屈は分かる。

ただ4人チーム・初導入という状況で、最初からリポジトリを分けると「あのデプロイ設定ってどのリポジトリだっけ？」という混乱が生まれやすい。なのでまずモノレポ構成にして、スケールしたら分ける、という判断をした:

```
infra/
├── apps/
│   ├── staging/
│   │   ├── api-server.yaml      # ArgoCD Application CR
│   │   └── worker.yaml
│   └── production/
│       ├── api-server.yaml
│       └── worker.yaml
└── manifests/
    ├── api-server/
    │   ├── Chart.yaml
    │   ├── values.yaml           # 共通デフォルト値
    │   ├── values-staging.yaml
    │   └── values-production.yaml
    └── worker/
        └── ...
```

ArgoCDのApplication定義はこんな感じになった:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-server-staging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # 削除時にKubernetesリソースも消す
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/infra
    targetRevision: main
    path: manifests/api-server
    helm:
      valueFiles:
        - values.yaml
        - values-staging.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true       # Git から消えたリソースを自動削除
      selfHeal: true    # 手動変更を検知してGitの状態に戻す
    syncOptions:
      - CreateNamespace=true
```

`finalizers` の行、最初は入れていなかった。Application CRを削除したとき、Kubernetes側のDeploymentやServiceが残り続けて手動でクリーンアップするはめになった。地味だけどこれは重要。

## selfHeal と HPA が競合した話（一番ハマった）

正直、ここが最大の落とし穴だった。

`selfHeal: true` を設定すると、誰かが手動で `kubectl edit` してもArgoCDが「Gitと違う」と判断して元に戻す。GitOpsの本質的な挙動で、意図通りだし、目的でもある。でも — HorizontalPodAutoscaler（HPA）を導入したとき、問題が起きた。

HPAは自動的に `spec.replicas` を変更する。ArgoCDはそれを「ドリフト」と見なしてSyncしようとする。結果、HPAがトラフィック増加を受けてreplicas=5にスケールアップした直後に、ArgoCDがGitの定義（replicas=2）に戻す、というカオスな状況になった。水曜の朝、トラフィックが増えている最中に起きてちょっと焦った。

解決策は `ignoreDifferences` を使うこと:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas   # HPA が管理するフィールドなので ArgoCD は無視する
```

ただしここで一つ罠がある。このjsonPointerを設定すると、Gitで `replicas` の値を変更しても反映されなくなる（無視されるから）。本当にreplicasのデフォルト値を変えたいときは、HPA側の `minReplicas` を変更するか、一時的にこの設定を外すか、という運用になる。少し不格好だけど、今のところこれで安定している。

`prune: true` も本当に慎重に扱う必要がある。Gitからファイルを削除すると、本番のKubernetesリソースも消える。ConfigMapを削除したとき、アプリが起動できなくなって — これを本番で踏んだのはひどかった。今はPRのレビューチェックリストに「このファイル/リソース、削除して本当に大丈夫か？」という項目を追加している。

あと、ArgoCDのSyncで失敗したとき、エラーメッセージが「resource already exists」だけで原因が分かりにくいことがある。これはたいていHelmのリリース名とArgoCD管理リソースが重複している場合で、既存のHelmリリースをArgoCDに移行するときに起きやすい。`argocd app sync --prune` で解決することが多いけど、怖くてなかなか実行できないやつ。

## 2週間使ってみての率直な評価

設定が安定してから2週間、デプロイ周りのSlackの議論がほぼなくなった。「ステージングに反映した？」「本番はいつ？」「誰がデプロイする？」というやり取りが消えて、PRをマージすればそれで終わり、という状態になった。これはかなり効いた。

体感として一番よかったのは、本番とステージングの設定ドリフトが完全になくなったこと。Gitが唯一の真実の源泉になるので、「なんか本番だけ挙動が違う」という調査に使う時間がゼロになった。あの月曜の朝の経験がなくなった、というのが何より大きい。

不満点も正直に書く。ArgoCDのUIはリソースグラフがきれいなんだけど、Applicationが増えるとブラウザが重くなる。50個以上のApplicationが入っている環境で試したとき、タブが明らかに重かった。私の今の環境は15個程度なので問題ないけど、大規模になると厳しいかもしれない。100個を超えたらArgoCDのサーバーリソースとUI設計をちゃんと考える必要があると思う — そこまでスケールした経験はまだないので正直なところ分からない。

FluxとArgoCDで迷っているなら、判断軸はシンプルだと思う。チームのKubernetes習熟度が混在している、または可視化を重視するならArgoCDから始めるのが無難。CLIで完結できてリソース効率を優先したいならFluxでも全然いい。どちらも成熟したツールで、「どちらが正解か」という問い自体に意味はない。

いきなりすべての環境にArgoCDを入れるより、まずステージング環境の1サービスだけを対象にセットアップして2週間運用してみることをすすめる。HPAとの競合とか、pruneの挙動とか、実際に使って初めて理解できることが多い。私もそうだった。

ArgoCDのSyncボタンを押して全環境のリソースが一斉に揃っていくのを眺める瞬間 — あれはちょっと気持ちいい。

<!-- Reviewed: 2026-03-08 | Status: ready_to_publish | Changes: removed English AI-tell phrases ("One thing I noticed:", "So、", "Right, so —"); softened nginx ingress annotation claim to reflect it as one solution rather than the definitive fix (fact-check found SSL passthrough is the other documented approach); removed redundant sentence about "Helmのvaluesファイルを更新したのに" (implied by context); made recommendation section more conversational, less listy; added bold to "やらかしポイント" callouts for scannability; tightened conclusion by ~80 words; technical facts verified correct except nginx annotation (softened) -->
