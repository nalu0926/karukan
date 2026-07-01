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
  - `process_key_empty` 内の `self.input_mode = InputMode::Alphabet`（Shift+alpha 検出）をコメントアウト
  - `process_key_composing` 内の同様のブロックをコメントアウト
- 英数キー（JIS）や右⌘タップでのモード切替は引き続き有効

### 3. 学習キャッシュの読み最大文字数制限

長い読み（文節以上）の学習を抑制し、予測変換候補の汚染を防ぐ。

- `karukan-engine/src/learning.rs`: `LearningCache` に `max_reading_chars` フィールド。`record()` で読みが上限超過なら学習スキップ
- `karukan-im/src/config/settings.rs`: `LearningSettings` に `max_reading_chars` 追加
- `karukan-im/config/default.toml`: `max_reading_chars = 8`
- `karukan-im/src/core/engine/init.rs`: 初期化時に `max_reading_chars` を渡す

`config.toml` の `[learning]` セクションで変更可能（0 = 無制限）:

```toml
[learning]
max_reading_chars = 8
```

## 運用メモ

- 学習キャッシュ: `~/Library/Application Support/com.karukan.karukan-im/learning.tsv`
- LaunchAgent `com.nalu.karukan-clear` で毎日 4:00 AM に learning.tsv を削除（日次リセット）
