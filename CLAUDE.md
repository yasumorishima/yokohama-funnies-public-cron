# CLAUDE.md — yokohama-funnies-public-cron

横浜ファニーズのサイト（private の `yasumorishima/yokohama-funnies`）のための GitHub Actions だけを置く public リポジトリです。
**運用の正本は private 側の `CLAUDE.md`**。ここには、この repo を触るときに要ることだけを書きます。

## 守ること
- **Actions のログは誰でも読める**。名前・メール・背番号と人の対応など、個人の情報が出る処理を置かない。
  - ログに出すのは UID の先頭 8 桁まで。
  - `client_payload` などの個人情報を step の `env:` に入れない（step の `env:` はログの見出しに平文で出る）。
- 秘密は Secrets だけ。値を `echo` しない（トークンは `::add-mask::`）。
- `health-check.yml` は **CRLF**（他は LF）。書き換えで全行の差分にしない。
- `member-request.yml` は申請者の氏名とメールをイベントのファイルから読み、ジョブの最初に `::add-mask::` で登録している（2026-10-24 まで env に入れていてログに平文で出ていた）。この形を崩さない。
- 監視（`health-check.yml` / `scripts/`）は `minami-public-cron` に同じ写しがある＝**片方だけ直さない**。
- README の Workflows 表は自動では更新されない。workflow を足したら表も直す。

## workflow
| file | 起動 | 役割 |
|---|---|---|
| `member-request.yml` | `repository_dispatch: member-request` | 入会申請を受けて private repo に `config/members/<uid>.yml` の PR を作る。`user_roles` の承認待ち行もここで作る |
| `sync-roles.yml` | 5 分おき・手動・`repository_dispatch: sync-roles` | private repo の `config/members*.yml` を読み、Supabase の role を同期。承認メール・選手の紐付けも |
| `health-check.yml` | 毎時・手動 | 会員パイプラインの監視。異常時だけ private repo に Issue とメール（一過性では通知しない） |
| `health-check-ack.yml` | `repository_dispatch: health-check, gas-heartbeat` | 監視の往復の応答側 |
| `keep-alive.yml` | 週 1 | サイトを叩いて Supabase Free の自動休止を防ぐ |
| `purge-deleted-photos.yml` | 毎日・手動 | 論理削除から 7 日過ぎた写真を物理削除 |
| `warm-weather.yml` | 30 分おき | `/weather` のキャッシュを温める |
| `update-readme-stats.yml` | 毎月 1 日 | private repo の README の統計欄を更新 |

## sync-roles の即時起動
- 会員承認の直後の即時同期は、private の実行役 `yasumorishima/site-ops` が `workflow_dispatch` で起動する（cloud セッションから直接は起動できない）。
- ⛔ `push` などブランチ上で動く起動条件を足さない（2026-10-24 にいったん足して戻した）。
- ⚠️ ただし、それだけでは鍵は守れない。cloud セッションがこの repo につながれていれば、ブランチに workflow を1 本足すだけで repo の Secrets（Supabase の service role キー・GitHub App の秘密鍵）を読める。⛔ **この repo を cloud につながない**（2026-10-25）。
- ⛔ **鍵を env に持つ step でサイト repo の clone のコードを走らせない。clone に書くのは秘密を含まない中身だけ**（2026-10-25。member-request の会員ファイル＝uid・role・期。README の数字は鍵のない step で書く）。サイトの README から読んだ値は数字か確かめてから使う（`sed` の置換や `$(( ))` に生のまま入れるとコードが走る）。clone の中身は cloud セッションが書き換えられる。python を clone の中で走らせるなら `python3 -P`（置かれた `yaml.py` 等を読まない）、サイトの checkout は `persist-credentials: false`、件数を取る鍵は `scripts/count-db-rows.sh` の step だけ、README は symlink なら止まる。サイトの README から他の README へ写す値は数字だけ。

## Session protocol
- 返答は日本語・です/ます。走らせていないこと・確かめていないことは書かない。
- 変更は PR で出し、checks を見てから自分で merge する。終わったら private 側 `CLAUDE.md` の Progress log に 1 行書く。
