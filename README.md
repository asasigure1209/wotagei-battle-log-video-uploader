# ヲタ芸バトルログの動画ファイルアップロード

本番に動画をアップロードして、Web アプリで表示を確かめるまでの通しの手順は [docs/production-upload.md](docs/production-upload.md) にまとめている。

## 前提

- [mise](https://mise.jdx.dev/) / ffmpeg (libx264 有効、`mise manifest` は同梱の ffprobe を使う) / rclone がインストールされていること
  - Homebrew なら `brew install ffmpeg rclone` で入る (Homebrew の ffmpeg は libx264 入り)
- rclone のリモート `r2-wotagei-battle-log` (Cloudflare R2) が `rclone config` で設定済みであること
  - R2 の API トークンをこのバケットだけに絞っている場合は、`no_check_bucket = true` も設定する。設定しないと、rclone がバケットを作れるか確かめようとして 403 になることがある

    ```bash
    rclone config update r2-wotagei-battle-log no_check_bucket true
    ```

  - 接続は `rclone lsf r2-wotagei-battle-log:wotagei-battle-log` で確かめられる (バケットを絞ったトークンでは、全バケットを一覧する `rclone lsd r2-wotagei-battle-log:` は 403 になるが問題ない)
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

`--series`にイベントシリーズID、`--event`にイベントIDを指定する（どちらも必須。英数字と`_` `-`のみ）。

```bash
# すべての動画ファイルをアップロード
mise upload --series elnino --event elnino_vol_5 --all

# 指定した動画ファイルをアップロード
mise upload --series elnino --event elnino_vol_5 --include "本戦/*.mp4"

# 実際には転送せず、対象のファイルだけを確認する
mise upload --series elnino --event elnino_vol_5 --all --dry-run
```

オブジェクトキーは`<イベントシリーズID>/<イベントID>/<encoded_videos配下の相対パス>`になる（例: `elnino/elnino_vol_5/予選/M1256.mp4`）。

### 4. Web アプリの DB に取り込むデータを作る

```bash
mise manifest --series elnino --event elnino_vol_5
```

`encoded_videos`配下の動画を調べ、`manifests/<イベントシリーズID>/<イベントID>.yaml`に、Web アプリの DB に取り込むデータの雛形を書き出す。動画ごとのオブジェクトキー・長さ（秒）・サイズ（バイト）は自動で入る。

書き出した YAML に、次を書き込む。

- `series.name` / `event.name` / `event.date`（`YYYY-MM-DD`）/ `event.contact`
- `entries`: エントリー No. と名前（1 行に 1 人。例: `1: BLAZE`）
- 動画ごとの `performers`: 出ているエントリー No.（予選は 1 人で`[1]`、本戦は 2 人で`[1, 2]`）

`videos`の並び順が、イベントの中での動画の No. になる。書き出した直後は分類のディレクトリ順・ファイル名順なので、必要なら並べ替える。

書き込んだ内容を消さないよう、YAML がすでにあるときは書き出さない。作り直すときは`--force`を付ける。書き終えた YAML は、Web アプリの`db:import`で DB に取り込む。

## 開発

スクリプト (`mise-tasks/` 配下) を変えたら、shellcheck でチェックする。shellcheck は `mise.toml` の `[tools]` に入れているので、`mise install` で入る。

```bash
shellcheck mise-tasks/*
```

スクリプトは macOS 標準の bash 3.2 でも動くように書く。たとえば `set -u` の下で空の配列を `"${arr[@]}"` と展開するとエラーになるので、`${arr[@]+"${arr[@]}"}` と書く。
