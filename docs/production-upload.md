# 本番に動画をアップロードする手順

1 つのイベントの動画をエンコードして R2 にアップロードし、Web アプリの本番の DB に取り込んで、本番ページで表示を確かめるまでの手順。

使うリポジトリは 2 つ。どちらも同じ親ディレクトリに置いている前提で書く。

- このリポジトリ (wotagei-battle-log-video-uploader): エンコード・アップロード・manifest の書き出し
- wotagei-battle-log-web-app: manifest を DB に取り込む (`db:import`)

以下、例としてシリーズID `elnino`・イベントID `elnino_vol_5` を使う。ID は英数字と `_` `-` だけで書く。

| 項目 | 本番の値 |
| --- | --- |
| Web アプリ | https://wotagei-battle-log-web-app.vercel.app |
| D1 の中継 Worker | https://wotagei-battle-log-d1-proxy.mkmokymh.workers.dev |
| R2 | リモート `r2-wotagei-battle-log`、バケット `wotagei-battle-log` |

## 0. 準備 (初回だけ)

### ツール

```bash
brew install ffmpeg rclone   # Homebrew の ffmpeg は libx264 入り
cd wotagei-battle-log-video-uploader && mise trust && mise install
cd ../wotagei-battle-log-web-app && mise trust && mise install && bun install
```

### rclone のリモート

`rclone config` で、Cloudflare R2 のリモート `r2-wotagei-battle-log` を作る。R2 の API トークンをこのバケットだけに絞っている場合は、`no_check_bucket` も設定する (設定しないと、アップロードのときに 403 になることがある)。

```bash
rclone config update r2-wotagei-battle-log no_check_bucket true
rclone lsf r2-wotagei-battle-log:wotagei-battle-log   # エラーが出なければつながっている
```

`rclone lsd r2-wotagei-battle-log:` (全バケットの一覧) は、バケットを絞ったトークンでは 403 になるが問題ない。

### 本番の DB につなぐための設定ファイル

`db:import` は、中継 Worker に SQL を送って本番の D1 に書き込む。Worker は受け取った SQL をそのまま実行するので、トークン (Worker の `PROXY_TOKEN`) を知っている人だけが使えるようにしている。Worker の URL とトークンを、リポジトリの外のファイルに置く。

```bash
mkdir -p ~/.config/wotagei
umask 077   # 自分だけが読めるファイルにする
cat > ~/.config/wotagei/prod.env <<'EOF'
NUXT_D1_PROXY_URL=https://wotagei-battle-log-d1-proxy.mkmokymh.workers.dev
NUXT_D1_PROXY_TOKEN=<Vercel の NUXT_D1_PROXY_TOKEN と同じ値>
EOF
```

トークンがわからなくなったときは、[トークンを作り直す](#トークンを作り直す) を見る。

## 1. 元の動画を置く

`source_videos/` と `encoded_videos/` は 1 つのイベント用の作業場所で、イベントごとの階層はない。**前のイベントの動画が残っていると、それも今回のイベントのキーでアップロードされてしまう**ので、始める前に空にする (`.gitkeep` は残す)。

```bash
find source_videos encoded_videos -mindepth 1 -not -name .gitkeep   # 何も出なければ空
```

元の動画を、分類 (予選・本戦など) ごとのディレクトリに分けて置く。ファイル名は .mp4 にする。

```
source_videos/
  予選/
    M1256.mp4
    M1257.mp4
  本戦/
    M1268.mp4
```

## 2. エンコードする

```bash
mise encode --all
```

`encoded_videos/` に、同じディレクトリ構成でエンコード済みの動画ができる。出力済みのファイルはスキップするので、途中で止まっても同じコマンドで続きから再開できる。

## 3. R2 にアップロードする

まず `--dry-run` で、対象のファイルと送り先を確かめる。

```bash
mise upload --series elnino --event elnino_vol_5 --all --dry-run
```

`upload to: r2-wotagei-battle-log:wotagei-battle-log/elnino/elnino_vol_5` と、今回のイベントの動画だけが出ていれば、`--dry-run` を外してアップロードする。

```bash
mise upload --series elnino --event elnino_vol_5 --all
rclone lsl r2-wotagei-battle-log:wotagei-battle-log/elnino/elnino_vol_5   # R2 に入ったファイルとサイズ
```

R2 のキーは `elnino/elnino_vol_5/予選/M1256.mp4` のように、`<シリーズID>/<イベントID>/<encoded_videos からの相対パス>` になる。アップロード済みのファイルは rclone が飛ばすので、途中で止まっても同じコマンドで再開できる。

## 4. manifest を書く

```bash
mise manifest --series elnino --event elnino_vol_5
```

`manifests/elnino/elnino_vol_5.yaml` に、動画ごとのキー・長さ・サイズが入った雛形ができる。これに次を書き込む。

- `series.name`・`event.name`・`event.date` (`YYYY-MM-DD`)・`event.contact`
- `entries`: エントリー No. と名前を 1 行に 1 人 (例: `1: BLAZE`)
- 動画ごとの `performers`: 出ているエントリー No. (予選は `[1]`、本戦は `[1, 2]` のように)

`videos` の並び順が、イベントの中での動画の No. になる。雛形は分類のディレクトリ順・ファイル名順に並んでいるので、必要なら並べ替える。

書き込んだ内容を消さないよう、YAML がすでにあると `mise manifest` は書き出さない。作り直すときだけ `--force` を付ける。書き込んだ YAML は、あとで直して取り込み直すときに使うので残しておく。

## 5. 本番の DB に取り込む

Web アプリのリポジトリで実行する。設定ファイルの環境変数がそのあとのコマンドに残らないよう、`( )` の中だけで読み込む。

```bash
cd ../wotagei-battle-log-web-app
manifest=../wotagei-battle-log-video-uploader/manifests/elnino/elnino_vol_5.yaml

# 確かめるだけ (DB には書かない)
(set -a; . ~/.config/wotagei/prod.env; set +a; bun run db:import "$manifest" --dry-run)
```

次の 2 点を確かめる。

- `取り込み先:` が本番の Worker の URL (`https://wotagei-battle-log-d1-proxy.mkmokymh.workers.dev`) になっている。`file:.data/db.sqlite` と出たら、設定ファイルを読めていない
- エントリー・動画・出演の件数が manifest と合っている。書き漏れなどがあれば、取り込む前にまとめて一覧が出るので、YAML を直してやり直す

問題なければ取り込む。

```bash
(set -a; . ~/.config/wotagei/prod.env; set +a; bun run db:import "$manifest")
```

取り込みは、そのイベントのエントリー・動画・出演者を入れ直す (同じシリーズのほかのイベントには触れない)。1 つのトランザクションで流すので、途中で失敗しても DB は変わらない。YAML を直したときは、同じコマンドでもう一度取り込めばよい。

## 6. 本番ページで確かめる

本番の Web アプリには Vercel のログインがかかっているので、Vercel にログインしたブラウザで開く。

| ページ | 確かめること |
| --- | --- |
| https://wotagei-battle-log-web-app.vercel.app/elnino/elnino_vol_5 | イベント名と、エントリーの全員が並んでいる |
| https://wotagei-battle-log-web-app.vercel.app/elnino/elnino_vol_5/players/01 | そのエントリーが出ている動画のファイル名・長さ・サイズ。No. は 2 桁 (`01`) で書く |

何人か選んで、出ている動画が manifest の `performers` どおりか確かめる。

動画の再生・ダウンロードは、まだ Web アプリ側にない。今はダウンロードボタンを押しても保存済みの見た目になるだけで、R2 の動画は読まない。

## 7. 後片付け

- `source_videos/` と `encoded_videos/` を空にする (次のイベントを始める前に、必ず空にする)
- `manifests/` の YAML は、取り込み直すときのために残しておく

## 困ったとき

### アップロードが 403 になる

rclone のリモートに `no_check_bucket = true` が入っているか確かめる ([0. 準備](#rclone-のリモート))。

### 本番ページが 500 エラーになる

Web アプリが本番の DB を読めていない。Vercel の環境変数 `NUXT_D1_PROXY_URL` が本番の Worker の URL になっているか、`NUXT_D1_PROXY_TOKEN` が Worker の `PROXY_TOKEN` と同じか確かめる。環境変数を変えたら、Vercel で再デプロイしないと反映されない。

Worker とトークンが合っているかは、次で確かめられる (200 で結果が返れば合っている。401 ならトークンが違う)。

```bash
(set -a; . ~/.config/wotagei/prod.env; set +a
 curl -s -w '\nHTTP %{http_code}\n' -X POST "$NUXT_D1_PROXY_URL/query" \
   -H "authorization: Bearer $NUXT_D1_PROXY_TOKEN" -H 'content-type: application/json' \
   -d '{"sql":"select id, name from event_series","params":[]}')
```

### 間違えて取り込んだイベントを消す

`db:import` には消す機能がないので、SQL で消す。イベントを消すと、そのエントリー・動画・出演者も一緒に消える (`ON DELETE CASCADE`)。シリーズとほかのイベントは残る。

```bash
(set -a; . ~/.config/wotagei/prod.env; set +a
 curl -s -X POST "$NUXT_D1_PROXY_URL/query" \
   -H "authorization: Bearer $NUXT_D1_PROXY_TOKEN" -H 'content-type: application/json' \
   -d '{"sql":"delete from events where series_id = ? and id = ?","params":["elnino","elnino_vol_5"]}')
```

R2 の動画も消すときは、`rclone purge` で消す。`upload` は `rclone copy` なので、ローカルで消しても R2 からは消えない。

```bash
rclone purge r2-wotagei-battle-log:wotagei-battle-log/elnino/elnino_vol_5
```

### トークンを作り直す

トークンは Worker の `PROXY_TOKEN` と Vercel の `NUXT_D1_PROXY_TOKEN` に同じ値を入れる。本番のアプリが DB を読めなくなる時間を短くするため、Vercel を先に書き換えておき、Worker を変えたらすぐ再デプロイする。

1. 新しい値を作って `~/.config/wotagei/prod.env` に書く

   ```bash
   umask 077
   cat > ~/.config/wotagei/prod.env <<EOF
   NUXT_D1_PROXY_URL=https://wotagei-battle-log-d1-proxy.mkmokymh.workers.dev
   NUXT_D1_PROXY_TOKEN=$(openssl rand -hex 32)
   EOF
   ```

2. Vercel の Settings → Environment Variables で `NUXT_D1_PROXY_TOKEN` を同じ値に書き換える (再デプロイするまでは反映されない)
3. Worker の `PROXY_TOKEN` を書き換える。ここから 4 が終わるまで、本番のアプリは DB を読めない

   ```bash
   cd wotagei-battle-log-web-app
   bunx wrangler login
   grep TOKEN ~/.config/wotagei/prod.env | cut -d= -f2 \
     | bunx wrangler secret put PROXY_TOKEN -c workers/d1-proxy/wrangler.toml
   ```

4. すぐに Vercel の Deployments から本番を Redeploy する
5. [本番ページが 500 エラーになる](#本番ページが-500-エラーになる) のコマンドで、200 が返るか確かめる
