---
title: "PythonアプリケーションへのGitHub Actionsの設定: 自分が2週間かけて学んだこと"
emoji: "🚀"
type: "tech"
topics: ["github-actions", "python", "ci-cd", "devops", "automation"]
published: true
---

去年の秋、小規模なSaaSプロジェクト（FastAPIバックエンド、PostgreSQL、チームは自分含めて3人）のCI/CDをCircleCIからGitHub Actionsへ移行することになった。理由は単純で、コストだ。CircleCIの請求書が月々じわじわ増えていて、ある時「GitHubにすでに払ってるのに、なぜCIまで別サービスに払うんだろう」と気づいた。

移行自体は2週間かかった。思ったより長かった—ドキュメントが少ないからじゃなく、情報が多すぎて何が自分のユースケースに合うのかを見極めるのに時間がかかったから。この記事はその経験から書いている。特にPythonアプリを対象に、実際に動いている設定を共有する。

---

## まず基本のワークフローから作る

GitHub Actionsのワークフローは `.github/workflows/` ディレクトリにYAMLファイルとして置く。ファイル名は何でもいい。自分は `ci.yml` にしている。シンプルに。

最初に作った最低限のワークフローはこんな感じだった:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/ -v
```

これで動く。ただ、このままだと毎回 `pip install` が走るので遅い。依存関係が多いプロジェクトだと特に。キャッシュを追加するだけで体感がかなり変わる。

---

## キャッシュとPythonバージョン行列で実用的にする

`setup-python` アクション（v5以降）にはキャッシュ機能が内蔵されている。`cache: 'pip'` を指定するだけでいい。地味だけどこれが一番効いた改善だった—パイプラインの実行時間が半分近くに縮んだ。

複数のPythonバージョンでテストすることも重要だ。ライブラリを作っているなら特に。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false  # 一つのバージョンが落ちても他は続ける
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"           # requirements.txtのハッシュでキャッシュ
          cache-dependency-path: "requirements*.txt"

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run linting (Ruff)
        run: ruff check .

      - name: Run type checking (mypy)
        run: mypy src/

      - name: Run tests with coverage
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            -v

      - name: Upload coverage report
        uses: codecov/codecov-action@v5
        if: matrix.python-version == '3.12'  # カバレッジは1回だけ上げる
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage.xml
```

`fail-fast: false` は地味に重要だと思っている。3.10で落ちたとき、3.12での結果も見たいことが多いから。デフォルトはtrueで、一つ落ちると全部止まる。

lintはRuffを使っている。flake8 + isort + blackの組み合わせを長年使っていたけど、Ruffに移行してから設定が圧倒的にシンプルになった。速度も速い。正直、もっと早く移行すればよかった。

---

## シークレットと環境変数のハマりどころ

ここが最初に詰まった部分だ。DBのURLやAPIキーをどう渡すか。

GitHub ActionsのSecretsは `Settings > Secrets and variables > Actions` から設定する。ワークフローからは `${{ secrets.MY_SECRET }}` で参照できる。ここは問題ない。

問題はテスト用のデータベースだ。PostgreSQLを使っているテストは、CIでもDBが必要になる。最初はDockerを自分で起動しようとしたけど、GitHub Actionsには `services` という仕組みがあってそっちの方がきれいに書ける:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        # ヘルスチェック: DBが起動するまで待つ (これを忘れると接続エラーになる)
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb

    steps:
      - uses: actions/checkout@v4
      # ... 以下は通常通り
```

`options` のヘルスチェック設定を忘れると「接続拒否」エラーで詰まる。自分は最初これを省いて30分くらい原因を探した。PostgreSQLコンテナ自体は起動しているのにアプリが繋がれない—DBプロセスの初期化がまだ終わっていないから。ヘルスチェックを入れると、DBが準備できてからジョブが進むようになる。

シークレットについてもう一つ: フォークからのPull Requestはデフォルトでシークレットにアクセスできない。オープンソースプロジェクトで外部コントリビューターのPRを受け付けるなら、この制約を念頭に置く必要がある。`pull_request_target` というイベントもあるが、セキュリティ上の落とし穴があるので慎重に使う必要がある（ここは正直100%自信を持って語れる領域じゃないので、公式ドキュメントを参照してほしい）。

---

## Dependabotとの組み合わせ: 自動依存関係更新

これはおまけ的な話だが、設定してから明らかに楽になった。

`.github/dependabot.yml` を置くだけで、依存関係の更新PRを自動で作ってくれる:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
    groups:
      # マイナー・パッチの更新はまとめてPRにする
      minor-and-patch:
        update-types:
          - "minor"
          - "patch"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"  # Actionsの更新は月1回で十分
```

`groups` の設定はDependabot v2から使えるようになった機能で、マイナー・パッチアップデートをまとめて1つのPRにしてくれる。これがないと毎週10〜20個のPRが来て、レビューが追いつかなくなる。正直、この機能が来る前のDependabotはちょっと鬱陶しかった（チームから「また更新PR来てる」と言われる係になるやつだ）。

GitHub ActionsのアクションもDependabotで管理できる点は意外と知られていない。`actions/checkout@v4` のような参照が古くなっても自動でPRが来る。

---

## デプロイワークフロー: mainへのマージ後に自動デプロイ

CIが通ったら自動でデプロイしたい。自分たちはAWS ECSを使っていて、Dockerイメージをビルドして ECR にプッシュし、ECSのサービスを更新するフローになっている。

ジョブ間の依存関係は `needs` で表現する。全部書くと長くなるので要点だけ:

```yaml
jobs:
  test:
    # ... テストジョブ

  build-and-deploy:
    needs: test          # testジョブが成功してから実行
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
          docker push $ECR_REGISTRY/myapp:$IMAGE_TAG

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment
```

`IMAGE_TAG: ${{ github.sha }}` でコミットハッシュをタグにしているのはおすすめ。`latest` タグだけだと「今デプロイされているのはどのコミットか」が追いにくい。

`if: github.ref == 'refs/heads/main' && github.event_name == 'push'` の条件も重要。これがないと、mainへのPRを作っただけでもデプロイが走ってしまう—実際一度やった。

---

## 実際に運用して気づいたこと

**ワークフローの実行時間は意識する**: GitHub Actionsは無料枠があるが（パブリックリポジトリは無制限、プライベートは月2000分）、チームが増えると枠を超えることがある。自分たちは `paths` フィルターを使って、ドキュメントの変更だけのPRではテストを走らせないようにした。

```yaml
on:
  push:
    paths-ignore:
      - "docs/**"
      - "*.md"
      - ".gitignore"
```

**コンカレンシーの制御**: 同じブランチへの連続プッシュで複数のワークフローが走ると無駄になる。`concurrency` で制御できる:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # 古い実行をキャンセルして最新だけ走らせる
```

これを入れてからCIの無駄な実行がかなり減った。

**アクションのバージョンはコミットハッシュで固定する選択肢もある**: セキュリティ意識が高い組織では `actions/checkout@v4` の代わりに `actions/checkout@<コミットハッシュ>` を使うことがある。タグは書き換えられる可能性があるが、コミットハッシュは不変だから。3人チームではそこまでやっていないが、エンタープライズ環境なら検討する価値がある。

**ジョブ間の出力値の受け渡しには注意**: 地味にハマったのがここだ。ジョブをまたいで値を渡すには、送り側のジョブで `outputs` を明示的に定義する必要がある。ステップの出力を `jobs.<job_id>.outputs` に含めないと、次のジョブから `${{ needs.<job_id>.outputs.xxx }}` で参照できない—「なんで空になるんだ」と30分悩んだことがある。

---

## 自分が実際に推奨する構成

移行から半年経った今の視点で言うと:

**小規模チーム・個人プロジェクト**なら、最初の基本ワークフロー + キャッシュで十分だ。Dependabotも初日から有効にしておく。複数Pythonバージョンのマトリックスは、ライブラリを公開するなら必要だが、内部アプリなら1バージョンで十分なことが多い。

**複数人が関わるプロジェクト**では、lintとtype checkをCIで強制するのが長期的に効いてくる。PRごとに「ここのインポート順が...」みたいなレビューコメントが消える。Ruffの導入はここで本当に楽になった—設定ファイルが1つにまとまって、CIの設定もシンプルになる。

**デプロイまで自動化するなら**、ステージング環境へのデプロイを先に自動化して、本番は手動トリガー（`workflow_dispatch`）にする段階的なアプローチが安心だと思う。全部自動化したい気持ちはわかるが、本番への自動デプロイは信頼性の高いテストスイートがないと正直怖い。

CircleCIからの移行を後悔していない理由はシンプルで、GitHubと同じ場所で全部管理できるのが思ったより快適だから。PRのステータスチェックやDependabotとの連携が自然に動く。CircleCIの時は「なぜかステータスが同期されない」みたいな問題が定期的にあった—その面倒がなくなったのは地味に大きい。

まず動くものを作って、そこから少しずつ改善していく。GitHub Actionsに限らず、CI/CDはそのアプローチが一番うまくいく。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: expanded meta_description, added missing ECR login step to deploy snippet, added job outputs gotcha, tightened em dash usage, added Dependabot personality aside, varied paragraph rhythm, tightened generic conclusion -->
