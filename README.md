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
