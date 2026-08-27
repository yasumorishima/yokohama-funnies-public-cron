# yokohama-funnies-public-cron

横浜ファニーズ サイト ([yokohama-funnies](https://github.com/yasumorishima/yokohama-funnies), private / 技術解説 (public): [yokohama-funnies-docs](https://github.com/yasumorishima/yokohama-funnies-docs)) の **public 化可能な cron workflow** を分離した public repo。

## Purpose

- GitHub Actions の無料枠を public repo の unlimited 枠で消化、 private repo の quota 圧迫を回避
- single point of failure (RPi5 self-hosted runner) からの脱却 (2026-05-16 RPi5 SSD 故障で funnies private 側 cron が 5/16 13:16 UTC 以降全 queued 状態 → 本 repo で migration)
- RPi5 cron は defense-in-depth として並行稼働継続 (即削除しない)

## Workflows

| File | Schedule | 役割 |
|---|---|---|
| `warm-weather.yml` | `*/30 * * * *` | funnies の `/weather` を HTTP GET で warm、 Vercel ISR cache (`revalidate=1800`) を refresh |
| `keep-alive.yml` | `0 0 * * 0` (週次) | `/schedule` を HTTP GET して SSR 経由で Supabase fetch を起こし、 Free plan の 7 日無活動 auto-pause を回避 (anon key 不要) |
| `purge-deleted-photos.yml` | `0 19 * * *` (毎日 JST 4:00) + dispatch | soft-delete >7日の photos を Supabase Storage+DB から物理削除 (2026-05-30 private repo から移行) |
| `member-request.yml` | `repository_dispatch: member-request` | サイトからの入会申請を受けて、 private repo (yokohama-funnies) に `config/members/<uid>.yml` を追加する PR を App token で作成。 申請時点の氏名 (`display_name`) を書くのもこの workflow |
| `sync-roles.yml` | `*/5 * * * *` + dispatch (`sync-roles`) | private repo の `config/members/*.yml` (per-uid) と `config/members.yml` (手編集 allowlist) を統合して読み、 Supabase `user_roles` の `role` / `graduation_class` を同期 |
| `health-check.yml` | `7 * * * *` + dispatch | 上 2 本 (会員パイプライン) の死活監視。 Vercel proxy → GAS → `repository_dispatch` の往復と GAS heartbeat の鮮度を検査し、 異常時のみ private repo に追跡 issue + メール。 **一過性 (proxy 5xx / Actions 障害 / GAS トリガー遅延) では通知しない** — 詳細は各ステップのコメント |
| `health-check-ack.yml` | `repository_dispatch: health-check, gas-heartbeat` | 上の往復の応答側。 GAS が打ち返した dispatch を受けて run を残すだけ (probe はこの run の `created_at` を見る) |
| `update-readme-stats.yml` | `0 0 1 * *` (毎月1日 09:00 JST) | yokohama-funnies を App token で checkout して `scripts/update-readme-stats.sh` を実行し、 README の `<!--stat:KEY-->...<!--/stat-->` marker を実測値で更新 |

`runs-on: ubuntu-latest` で public 無料枠運用。

## Required GitHub Secrets

| Name | 必須 | 値 |
|---|---|---|
| `VERCEL_APP_URL` | ✅ | funnies の deployed URL (`https://yokohama-funnies.vercel.app`) |

Settings → Secrets and variables → Actions から設定。

## What is NOT here

- ❌ 会員情報 / Supabase auth code
- ❌ 写真 / 名前 / 個人情報
- ❌ 認証付き API endpoint の実装 (本 repo は public endpoint を warm するのみ)

minami の同種 repo は [minami-public-cron](https://github.com/yasumorishima/minami-public-cron)。
