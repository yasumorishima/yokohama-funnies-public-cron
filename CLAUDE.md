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
| `sync-roles.yml` | 5 分おき・手動・`repository_dispatch: sync-roles`・下の push | private repo の `config/members*.yml` を読み、Supabase の role を同期。承認メール・選手の紐付けも |
| `health-check.yml` | 毎時・手動 | 会員パイプラインの監視。異常時だけ private repo に Issue とメール（一過性では通知しない） |
| `health-check-ack.yml` | `repository_dispatch: health-check, gas-heartbeat` | 監視の往復の応答側 |
| `keep-alive.yml` | 週 1 | サイトを叩いて Supabase Free の自動休止を防ぐ |
| `purge-deleted-photos.yml` | 毎日・手動 | 論理削除から 7 日過ぎた写真を物理削除 |
| `warm-weather.yml` | 30 分おき | `/weather` のキャッシュを温める |
| `update-readme-stats.yml` | 毎月 1 日 | private repo の README の統計欄を更新 |

## sync-roles をすぐ回す（cloud から）
- ブランチ `claude/ops-YYYYMMDD` に **`requests/sync-roles.json`** を push すると sync-roles が起動する。
  ```json
  {"memo": "PR 承認後の即時同期", "at": "2026-10-24T10:00:00+09:00"}
  ```
- 中身は memo と時刻だけ。**名前・UID・PR の中身は書かない**（public）。
- このブランチは main へ merge しない。会員承認の前後の手順は private 側 `CLAUDE.md` の「会員承認（3 段）」。

## Session protocol
- 返答は日本語・です/ます。走らせていないこと・確かめていないことは書かない。
- 変更は PR で出し、checks を見てから自分で merge する。終わったら private 側 `CLAUDE.md` の Progress log に 1 行書く。
