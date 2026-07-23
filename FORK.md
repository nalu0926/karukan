# nalu0926/karukan フォーク変更点

upstream: [togatoga/karukan](https://github.com/togatoga/karukan)

個人カスタマイズのため upstream に PR は出さない。

## ビルド・インストール

```bash
# 前提: Rust toolchain, cmake (brew install cmake)
cd karukan-macos && make install
# 初回はログアウト→ログインが必要（macOS が IME を認識するため）

# 2回目以降
make install && killall KarukanIME
```

## upstream の取り込み

```bash
git fetch upstream
git rebase upstream/main
# コンフリクトがあれば解消後 make install && killall KarukanIME
```

## フォーク変更一覧

### 1. Ctrl+K（カタカナモード切替）無効化

誤爆防止。使用中のキーボードに右 Ctrl がなく、左 Ctrl+K で意図せずカタカナモードに入る。

- `karukan-im/src/core/engine/input.rs`
  - `Keysym::KEY_K | Keysym::KEY_K_UPPER => return self.enter_katakana_mode()` をコメントアウト

### 2. Shift+アルファベット（半角 Alphabet モード切替）無効化

日本語入力中に Shift+英字で意図せず Alphabet モードに切り替わるのを防止。

- `karukan-im/src/core/engine/input.rs`
  - `process_key_empty` 内の Shift+alpha 検出（一時 Alphabet モード切替）を削除
  - `process_key_composing` 内の同様のブロックを削除
- 英数キー（JIS）や右⌘タップでのモード切替は引き続き有効
- upstream #38 は同じ問題を「単語単位の一時モード」で緩和したが、誤爆時に当該単語が英字化する挙動自体は残るため、フォークでは引き続き完全無効化
- この無効化（および 1. の Ctrl+K 無効化）が前提の upstream テスト29件は `#[ignore = "fork: ..."]` を付与している

### 3. 学習キャッシュの読み最大文字数制限

長い読み（文節以上）の学習を抑制し、予測変換候補の汚染を防ぐ。

- `karukan-engine/src/learning.rs`: upstream #64 の `LearningConfig` に `max_reading_chars` フィールドを追加（upstream の `max_surface_chars`＝surface 基準と併用）。`record()` で読みが上限超過なら学習スキップ
- `karukan-im/src/config/settings.rs`: `LearningSettings` に `max_reading_chars` 追加
- `karukan-im/config/default.toml`: `max_reading_chars = 8`
- `karukan-im/src/core/engine/init.rs`: 初期化時に `LearningConfig` 経由で渡す

`config.toml` の `[learning]` セクションで変更可能（0 = 無制限）:

```toml
[learning]
max_reading_chars = 8
```

### 4. Shift+Space で候補を逆方向に移動

変換候補選択中に Shift+Space で前の候補に戻る（一般的な IME の挙動に合わせる）。

- `karukan-im/src/core/engine/conversion.rs`
  - `process_key_conversion` で `Keysym::SPACE` に shift guard を追加し `prev_candidate()` を呼ぶ

### 5. 学習 surface ブロックリスト（部分一致）

特定の変換結果を学習対象から除外する。ブロックリスト語を**含む** surface も対象（部分一致）。

- `~/Library/Application Support/com.karukan.karukan-im/learning_blocklist.txt`（1行1語、`#` コメント可）
- 起動時ロード＋既存エントリのパージ、以降は定期パージ（`purge_learning_blocklist`）
- `karukan-engine/src/learning.rs`, `karukan-im/src/core/engine/init.rs`, `karukan-im/src/server/mod.rs`
- 閲覧・削除・ブロックリスト編集の Web UI: `scripts/learning_manager.py`（未コミットのローカルツール）

### 6. Ctrl+H / Ctrl+D を削除キーとして明示処理

エンジンが Ctrl+H を not_consumed で返すと、macOS では Cocoa 標準の `deleteBackward:` がクライアント側で発火し、marked text を迂回して確定済みテキストが削れる。Composing 状態に Ctrl+H=Backspace / Ctrl+D=Delete、Conversion 状態に Ctrl+H=Backspace を追加。

- `karukan-im/src/core/engine/input.rs`, `conversion.rs`, `keycode.rs`

## 運用メモ

- 学習キャッシュ: `~/Library/Application Support/com.karukan.karukan-im/learning.tsv`
- LaunchAgent `com.nalu.karukan-clear` で毎日 4:00 AM に learning.tsv を削除（日次リセット）
