# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Herdr** は AI コーディングエージェント向けのターミナルワークスペースマネージャー（Rust 製 TUI アプリケーション）。エージェントの状態（blocked/working/done/idle）を検出しながら、ワークスペース・タブ・ペインを管理するターミナルマルチプレクサ。

## Commands

```bash
# ビルド
cargo build --release

# テスト（nextest + Python スクリプト検証）
just test

# CI チェック（フォーマット確認 + nextest）
just ci

# 単一テスト実行
cargo nextest run <test_name>
# または
cargo test <test_name>

# フォーマット
cargo fmt

# Lint
cargo clippy
```

`just ci` が PR マージ前の必須チェック（`cargo fmt --check` + `cargo nextest run`）。

## Architecture

### 動作モード

アプリケーションは 3 つのモードで動作する（`src/main.rs` で dispatch）：

1. **Server モード** (`--server`): バックグラウンドで動作するヘッドレスサーバー
2. **Client モード** (`--client`): サーバーに接続する薄いクライアント
3. **Monolithic モード** (default): サーバーなしの単一プロセスモード（`--no-session`）

通常起動時はサーバーが自動検出・起動される（`src/server/autodetect.rs`）。

### コアデータフロー

```
ユーザー入力 (crossterm)
  → raw_input (RawInputEvent)
  → app::input (navigate/terminal モード分岐)
  → app::actions (純粋な状態変化)
  → AppState (ワークスペース/タブ/ペイン階層)
  → app::runtime (async イベントループ: PTY 読み取り、API、タイマー)
  → pane::terminal (GhosttyPaneTerminal - C ライブラリ統合)
  → ui (ratatui レンダリング)
```

### 主要モジュール

| モジュール | 役割 |
|-----------|------|
| `src/app/` | アプリ状態マシン、イベントループ、レンダリング統制 |
| `src/app/state.rs` | `AppState`（ワークスペース/タブ/ペイン）と `Mode`（navigate/terminal/settings 等） |
| `src/app/actions.rs` | 状態変化の純粋関数（create/close/split/rename） |
| `src/pane.rs` | ペインのライフサイクル（PTY スポーン、ターミナルエミュレーション、状態検出、スクロールバック） |
| `src/detect.rs` | エージェント識別と状態マシン（blocked/working/done/idle 遷移） |
| `src/server/` | ヘッドレスサーバー、IPC プロトコル、クライアント接続管理 |
| `src/client/` | 薄いクライアント、入力転送、フレームブリッティング |
| `src/api/` | ワークスペース/ペイン操作用 Unix ソケット IPC サーバー |
| `src/persist/` | セッションのシリアライズ/リストア |
| `src/layout.rs` | ペインタイリングアルゴリズム（縦横分割） |
| `src/ui/` | ratatui レンダリングパイプライン |
| `src/config/` | TOML 設定、キーバインド DSL、テーマ定義 |
| `src/ghostty/` | vendored libghostty-vt C ライブラリへの FFI バインディング |
| `src/platform/` | OS 固有コード（macOS/Linux、シグナル処理、プロセス情報） |

### 重要な設計上の制約

- **ネストした herdr の防止**: `HERDR_ENV=1` 環境変数で多重起動を検出・ブロック（`src/main.rs`）
- **ターミナルエミュレーション**: `portable-pty` + vendored libghostty-vt（C、Zig でビルド）を使用。`build.rs` が Zig ビルドを担当
- **非同期ランタイム**: Tokio マルチスレッドで複数の並行イベントソース（入力、PTY、API、タイマー）を処理
- **エージェント検出**: プロセス名マッチング + ターミナル出力の正規表現（ゼロ設定で動作）

### 設定・ログ

- 設定ファイル: `~/.config/herdr/config.toml`
- ログ: `~/.config/herdr/herdr.log`, `herdr-client.log`, `herdr-server.log`（ローテーティング）
- デフォルト設定: `src/main.rs` の `DEFAULT_CONFIG` 定数に埋め込み

### ビルド要件

- Rust 1.70+ (2021 edition)
- Zig コンパイラ（libghostty-vt のビルドに必要）
- Python 3（`just test` のスクリプト検証に必要）
- Linux または macOS のみサポート
