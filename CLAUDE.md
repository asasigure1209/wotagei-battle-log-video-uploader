# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

ヲタ芸バトルログの動画を ffmpeg でエンコードし、rclone で Cloudflare R2 にアップロードするための mise タスク集。ビルド・テストはなく、実体は `mise-tasks/` 配下の bash スクリプト 2 本のみ。前提ツールは mise / ffmpeg (libx264) / rclone で、rclone リモート `r2-wotagei-battle-log` が設定済みであること（詳細は README.md）。

## コマンド

```bash
mise encode --all                      # source_videos/ → encoded_videos/ (出力済みはスキップ)
mise encode --include "本戦/*.mp4"      # glob で絞り込み
mise encode --all --force              # 出力済みも再エンコード
mise upload --series elnino --event elnino_vol_5 --all --dry-run   # 転送せず対象だけ確認
mise upload --series elnino --event elnino_vol_5 --include "本戦/*.mp4"
shellcheck mise-tasks/*                # スクリプト変更時の静的チェック
```

## 構成と仕組み

- **データの流れ**: `source_videos/<分類>/*.mp4` → `mise encode` → `encoded_videos/<分類>/*.mp4`（同じディレクトリ構成）→ `mise upload` → R2。オブジェクトキーは `<シリーズID>/<イベントID>/<encoded_videos/ からの相対パス>`（例: `elnino/elnino_vol_5/予選/M1256.mp4`）。シリーズID・イベントIDは `upload` の必須フラグ `--series` / `--event` で指定し、英数字と `_` `-` だけを許可している。ローカルの `source_videos/` / `encoded_videos/` はイベント単位の作業場所で、イベントの階層は持たない。動画本体は `.gitignore` で除外し、`.gitkeep` だけ管理している。
- **mise のファイルタスク**: `mise-tasks/` 内の実行可能ファイルがそのままタスクになる。`#MISE description=...` と `#USAGE flag ...` コメントがタスク定義で、フラグは `usage_<name>` 環境変数（`--dry-run` → `usage_dry_run`、値は `"true"`）としてスクリプトに渡る。タスクを追加する際はファイルに実行権限を付けること。
- **設定値**: R2 のリモート名とバケット名は `mise.toml` の `[env]` (`R2_REMOTE`, `R2_BUCKET`) で定義し、`upload` が参照する。
- **エンコード設定**: H.264 (libx264, CRF 20, preset slow, yuv420p) + AAC 192k + `faststart`。中断時に不完全なファイルが「済み」扱いされないよう、`<out>.part` に書き出してから `mv` している。拡張子が `.mp4` でないので、コンテナ形式は `-f mp4` で明示している。
- **対象ファイルの選び方**: `encode` と `upload` は同じ方法で対象を選ぶ。`find ... -name '*.mp4'` で列挙し、相対パスを bash の `[[ $rel == $pattern ]]` で glob 照合する（`*` は `/` もまたぐので、`本戦/*.mp4` はサブディレクトリにもマッチする）。`upload` は rclone の `--include` を使わず、選んだ一覧を `--files-from-raw` で渡している。照合方法を変えるときは両方のスクリプトを揃えること。

## 注意点

- `upload` は `rclone copy` なので、ローカルで削除したファイルは R2 側から消えない。
