# ヲタ芸バトルログの動画ファイルアップロード

## 前提

- [mise](https://mise.jdx.dev/) / ffmpeg (libx264 有効) / rclone がインストールされていること
- rclone のリモート `r2-wotagei-battle-log` (Cloudflare R2) が `rclone config` で設定済みであること
- 初回は `mise trust` でこのディレクトリの設定を信頼させる

## 手順

### 1. 元の動画ファイルを分類してディレクトリに分けて保存する

以下のように、`source_videos`配下に保存する

- source_videos/
  - 予選
    - M1256.mp4
	- M1267.mp4
  - 本戦
    - M1268.mp4

### 2. 動画をエンコードして、アップロードに適したフォーマットにする

```bash
# すべての動画ファイルをエンコード
mise encode --all

# 指定した動画ファイルをエンコード
mise encode --include "本戦/*.mp4"

# 出力済みのファイルも再エンコードする
mise encode --all --force
```

`mise encode`コマンドを実行後、`encoded_videos`ディレクトリ配下に`source_videos`と同じディレクトリ構成でエンコード済みの動画が出力される。フォーマットはH.264 (CRF 20) + AAC 192k。

すでに`encoded_videos`配下に出力済みのファイルがある場合はスキップされる。作り直したいときは`--force`を指定する。

### 3. 動画をR2のバケットにアップロードする

```bash
# すべての動画ファイルをアップロード
mise upload --all

# 指定した動画ファイルをアップロード
mise upload --include "本戦/*.mp4"

# 実際には転送せず、対象のファイルだけを確認する
mise upload --all --dry-run
```

オブジェクトキーは`encoded_videos`配下の相対パスがそのまま使われる（例: `予選/M1256.mp4`）。

