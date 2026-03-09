---
title: "PythonアプリのGitHub Actions完全セットアップ: 2週間試行錯誤して学んだこと"
emoji: "🚀"
type: "tech"
topics: ["github-actions", "python", "ci-cd", "devops", "testing"]
published: true
---

去年の秋、3人チームで開発しているFastAPIアプリのCIをCircleCIからGitHub Actionsに移行した。理由はシンプルで、CircleCIの月額が3人規模のチームには重くなってきたこと、あとコードもPRもGitHub上で管理しているのに、CIだけ別サービスを使う意味を感じなくなってきたから。

移行に2週間かかった。「週末でサクッと終わる」と思っていたのに、キャッシュとsecrets周りで想定外の問題に立て続けにハマった。この記事はその記録でもあり、最終的に落ち着いた構成の解説でもある。

## CircleCIからの移行: 最初の30分でどこまで動くか

GitHub Actionsの最小構成は、正直びっくりするくらいシンプルだ。`.github/workflows/ci.yml`を作るだけで動き始める。

```yaml
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

      - name: Python 3.12のセットアップ
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: 依存関係のインストール
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: テスト実行
        run: pytest tests/ -v
```

これで動く。本当にこれだけ。CircleCIだと`config.yml`の構文を覚えながらOrbsを調べながら、という感じだったけど、GitHub Actionsは最初の30分でテストが通った。

ただし、これだとPython 3.12でしか動作確認できない。うちのアプリはAWS Lambdaにデプロイしていて、本番環境が必ずしも最新バージョンを追いかけているわけじゃない。なのでmatrix testingが必要になった。

## Matrix Testingの罠: `fail-fast`を忘れて無駄に時間を溶かした話

複数のPythonバージョンでテストするには`strategy.matrix`を使う。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
      fail-fast: false  # ← これを忘れると後悔する

    steps:
      - uses: actions/checkout@v4

      - name: Python ${{ matrix.python-version }}のセットアップ
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: 依存関係のインストール
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: lint (ruff)
        run: ruff check .

      - name: 型チェック (mypy)
        run: mypy app/ --ignore-missing-imports

      - name: テスト実行
        run: pytest tests/ -v --tb=short
```

`fail-fast: false`を設定しないと、Python 3.10のジョブが失敗した瞬間に3.11と3.12がキャンセルされる。最初これを忘れて、3.10の問題を直したと思ったら実は3.11でも別の問題があった、という状況になった。全バージョンの結果を一度に見たいなら必須の設定だ。

`continue-on-error`という設定もある。これをmatrixの特定itemに設定すると、そのバージョンが失敗してもワークフロー全体を成功扱いにできる。Python 3.13のbeta版でテストしたいけどCIを赤にしたくない、という場合に便利。ただ、私はあまり使っていない。失敗は失敗として見たい派なので。

そういえば、matrixを使うとジョブ数が増えるので、GitHub Actionsの無料枠(月2000分、プライベートリポジトリの場合)が思ったより速く減る。3バージョン×平均3分=9分/PRというのは地味に効いてくる。これはキャッシュで解決できる。

## pip-cacheのキー設計を間違えてビルドが遅くなった話

最初にキャッシュを設定したとき、こう書いた:

```yaml
- name: pipキャッシュ
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

これで動いた。最初は。

問題は`requirements.txt`と`requirements-dev.txt`を別管理していたこと。`hashFiles('**/requirements*.txt')`はどちらも含むので一見よさそうなんだけど、`requirements-dev.txt`だけ変わったときに`requirements.txt`のキャッシュも無効化される。逆も然り。

2週間後、pytestのバージョンを上げようとして`requirements-dev.txt`を1行変更したら、全matrixで依存関係の再インストールが走った。3バージョン分。そのとき初めて「あ、これは設計が間違っていた」と気づいた——しかも原因は単純で、`actions/setup-python`自体にキャッシュ機能があることを見落としていただけだった。

直した後の設定はこれだけ:

```yaml
- name: Python ${{ matrix.python-version }}のセットアップ
  uses: actions/setup-python@v5
  with:
    python-version: ${{ matrix.python-version }}
    cache: 'pip'                      # これだけでOK
    cache-dependency-path: |
      requirements.txt
      requirements-dev.txt
```

`actions/setup-python@v4`以降はこの`cache`オプションが使えて、内部でPythonバージョンとrequirementsファイルのハッシュを組み合わせたキーを自動生成してくれる。`actions/cache`を手動で書くより管理が楽で、しかもより賢い。

ビルド時間の変化: キャッシュなしで平均4分12秒 → キャッシュヒット時38秒。これは体感がかなり違う。

## `GITHUB_TOKEN`と自前Secretsの使い分け — ここが一番ハマった

正直これが一番時間を使った。GitHubには`GITHUB_TOKEN`というシークレットが自動で用意されていて、リポジトリへの書き込みやIssueへのコメントなどができる。これはリポジトリの設定から権限を調整できる。

一方、AWS LambdaにデプロイするにはAWSの認証情報が必要で、これは自分でリポジトリのSecretsに登録する必要がある。最初はこういう構成にしていた:

```yaml
- name: AWS Lambdaにデプロイ
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_DEFAULT_REGION: ap-northeast-1
  run: |
    aws lambda update-function-code \
      --function-name my-fastapi-app \
      --zip-file fileb://deployment.zip
```

問題はこのステップをPRのpushでも実行していたこと。PR作成のたびにLambdaが上書きされる。金曜の午後にこれに気づいて、かなり急いで修正した。

条件を追加すれば解決できる:

```yaml
- name: AWS Lambdaにデプロイ (mainへのpushのみ)
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_DEFAULT_REGION: ap-northeast-1
  run: |
    aws lambda update-function-code \
      --function-name my-fastapi-app \
      --zip-file fileb://deployment.zip
```

今はOIDC認証に移行していて、アクセスキーをSecretsに置いていない。OIDCを使うとGitHub ActionsがAWSに対してIAMロールを一時的に引き受ける形になるので、長期間有効な認証情報をどこにも保存しなくて済む。設定は少し複雑だけど、セキュリティの観点からは明らかにこちらが正しい。

`GITHUB_TOKEN`の権限についても一点。デフォルトでは広めの権限が付与されているリポジトリもある。ワークフローファイルの先頭で明示的に最小限の権限だけを宣言する習慣をつけた方がいい:

```yaml
permissions:
  contents: read  # コードの読み取りだけ
```

## 実際に本番で3ヶ月動いているworkflow.yml全文

2週間の試行錯誤を経て、今うちのリポジトリにあるのはこれだ:

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    name: テスト (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
      fail-fast: false

    steps:
      - uses: actions/checkout@v4

      - name: Python ${{ matrix.python-version }}のセットアップ
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
          cache-dependency-path: |
            requirements.txt
            requirements-dev.txt

      - name: 依存関係のインストール
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: lint (ruff)
        run: ruff check . --output-format=github  # PR上でファイルと行番号を直接表示

      - name: 型チェック (mypy)
        run: mypy app/ --ignore-missing-imports

      - name: テスト実行
        run: pytest tests/ -v --tb=short --cov=app --cov-report=xml

  deploy:
    name: デプロイ (mainのみ)
    needs: test  # testジョブが全部通ってから実行
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    permissions:
      contents: read
      id-token: write  # OIDC認証に必要

    steps:
      - uses: actions/checkout@v4

      - name: AWSの認証 (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: デプロイパッケージ作成
        run: |
          pip install -r requirements.txt -t ./package
          cp -r app/ ./package/
          cd package && zip -r ../deployment.zip .

      - name: Lambda更新
        run: |
          aws lambda update-function-code \
            --function-name my-fastapi-app \
            --zip-file fileb://deployment.zip
```

`ruff check`に`--output-format=github`を渡しているのは地味なポイントで、これをつけるとlintエラーがGitHub上のファイルビューに注釈として表示される。PR上でどのファイルの何行目が問題かが一目でわかって、レビューの手間が減った。

---

新しいPythonプロジェクトにGitHub Actionsを設定するなら、私はこの順番でやることをすすめる: まず最小構成(checkout + setup-python + pip install + pytest)で動かす。次にmatrix + キャッシュを追加する。最後にデプロイを別jobとして追加する。

一気に全部設定しようとすると、どこで詰まったかわからなくなる。私がそれをやって2週間かかった。段階的にやれば3日で終わると思う。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: expanded meta_description to ~158 chars; removed English filler "Right," replaced with natural Japanese transition; added em-dash aside in cache section; clarified free tier is for private repos; minor phrasing tightened throughout -->
