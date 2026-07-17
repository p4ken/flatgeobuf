# flatgeobuf 公式リポジトリへの PR 提出チェックリスト

対象: `feature/modify-column-order` ブランチ(Rust クレートのみの変更)を
[flatgeobuf/flatgeobuf](https://github.com/flatgeobuf/flatgeobuf) に PR として出すために必要なこと。
(2026-07-16 時点の調査結果)

## 現状の整理

- fork: `p4ken/flatgeobuf`(remote 名 `origin`)、upstream は remote 名 `flatgeobuf`
- upstream `master` はこのブランチの分岐点から 1 コミットだけ先行(`be2b132`、C++ のみの変更)→ **コンフリクトなし**
- ブランチ上のコミットは 3 つ:

| コミット | 内容 | 公開 API への影響 |
|---|---|---|
| `a65e469` Reorder columns order | `FgbWriter::sort_columns_by()` 追加。既存 feature を再エンコードせずに列順を変更 | `sort_columns_by` 追加(minor)、**`Error::IllegalColumnIndex` variant 追加(major、後述)** |
| `e64dde2` Reduce INFO logs | プロパティと列の照合を名前ベースにし、feature ごとの INFO ログを削減。doc コメント追加 | なし(内部変更) |
| `7a56691` Reduce peak memory in FgbWriter::write | R-tree index をリーフノードから直接ストリーム書き出しし、`write()` のピークメモリを削減 | `PackedRTree::stream_write_from_leaves()` 追加(minor) |

## 必須の対応(CI を通すため)

### 1. rustfmt の差分を解消する

`cargo fmt --check` が [file_writer.rs:429](src/rust/src/file_writer.rs#L429) 付近で差分を検出。
CI に fmt ジョブはないが、マージ前に直しておくのが無難。

```sh
cd src/rust && cargo fmt
```

### 2. semver 問題への方針を決める(最重要)

CI に `rust-semver-checks` ジョブ(cargo-semver-checks-action、crates.io の公開版 6.0.1 と比較)がある。
現在の変更は Cargo.toml のバージョン(6.0.1)を変えていないため、**API 追加だけでも CI が fail する見込み**。

- `Error` enum は `#[non_exhaustive]` が付いていない公開 enum → `IllegalColumnIndex` variant の追加は **breaking change(major bump = 7.0.0 が必要)**
- `sort_columns_by` / `stream_write_from_leaves` の追加は minor bump(6.1.0)で足りる

選択肢:

- **案A**: 既存の variant(例: `IllegalHeaderSize`)を流用するか関数シグネチャを見直して breaking を回避し、minor 相当に収める
- **案B**: PR 内で `version = "7.0.0"` に bump する(ただしバージョン bump はメンテナが `[rust] Bump to x.y.z` コミットで行う慣習。勝手に bump するより事前相談が良い)
- **案C**: PR 説明に breaking である旨を明記し、semver-checks の扱いをメンテナに委ねる

→ どの案でも PR 説明で **breaking change の有無を明示** すること。

### 3. 無関係な変更を除外する

- `examples/leaflet/filtered.html` に未コミットの空白のみの差分がある(先頭行のインデントはむしろ崩れている)。PR には含めない:

  ```sh
  git restore examples/leaflet/filtered.html
  ```

### 4. upstream に rebase する

```sh
git fetch flatgeobuf
git rebase flatgeobuf/master   # コンフリクトなしの見込み
```

## ローカル検証の状況(2026-07-16 実施)

| チェック(CI 相当) | 結果 |
|---|---|
| `cargo test`(default features) | ✅ 9 + 13 doctest 全パス(`sort_columns` テスト含む) |
| `cargo test --no-default-features` | ✅ 全パス |
| `cargo fmt --check` | ❌ 1 箇所差分あり(上記 1.) |
| `cargo check --target wasm32-unknown-unknown` | ✅ パス |
| `cargo semver-checks` | 未実施(ツール未インストール)。CI で fail する可能性が高い(上記 2.) |

## PR の出し方・体裁

### PR の分割を検討する

3 コミットは論理的に独立している。upstream のマージ履歴は 1 PR = 1 トピックが基本。推奨:

1. **PR 1**: 列順変更機能(`a65e469` + 前提となる `e64dde2` の名前ベース照合)
2. **PR 2**: `write()` のピークメモリ削減(`7a56691`)— こちらは単独で完結しており、breaking もなくマージされやすい

まとめて 1 PR でも通る可能性はあるが、レビューしやすさと「メモリ削減だけ先に取り込む」余地を考えると分割が有利。

### コミットメッセージの確認

- `7a56691` に `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` の trailer が入っている。公式 PR に残すかどうかを判断し、外すなら `git rebase` で編集する。

### PR タイトル・説明

- タイトルは `[rust]` プレフィックスを付ける慣習がある(例: `[TS] Reworked API for deserialize (#496)`)
- 説明に入れるべきこと:
  - 動機・ユースケース(なぜ列順を変えたいか / メモリ削減が必要な規模)
  - メモリ削減はビフォー・アフターの数値があると強い(`benches/` があるので計測可能)
  - breaking change の明示(上記 2.)
  - `sort_columns_by` の使用例(doctest がそのまま使える)

### 事前 issue の検討(任意)

- CONTRIBUTING.md は存在せず、明文化された手順はない。**CI green が実質的な要件**
- 既存 issue を検索した限り、列順変更・writer メモリに直接対応する open issue はない
  (closed の #209 "Rust writer produces bigger files than GDAL" が writer 品質の文脈で近い程度)
- `sort_columns_by` のような API 追加は、先に issue を立てて需要とインターフェースの感触を聞くとレビューがスムーズ

### 提出手順

```sh
git push origin feature/modify-column-order   # rebase 後は --force-with-lease
```

`gh` CLI は未インストールのため、ブラウザで以下から PR を作成:

https://github.com/flatgeobuf/flatgeobuf/compare/master...p4ken:flatgeobuf:feature/modify-column-order

## 任意(親切系)

- [src/rust/CHANGELOG.md](src/rust/CHANGELOG.md) への追記(過去はリリース時にメンテナがまとめて記載しているので必須ではない)
- `fgbutil` / `benches` 側で新 API を使う例の追加は不要(既存 PR もしていない)
