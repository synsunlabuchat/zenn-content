---
title: "2026年ローカルLLMの動かし方: Ollama・LM Studio・Jan完全ガイド"
emoji: "🚀"
type: "tech"
topics: ["\u30ed\u30fc\u30ab\u30ebllm", "ollama", "lm-studio", "jan", "\u751f\u6210ai"]
published: true
---

# 2026年ローカルLLMの動かし方: Ollama・LM Studio・Jan完全ガイド

APIの課金が積み上がっていく請求書を見て、「自分のマシンでモデルを動かせないか」と考えたことはないだろうか。あるいは、社内の機密データをクラウドに送ることへの不安から、オフライン環境でAIを使いたいと思っていた人もいるかもしれない。

2026年現在、**ローカルLLM**を自分のハードウェアで動かすことは、もはや研究者の専売特許ではない。Ollama、LM Studio、Janといったツールの成熟により、ラップトップ1台から始められる環境が整っている。本記事では、それぞれのツールのセットアップ方法から実際の活用まで、具体的なコマンドと一緒に解説する。

---

## ローカルLLMを選ぶ理由

コスト、プライバシー、レイテンシ。この三つが、エンジニアがローカル実行に踏み切る主な動機だ。

**コスト面**: GPT-4o や Claude 3.5 Sonnet を大量に呼ぶと、月額数万円の請求が発生するケースは珍しくない。一方、ローカルLLMなら電気代のみで動かし続けられる。

**プライバシー面**: 個人情報や営業秘密を含むプロンプトを外部サーバーに送ることに、法務部門からNGが出る場面は増えている。ローカル実行なら通信が一切発生しない。

**レイテンシ面**: ネットワーク遅延がゼロになる。特にストリーミング出力を前提としたアプリケーションでは、体感速度が大きく改善する。

ただし、現実的なトレードオフも存在する。Llama 3.3 70B や Mistral Large といったモデルを快適に動かすには、高VRAM GPUが必要だ。MacBook Pro M3/M4 シリーズなら統合メモリの恩恵でCPU推論より高速に動くが、それでもGPT-4クラスのモデルと品質で同等とは言いにくい。用途と要件を見極めた上で導入するのが正解だ。

---

## Ollama: CLIファーストのシンプルな選択肢

Ollamaは、DockerライクなCLIでモデルの管理と実行を行うツールだ。シンプルさと拡張性のバランスが良く、CI/CDパイプラインへの組み込みや、APIサーバーとしての利用に向いている。

### インストール

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windowsはインストーラーをダウンロードして実行
# https://ollama.com/download
```

インストール後、デーモンが自動起動する。`ollama serve` で手動起動することもできる。

### モデルのダウンロードと実行

```bash
# Llama 3.2 (3B) を取得して対話
ollama run llama3.2

# Qwen2.5 Coder 7B を取得（コーディングタスク向け）
ollama run qwen2.5-coder:7b

# 日本語性能が高いモデルの例
ollama run elyza/elyza-japanese-llama-2-7b-instruct
```

初回実行時にモデルが自動でダウンロードされ、以降はローカルキャッシュから即起動する。

### REST APIとして使う

Ollamaはデフォルトで `http://localhost:11434` にOpenAI互換APIを提供する。

```bash
curl http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [
      {"role": "user", "content": "Pythonでフィボナッチ数列を生成する関数を書いて"}
    ],
    "stream": false
  }'
```

Python SDKからも同じエンドポイントが使える。

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",  # 任意の文字列でOK
)

response = client.chat.completions.create(
    model="llama3.2",
    messages=[
        {"role": "user", "content": "SQLのINDEXが効かないケースを3つ挙げて"}
    ]
)
print(response.choices[0].message.content)
```

### Modelfile: モデルのカスタマイズ

Dockerfileに相当する `Modelfile` を使えば、システムプロンプトや温度パラメータを固定したカスタムモデルを作れる。

```dockerfile
FROM llama3.2

SYSTEM """
あなたはシニアバックエンドエンジニアです。
回答は常に日本語で、コードにはコメントを付けてください。
"""

PARAMETER temperature 0.3
PARAMETER num_ctx 4096
```

```bash
ollama create my-engineer -f ./Modelfile
ollama run my-engineer
```

---

## LM Studio: GUIで直感的に使えるデスクトップアプリ

LM StudioはElectronベースのデスクトップアプリで、Hugging FaceからGGUFフォーマットのモデルを直接検索・ダウンロードし、チャットUIで試せる。エンジニアリング知識が浅いチームメンバーに渡す場合や、モデルを素早く試したい場面に便利だ。

### セットアップ

[lmstudio.ai](https://lmstudio.ai) から各OS向けのインストーラーをダウンロードしてインストールする。起動後、内蔵の検索UIでモデル名を入力するだけでダウンロードできる。

### モデル選択の指針

LM Studioの画面左上にある「Model Compatibility」インジケータが参考になる。使用しているMacのUnified Memory容量やGPUのVRAMに対して、モデルが快適に動作するかどうかを緑/黄/赤で示してくれる。

実用的な目安:
- **8GB RAM**: 3B〜7Bモデルのq4_K_M量子化版
- **16GB RAM**: 13Bモデルのq4_K_M、または7Bのq8_0
- **32GB RAM**: 30B〜34Bモデルのq4_K_M

### ローカルAPIサーバーとして使う

LM Studioにも「Local Server」モードがある。左側のサーバーアイコンをクリックし「Start Server」を押すと、`http://localhost:1234/v1` でOpenAI互換APIが立ち上がる。Ollamaと同様に、既存のOpenAI SDKコードの `base_url` を変えるだけで接続できる。

```python
import openai

openai.base_url = "http://localhost:1234/v1/"
openai.api_key = "lm-studio"

response = openai.chat.completions.create(
    model="mistral-7b-instruct",  # LM Studioで読み込んでいるモデル名
    messages=[{"role": "user", "content": "Dockerfileのベストプラクティスを教えて"}],
    temperature=0.7,
)
print(response.choices[0].message.content)
```

---

## Jan: プライバシー重視のオールインワンクライアント

JanはOllamaやLM Studioとは少し異なるポジションを持つ。完全オフライン動作にこだわったオープンソースプロジェクトで、テレメトリーが一切ない点を明示的に売りにしている。UIはチャット履歴の管理機能が充実しており、日常的なAI活用ツールとして使いやすい。

### インストールと初期設定

[jan.ai](https://jan.ai) からダウンロードしてインストールする。初回起動時にモデルのダウンロードを案内するウィザードが表示される。

Janの「Hub」タブからモデルをダウンロードするか、すでにOllamaでダウンロード済みのモデルをJanから参照する設定もできる。

### Janの独自機能: スレッド管理とアシスタント設定

Janの強みは、プロジェクト別・目的別にスレッドとアシスタント設定を分けて保存できる点だ。例えば「コードレビュー用」「技術調査用」「メール文案用」といった用途ごとにシステムプロンプトとモデルを組み合わせて保存しておける。

設定は `~/jan/assistants/` 以下にJSONで保存されているため、Gitで管理してチームに配布することも可能だ。

```json
{
  "id": "code-reviewer",
  "object": "assistant",
  "name": "コードレビュアー",
  "model": {
    "id": "qwen2.5-coder-7b",
    "settings": {
      "temperature": 0.2,
      "max_tokens": 2048
    }
  },
  "instructions": "あなたは経験豊富なシニアエンジニアとしてコードレビューを行います。セキュリティ、パフォーマンス、可読性の観点から具体的なフィードバックをください。"
}
```

---

## ツール比較: 用途別の選び方

三つのツールは競合というより、用途によって使い分けるものだ。

| 観点 | Ollama | LM Studio | Jan |
|---|---|---|---|
| 操作方法 | CLI / REST API | GUI | GUI |
| 対象ユーザー | エンジニア | 非エンジニア含む | エンジニア〜一般 |
| API互換性 | OpenAI互換 | OpenAI互換 | OpenAI互換 |
| チーム配布 | Modelfileで設定共有 | 設定のエクスポートが手動 | JSONで管理可能 |
| プライバシー重視度 | 高 | 中〜高 | 最高 |
| モデルソース | Ollama Hub | Hugging Face | Hugging Face / Ollama |

### 推奨ユースケース

**Ollamaを選ぶ場面:**
- Dockerコンテナや自動化スクリプトに組み込む
- チームの開発環境として `.env` と `Modelfile` で再現性を確保したい
- LangchainやLlamaIndexとの統合 (`ChatOllama` クラスが標準対応)

```python
# LangchainでのOllama利用例
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.2", temperature=0)
response = llm.invoke("Pythonのデコレータを簡潔に説明して")
print(response.content)
```

**LM Studioを選ぶ場面:**
- モデルを試して評価するプロトタイピングフェーズ
- GUIで操作したいチームメンバーに渡す
- Hugging Face上の最新モデルをすぐに試したい

**Janを選ぶ場面:**
- 機密性の高い業務でAIを日常的に使いたい
- 法務・コンプライアンス上、通信ログを完全にゼロにしたい
- チャット履歴を組織内で管理したい

---

## 2026年注目のモデルと用途別推薦

ローカルLLMの選択肢は2026年に入りさらに広がっている。

**汎用・日本語対応:**
- **Qwen2.5 72B (Q4_K_M)**: 中国発だが日本語性能が高く、72Bクラスのローカルモデルとして現実的な選択肢。32GB以上のVRAMまたはUnified Memoryが必要。
- **Llama 3.3 70B**: MetaのフラッグシップオープンモデルのInstruct版。英語中心だが多言語対応も実用レベル。

**コーディング特化:**
- **Qwen2.5 Coder 32B**: コード補完・生成に特化。多くのベンチマークでGPT-4oに迫る結果を出している。
- **DeepSeek Coder V2**: コスト効率の高いMoEアーキテクチャで、推論コストを抑えながら高精度を実現。

**軽量・高速 (8GB以下のVRAMでも動く):**
- **Phi-4 Mini**: Microsoftの小型モデル。3.8Bながら推論能力が高い。
- **Gemma 3 4B**: GoogleのGemmaシリーズ最新版。日常的なタスクには十分な品質。

量子化の選び方については、`q4_K_M` が品質とサイズのバランスが良く、まず試すべき標準的な選択肢だ。品質を優先するなら `q8_0`、速度・サイズを優先するなら `q2_K` を検討する。

---

## まとめと次のステップ

ローカルLLMは、クラウドAPIの「とりあえずOpenAI」という選択に対するもう一つの現実的な選択肢として定着しつつある。コスト、プライバシー、オフライン動作といった要件が一つでも当てはまるなら、試してみる価値は十分にある。

導入のハードルは思ったより低い。まずは以下の手順で始めてみてほしい。

1. `ollama run llama3.2` で最小構成を体験する
2. 自分のユースケースに合ったモデルを選ぶ
3. 既存のOpenAIクライアントコードの `base_url` をローカルエンドポイントに変えて動作確認する
4. Modelfile やアシスタント設定をGitで管理し、チームに展開する

「クラウドAPIで十分では」という判断も正しい。しかし、一度ローカルで動かしてみると、レイテンシの低さとプライバシーの安心感は体感として分かる。まず小さく試して、そこから判断するのが現実的なアプローチだ。

---

*モデルの量子化形式やOllamaのDockerイメージを使ったチーム展開については、続編記事で詳しく取り上げる予定です。*
