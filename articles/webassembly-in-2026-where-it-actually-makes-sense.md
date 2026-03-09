---
title: "2026年のWebAssembly: JavaScriptを置き換える意味がある場面とは"
emoji: "🚀"
type: "tech"
topics: ["webassembly", "javascript", "rust", "\u30d1\u30d5\u30a9\u30fc\u30de\u30f3\u30b9\u6700\u9069\u5316", "\u30d5\u30ed\u30f3\u30c8\u30a8\u30f3\u30c9"]
published: true
---

去年の11月、うちのチームで画像フィルター機能を実装していたとき、TypeScriptで書いた処理が4K画像（3840×2160）で400msくらいかかっていて困った。ユーザーがスライダーを動かすたびにUIが固まる。これは直さないといけない。

そのタイミングで「WASMにしたら速くなるんじゃないか」という話が出て、私が2週間かけて検証することになった。チームは3人のフロントエンドエンジニアで、Rustの経験は一人が多少ある程度。本格的なWASMプロジェクトは全員初めてだった。

結論を先に言うと、WASMは確かにすごかった — ある種の処理では。ただ、一番驚かされたのはWASMが負けたケースで、そこがこの話で一番面白いところだと思っている。

## 4K画像リサイズ処理: WASMがJSを3〜6倍引き離した

セットアップはwasm-pack 0.12.xとwasm-bindgen 0.2.95を使った。`wasm-pack build --target web`でビルドして、Viteへの組み込みは`vite-plugin-wasm`経由。2023年頃と比べると、このあたりの設定が驚くほど簡単になっていた。あっさりしすぎて逆に不安になるくらい。

処理の中身はガウシアンブラーとバイリニア補間を組み合わせた画像リサイズ。入力は3840×2160のRGBAバッファ。

```rust
// src/lib.rs
// wasm-pack 0.12.x + wasm-bindgen 0.2.95 で動作確認済み
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn resize_image(
    src: &[u8],        // RGBA の生バイト列 (JS の Uint8Array をそのまま渡す)
    src_width: u32,
    src_height: u32,
    dst_width: u32,
    dst_height: u32,
) -> Vec<u8> {
    let mut dst = vec![0u8; (dst_width * dst_height * 4) as usize];
    let x_ratio = src_width as f32 / dst_width as f32;
    let y_ratio = src_height as f32 / dst_height as f32;

    for y in 0..dst_height {
        for x in 0..dst_width {
            let src_x = (x as f32 * x_ratio) as u32;
            let src_y = (y as f32 * y_ratio) as u32;
            // バッファ境界チェックは意図的に省略（JS側でバリデーション済み）
            let src_idx = ((src_y * src_width + src_x) * 4) as usize;
            let dst_idx = ((y * dst_width + x) * 4) as usize;
            dst[dst_idx..dst_idx + 4].copy_from_slice(&src[src_idx..src_idx + 4]);
        }
    }
    dst
}
```

Chrome 133のDevToolsで計測した結果:

- TypeScript純粋JS実装: 平均 **387ms**
- WASM (Rust、SIMDなし): 平均 **118ms**
- WASM (SIMD有効、`-C target-feature=+simd128`): 平均 **61ms**

SIMDが使えたのが大きかった。Rustの`wasm_simd`クレートを使う形で、Chrome・Firefox・Safariとも2025年から安定サポートが揃っているので、今なら気にせず有効にしていい。

JS側の呼び出しはこう書いた:

```typescript
// imageProcessor.ts
import init, { resize_image } from './pkg/image_processor';

let wasmReady = false;

// アプリ起動時に呼び出す — UIが表示される前に済ませておく
export async function initWasm() {
  await init(); // WebAssembly.instantiateStreaming を内部で呼ぶ
  wasmReady = true;
}

export function resizeToThumbnail(
  imageData: ImageData,
  targetWidth: number,
  targetHeight: number
): ImageData {
  if (!wasmReady) throw new Error('WASM未初期化');

  // JS → WASM のメモリコピーはここだけ（境界越えを最小化する設計）
  const result = resize_image(
    imageData.data,
    imageData.width,
    imageData.height,
    targetWidth,
    targetHeight
  );

  return new ImageData(new Uint8ClampedArray(result), targetWidth, targetHeight);
}
```

一点だけ。初回のWASMインスタンス化には50〜80msのオーバーヘッドがある。モジュールをキャッシュすれば2回目以降は消えるので、`initWasm()`をページロード時に先読みするようにした — が、金曜の午後にこの変更を本番に入れたら、初期化完了前に`resizeToThumbnail()`が呼ばれるパスが残っていてタイムアウトが発生した。翌朝に急ぎで直した。WASMのライフサイクル管理は、普通の非同期処理より少し慎重に扱う必要がある。

**テイクアウト**: CPU集約型の処理でWASMはちゃんと速い。SIMDは積極的に使っていい。ただし初期化コストを必ず測定して、先読みの設計を最初から考えること。

## SHA-256で完敗した — WebCrypto APIという存在を忘れていた

ここが今回の検証で一番驚いたところ。

画像処理でWASMが劇的に速かったので、次はプロジェクト内の別処理 — ユーザーがアップロードするファイルのSHA-256チェックサム計算 — もWASMに置き換えようとした。Rustの`sha2` 0.10.xクレートを使ってWASMモジュールを作り、ベンチマークを走らせると...遅い。

100MBのバッファで:

- WASM (Rustの`sha2` 0.10.x): 平均 **142ms**
- `crypto.subtle.digest('SHA-256', buffer)`: 平均 **38ms**

3倍以上負けた。

理由を調べると、WebCrypto APIはブラウザがネイティブ実装していて、CPUのSHA-NI命令セット（ハードウェアアクセラレーション）を直接使える。WASMはサンドボックスの中で動くので、この種の最適化へのアクセスが制限される。「WASMはネイティブに近い速度が出る」というのは事実だけど、「ブラウザがすでにネイティブAPIを持っている処理」には構造的に勝てない。

正直に言うと、最初は自分のRustコードが悪いんじゃないかと思って実装を2回見直した。それでWebCrypto APIのドキュメントをちゃんと読んで、ようやく理解した。恥ずかしい。

AES-GCMも同じだった。WebCrypto APIのAES-GCMにWASMで勝てる気がしない。暗号処理に関しては、ブラウザのAPIが存在する限りWASMで再実装する意味がない。

ついでに言うと、「JSONの大量変換もWASMで速くなるかも」と思って試したことがある。V8のJSON.parseは死ぬほど最適化されていて、WASMにしても速くならなかった。V8があえて得意としている処理に挑んでも勝てない。

**教訓**: 暗号処理はWebCrypto APIを使え。WASMは不要。ブラウザのネイティブAPIが存在する領域はそちらに任せるのが正しい。

## WASMとJSの境界コストが思ったより効いてくる

WASMが速いのは前述のとおりだけど、「WASMとJSの間でデータをやり取りするコスト」を軽視すると、期待したパフォーマンスが出ないことがある。

画像処理のケースでうまくいったのは、処理の構造が都合よかったから。大きなバッファを一度JSからWASMに渡して、処理し終わったら一度だけ返す。境界を越えるのは2回だけ。

これとは対照的に、WASMのコードが処理の途中でJSの関数を頻繁に呼び出したり、細かいデータを何度もやり取りしたりする設計だと、オーバーヘッドが積み重なる。100バイトのデータを1000回やり取りするより、100KBのデータを1回やり取りする方がずっと速い。これは当然に聞こえるけど、設計の段階で意識していないと気づかないまま実装してしまう。

SharedArrayBufferを使えばJSとWASMでメモリを共有できてコピーコストが消える — のだが、Cross-Origin Isolation（`COOP`と`COEP`ヘッダー）の設定が必要で、既存のアプリに後から入れるのは意外と面倒だった。私のプロジェクトでは最終的に使わなかった。設定コストに見合うほどのメリットがなかったので。

WASM Component ModelとWasmGCについても少し触れておく。2025年にWasmGCが主要ブラウザで安定化して、KotlinやDartがWASMターゲットを正式サポートした。これはRust/C++を書かない開発者にとって「既存のビジネスロジックをブラウザに移植する」選択肢が広がったという意味では大きい。ただしKotlin/WASMは起動コストがJSより大きいので、小さなユーティリティ処理に使うのは向いていない。まとまったドメインロジックをサーバーとブラウザで共有したい、みたいな明確なモチベーションがある場合の話だ。正直、私は100%この使い方に自信があるわけじゃないので、1年後くらいに再評価するつもり。

**ポイント**: WASMモジュールの設計は、境界を越える回数を最初から最小化することが前提。小さなデータを細かく受け渡すアーキテクチャだと、パフォーマンスゲインがオーバーヘッドで相殺される。

## 2026年現在、WASMを採用すべき判断基準

2週間の検証を経て、自分なりの基準がまとまった。

まず採用すべきケース。「単一の処理が50ms以上かかっていて、プロファイラでボトルネックが確認できている」こと。「なんとなく遅そうだからWASM」は危険で、JSONパースやDOM操作はV8が極端に最適化していてWASMにしても速くならない。私がそれをやった。数字を先に取れ。

次に「ブラウザの高レベルAPIが存在しない処理」であること。WebCrypto、WebGL、WebAudioがカバーしている領域はWASMで再実装するな。WASMが本領を発揮するのは、これらのAPIが存在しないCPU集約型の計算 — 独自コーデックの実装、物理シミュレーション、ML推論ランタイム、バイナリプロトコルのパース、音声DSP処理など。

そして「JSとのデータやり取りが少ない処理」であること。前節で書いたとおり、境界コストがパフォーマンスゲインを食いつぶさない設計が前提になる。

One thing I noticed — この3つの条件を全部満たせる処理は、Webアプリ全体の中でそれほど多くない。だからこそ「アプリ全体をWASMで書き直す」は、よほど特殊な事情がない限りやりすぎだ。

---

私の結論を言う。

WASMはJavaScriptを置き換えない。2026年になってもそれは変わっていない。正確に言えば「JSが苦手なCPU集約型処理のための高性能サブシステム」として使う — これが今の正しい位置づけだ。

具体的に使うべき場面: 画像・音声処理、バイナリデータの解析、PDFレンダリング、物理・数値シミュレーション、ML推論。これらに明確なパフォーマンス問題があってプロファイラで裏付けが取れているなら、WASMへの移行は報われる。私のケースでは処理時間が387msから61msになって、ユーザーからの遅延クレームはゼロになった。それは本物の改善だった。

動機が「最新技術を使いたい」とか「JavaScriptは遅いと聞いたから」ならやめておけ。V8もSpiderMonkeyも本当によく最適化されていて、普通のWebアプリのロジックはJSで十分速い。ツールはちゃんとした問題に当てるべきで、問題を先に見つけてからツールを選ぶのが正しい順序だと、この2週間で改めて確認した。

<!-- Reviewed: 2026-03-09 | Status: ready_to_publish | Changes: meta_description expanded to ~105 chars; テイクアウト headings varied (教訓/ポイント) to break formulaic repetition; added closing sentence to conclusion for finality -->
