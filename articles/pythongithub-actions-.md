---
title: "Pythonアプリケーション向けGitHub Actions設定: 実務で学んだ完全ガイド"
emoji: "🚀"
type: "tech"
topics: ["github-actions", "python", "ci-cd", "devops", "automation"]
published: true
---

去年の11月、3人チームで運用していたFastAPIのサービスが金曜の夕方に壊れた。原因はPythonのバージョン非互換。ローカルは3.11、本番は3.10で、特定のf-string構文が3.10では動かなかった。「次から気をつけよう」で終わらせようとしたら、2週間後にまったく同じ理由で別のサービスが落ちた。そこでようやくCI/CDをちゃんとやろうと決めた。

それまではJenkins（前職から引き継いだやつ）をなんとなく使っていた。設定がXMLで、誰もメンテしたくないオーラを醸し出していた。GitHub Actionsに移行した主な理由は「設定ファイルをコードと同じリポジトリで管理したい」という一点。YAMLが好きというわけではないけど、Gitで差分が見えるのは確実に助かる。

この記事は「GitHub Actionsの概要」ではない。基本概念はわかっているという前提で、実際にPythonプロジェクトに組み込むときにハマるポイント — キャッシュ戦略、マトリックスビルド、環境別デプロイの分離 — を中心に書く。

## ワークフローファイルの構成: 最初の設計が後で効いてくる

`.github/workflows/`にYAMLを置くだけ、というのは誰でも知っている。問題はその中身の設計だ。私が最初にやったミスは、テスト・リント・デプロイをすべて`ci.yml`一枚に詰め込んだこと。最初は「シンプルでいい」と思っていたけど、本番デプロイの条件だけ変えたいときに全体を読み直す羽目になって、チームの誰かが「このファイル、ちょっと怖いんですけど」と言い出した。

今は3ファイルに分けている:

```
.github/
  workflows/
    ci.yml        # PRごとに実行: テスト、リント、型チェック
    deploy.yml    # mainマージ時: ステージング自動→本番手動承認
    scheduled.yml # cronジョブ: DB バックアップ、定期レポートなど
```

ファイルを分けると`workflow_call`での再利用もできるし、Actions タブでどのワークフローが失敗したか一目でわかる。地味だけど、チームが「CIの状態を確認する」習慣が明らかに増えた。

基本のCI設定から始める:

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
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
      fail-fast: false  # 1つ失敗しても他のバージョンのテストは続ける

    steps:
      - uses: actions/checkout@v4

      - name: Python ${{ matrix.python-version }} をセットアップ
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"

      - name: 依存関係をインストール
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: テスト実行
        run: pytest --tb=short -q

      - name: リントと型チェック
        run: |
          ruff check .
          mypy src/
```

`fail-fast: false`は最初に付け忘れた。デフォルトは`true`なので、Python 3.10でテストが失敗すると3.11と3.12は実行されないまま終わる。どのバージョンでどこが壊れているかを一度に把握したいとき — 特にライブラリを作っているなら — これは必須だと思う。

## Pythonキャッシュ: `cache: 'pip'`だけでは不十分な理由

正直に言う。最初の1ヶ月はキャッシュ設定を雑にやっていた。`actions/setup-python@v5`の`cache: 'pip'`だけ設定して「キャッシュしてるから大丈夫」と思っていた。ある日チームメンバーが「PR出すたびに3分以上かかるんですけど、これ普通ですか?」と聞いてきて、初めてちゃんと調べた。

`cache: 'pip'`はpipのグローバルダウンロードキャッシュを保存するだけで、インストール済みパッケージをキャッシュするわけではない。つまり`requirements.txt`が変わっていなくても、毎回パッケージを再インストールする。ネットワーク往復は減るが、インストール時間そのものはほぼ変わらない。

仮想環境ごとキャッシュするのがずっとよい:

```yaml
- name: 依存関係キャッシュ
  uses: actions/cache@v4
  id: cache-venv
  with:
    path: ~/.venv
    key: venv-${{ runner.os }}-${{ matrix.python-version }}-${{ hashFiles('requirements*.txt') }}
    restore-keys: |
      venv-${{ runner.os }}-${{ matrix.python-version }}-

- name: 仮想環境を作成して依存関係をインストール
  if: steps.cache-venv.outputs.cache-hit != 'true'
  run: |
    python -m venv ~/.venv
    ~/.venv/bin/pip install --upgrade pip
    ~/.venv/bin/pip install -r requirements.txt
    ~/.venv/bin/pip install -r requirements-dev.txt

- name: 仮想環境をPATHに追加
  run: echo "$HOME/.venv/bin" >> $GITHUB_PATH
```

`hashFiles('requirements*.txt')`をキャッシュキーに含めることで、`requirements.txt`か`requirements-dev.txt`が変わったときだけキャッシュが無効化される。この変更後、平均実行時間が3分10秒から50秒くらいまで落ちた。うちのリポジトリはPRが1日に十数回走るので、Actionsの消費分数が体感できるくらい減った。

`restore-keys`の書き方も重要。完全一致するキャッシュがない場合、部分マッチで以前のキャッシュを使う。パッケージを1個追加した場合でも、既存の仮想環境を再利用してdiffだけインストールするので、完全な再インストールよりずっと速い。

PoetryやPDMを使っているなら、`pyproject.toml`とlockファイルをハッシュに含める。`hashFiles('pyproject.toml', 'poetry.lock')`のように。lockファイルを含めないと、依存関係のバージョンが変わってもキャッシュが更新されないリスクがある。

## テストカバレッジとPRへの自動コメント

テストが通っているだけでは不十分だと気づいたのは、カバレッジが1ヶ月でじわじわ下がっていくのを後から発見したときだった。コードは増えているのにテストが追いついていない、よくあるパターン。ローカルで`pytest --cov`を自分から実行する人は少ない。

PRにカバレッジレポートを自動でコメントとして追加すると、レビュー時に嫌でも目に入る:

```yaml
- name: カバレッジ付きでテスト実行
  run: |
    pytest tests/ \
      --cov=src \
      --cov-report=xml \
      --cov-report=term-missing \
      --cov-fail-under=75

- name: カバレッジをPRにコメント
  uses: MishaKav/pytest-coverage-comment@v1.3.2
  if: github.event_name == 'pull_request'
  with:
    pytest-xml-coverage-path: ./coverage.xml
    title: テストカバレッジ
    create-new-comment: false       # 既存コメントを更新（新規作成しない）
    report-only-changed-files: true # このPRで変更したファイルだけ表示
```

`report-only-changed-files: true`が地味に便利で、PR全体のカバレッジではなく「このPRで変更したコードのカバレッジ」を表示する。レビュアーが「このコード追加したけどテストは?」を一目で確認できる。

`--cov-fail-under`の数値について: うちは75%に設定しているけど、適切な値はプロジェクトによる。レガシーコードを抱えているプロジェクトでいきなり80%を設定すると誰もPRを出せなくなる。60%くらいから始めて、テストを書く文化が定着したら上げていく方が現実的だと思う。

あと、地味に重要なポイント: サードパーティのActionは`@main`で固定しないこと。`@v1.3.2`のように具体的なバージョンタグを使う。`@main`にしておくとアップストリームの変更で突然パイプラインが壊れる。実際に一度やられて、原因を特定するのに30分かかった。

## 環境別デプロイ: ステージング自動→本番手動承認

デプロイパイプラインで一番悩んだのが「どこまで自動化するか」だった。ステージングは全自動、本番は人間が確認してから — この構成がうちのチームには合っていた。GitHub Environmentsがまさにこの用途に向いている。

リポジトリのSettings → Environmentsで`staging`と`production`を作成し、`production`にだけRequired reviewersを設定する。production deployのジョブがトリガーされると、GitHub UIでチームメンバーが「Approve」を押すまで待機状態になる。

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Container Registryにログイン
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: メタデータを取得
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-

      - name: Dockerイメージをビルドしてプッシュ
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-image
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: ステージングにデプロイ
        env:
          IMAGE_TAG: ${{ needs.build-image.outputs.image-tag }}
        run: |
          # デプロイスクリプトはここに
          echo "Deploying $IMAGE_TAG to staging"

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # ここで手動承認待ち
    steps:
      - name: 本番にデプロイ
        env:
          IMAGE_TAG: ${{ needs.build-image.outputs.image-tag }}
        run: |
          echo "Deploying $IMAGE_TAG to production"
```

`cache-from: type=gha`と`cache-to: type=gha,mode=max`はGitHub ActionsのキャッシュをDockerのレイヤーキャッシュとして使う設定。依存関係が変わっていない場合、Dockerビルドが3分から40秒くらいに短縮された。

GitHub Container Registry（ghcr.io）は同じリポジトリのActionsから`GITHUB_TOKEN`で認証できるので、別途シークレットを設定しなくていい。Docker Hubを使う場合は`DOCKERHUB_USERNAME`と`DOCKERHUB_TOKEN`が必要になる。

各環境ごとに別のSecretsを持てるのも便利で、`staging`と`production`で同じキー名（`DATABASE_URL`とか）を別の値で管理できる。

## 実際にやらかしたミスと、そこから学んだこと

パイプラインのデバッグはローカルでほぼできない。`act`というツールを使えば部分的にはできるけど、GitHub本番環境との差異があるので、最終的にはコミットしてCIを回すしかない。これが地味につらくて、デバッグのコミットがGit履歴に残り続ける。

**タイムアウト設定を最初から入れる**

GitHub Actionsのデフォルトタイムアウトは6時間。あるとき統合テストがflaky状態になって、外部APIの応答待ちで止まったジョブが6時間回り続けた。翌朝Actionsの消費分数を見て気づいた。それ以来、すべてのジョブに`timeout-minutes`を設定している:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 20  # テストが20分を超えたら何かがおかしい
```

**`if`式でシークレットを使おうとした**

ジョブの実行条件にシークレットを使おうとしたことがある:

```yaml
# これは動かない
if: ${{ secrets.DEPLOY_TOKEN != '' }}
```

セキュリティ上の理由でシークレットは`if`式の中で評価されない。代わりに`vars`（シークレットではない変数）を使うか、ジョブの設計を見直す必要がある。これに気づかずに30分くらい「なぜ条件分岐が効かないんだ」と悩んだ。

**`GITHUB_TOKEN`の権限は最小化する**

デフォルトの`GITHUB_TOKEN`は思ったより広い権限を持っている。ワークフローファイルに明示的に書いておく:

```yaml
permissions:
  contents: read
  pull-requests: write  # PRコメントのために必要
  packages: write       # ghcr.ioプッシュのために必要
```

必要な権限だけを宣言することで、万が一のサプライチェーン攻撃のリスクも少し下がる。これは最初から習慣にした方がいい。

**サブモジュールの設定漏れ**

これは少し恥ずかしい話なんだけど — submoduleを含むプロジェクトで`actions/checkout@v4`を使ったとき、`submodules: recursive`を付け忘れた。ローカルでは正常に動くのに、CI上でのみ`ModuleNotFoundError`が出続けた。エラーメッセージがモジュールの問題を示しているので、全然関係ない方向で1時間以上調査していた。実際の修正は1行だった:

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

Right, so — こういうミスは設定した直後ではなく、しばらくしてから「あれ、これずっとおかしくなかった?」と気づくパターンが多い。ワークフローのログを定期的に見る習慣が大事だと思う。

## 結局、私が今やっている構成

3ヶ月ほどこの設定で運用して、落ち着いた構成をまとめる。

テストは3バージョン（3.10、3.11、3.12）のマトリックス実行。ただし、これはアプリケーションではなくライブラリを作っている場合の話。サービスなら本番環境に合わせた1バージョンだけでいい。私は最初、アプリケーションなのに3バージョン全部テストして、Actionsの消費分数を無駄にしていた。

カバレッジの下限は75%。理想を言えば80%まで上げたいけど、今のプロジェクトではそこまで達していない。焦って数字を上げるより、テストを書く文化を定着させる方が長期的に意味がある。

デプロイはステージング自動、本番手動承認の2段構え。ただ、これが10人超えのチームに通用するかというと自信がない — 人数が増えてデプロイ頻度が上がったときに手動承認がボトルネックになる可能性はある。今のうちのチームサイズ（3人）では、「誰かが確認した」という安心感の方がスピードより価値がある。

pytestのオプションは`pyproject.toml`に書いておくことを強くすすめる:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

こうするとワークフロー内は`pytest`だけで済む。ローカル実行も同じオプションになるので、「CIでは通ってローカルでは落ちる」という謎の現象が減る。

Jenkinsから移行して3ヶ月、戻りたいと思ったことは一度もない。設定をコードとして管理できること、PRレビューの流れに自然に組み込めること — この2点だけで十分に元が取れている。最初の設定が完璧じゃなくても問題ない。動かすことが先で、改善は後からできる。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: meta_description expanded with staging/prod detail; "One thing I noticed:" localized to natural Japanese; English mid-sentence "I am not 100% sure this scales beyond a 10-person team" replaced with Japanese-native phrasing -->
