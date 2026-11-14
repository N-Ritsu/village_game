# ふるさと百代記で使っている外部のサービス

プログラムの外で使っているサービス・アカウント・設定の一覧です(2026-11-14 時点)。
秘密の鍵の**値**はここには書きません(どこに置いてあるかだけを書きます)。ページに載る公開の値(広告の番号など)は書いてあります。

---

## 全体の図

```
遊ぶ人のブラウザ
 ├─ 画面(HTML・JS・音・画像) … Cloudflare Pages(furusato-momoyoki.pages.dev)
 │     └─ 古いアドレス furusato-momoyoki.nrwebgame.workers.dev は、Cloudflare Workers が新しいアドレスへ案内
 ├─ ログイン・セーブ・ランキング・出稼ぎの行き来 … Supabase(ログインは Google)
 ├─ 世帯枠の購入 … Supabase の処理(Edge Functions)→ Stripe
 ├─ 広告 … Google AdSense
 └─ 訪問者の数 … Cloudflare Web Analytics
検索 … Google Search Console / 宣伝 … X・itch.io
```

ゲームの計算(1年進める・子が育つ など)は、すべてブラウザの中で行います。計算用のサーバーはありません。

---

## 1. Cloudflare(画面の配信・訪問者の数)

| 項目 | 内容 |
| --- | --- |
| 役割 | ゲームの画面(静的なファイル)の配信 / 古いアドレスの案内 / 訪問者の数の計測 |
| 料金 | 無料プラン |
| アカウント | 運営者の Cloudflare アカウント(Account ID `362a63f85d03c1d4b4413295b0200eeb`) |
| 本番のアドレス | **https://furusato-momoyoki.pages.dev**(Cloudflare **Pages** のプロジェクト `furusato-momoyoki`、本番の枝 `main`) |
| 古いアドレス | https://furusato-momoyoki.nrwebgame.workers.dev(Cloudflare **Workers** の `furusato-momoyoki`)。開いた人を、ゲストの記録ごと新しいアドレスへ移す |
| workers.dev のサブドメイン | `nrwebgame`(アカウント全体で1つ。今後のゲームにも共通で付く) |
| 訪問者の数 | **Web Analytics**(cookie を使わない)。ホスト名 `furusato-momoyoki.pages.dev`、token `ddbf23c9097849de82816a9646c0d79b`(ページに載る公開の値) |
| 公開のしかた | `worker/` で `npm run deploy`(画面 → Pages)、`npm run deploy:redirect`(古いアドレスの案内 → Workers)。ユーザーが実行する |
| ログイン | wrangler の OAuth(`npx wrangler login`)。**ターミナルに `CLOUDFLARE_API_TOKEN` が残っていると、権限のない鍵が使われて 403 になる** → 命令の前に `env -u CLOUDFLARE_API_TOKEN -u CF_API_TOKEN` を付ける |
| 設定ファイル | `worker/wrangler.jsonc`(Pages)、`worker/redirect/wrangler.jsonc`(古いアドレスの案内)、`worker/config.production.json`(画面の設定) |

**なぜ Pages か**: AdSense は `○○.nrwebgame.workers.dev` を受け付けず、大もとの `nrwebgame.workers.dev`(何も置けない)を求めた。`○○.pages.dev` は、それ自体が大もとの扱いになるので、2026-11-09 に移した。

---

## 2. Supabase(ログイン・セーブ・ランキング・購入の処理)

| 項目 | 内容 |
| --- | --- |
| 役割 | ログイン(Google)/ セーブ / 購入の記録 / ランキング / 出稼ぎの行き来(ほかのプレイヤーの村との縁談) |
| 料金 | 無料プラン |
| プロジェクト | URL `https://uqepsgugcxwuxgwtafcw.supabase.co` |
| 表 | `saves`(セーブ)、`purchases`(購入の記録)、`rankings`(ランキング)、`city_pool`(都会に出た子) |
| 処理(Edge Functions) | `create-checkout`(支払いページを作る)、`confirm-checkout`(支払いを確かめて世帯枠を足す) |
| 秘密の値(本番) | `supabase secrets set` で登録: `STRIPE_SECRET_KEY`、`SITE_URL`(= `https://furusato-momoyoki.pages.dev`)。`SUPABASE_URL` などは Supabase が自動で渡す |
| 秘密の値(手元) | `supabase/functions/.env`(**公開しない・会話に貼らない**) |
| ログインの戻り先 | 管理画面の Authentication → URL Configuration。Site URL と Redirect URLs に `https://furusato-momoyoki.pages.dev`(古いアドレスも残してある) |
| データベースの変更 | `supabase/migrations/` に書き、ユーザーが `supabase db push` で本番に入れる |
| 注意 | 本番は、新しい表に API 用の役割の権限を**自動では付けない**。表を足したら `grant` を必ず書く |
| 管理用の作業 | `worker/scripts/backfill-rankings.mjs`(今までのセーブからランキングを作る。管理用の鍵は実行時に聞く) |
| 人数を見る | `supabase/daily_players.sql` を SQL Editor に貼る(ログインした人だけが数えられる) |
| 手元で動かす | Docker Desktop を起動して `supabase start`。テストは `node test/test_db.mjs` |

---

## 3. Google(ログイン・広告・検索)

### 3-1. Google Cloud(Google ログイン)
| 項目 | 内容 |
| --- | --- |
| 役割 | 「Googleでログイン」の OAuth クライアント |
| 状態 | 同意画面は「公開」済み(テストユーザー以外もログインできる) |
| 戻り先 | Supabase の受け口(`…supabase.co/auth/v1/callback`)。アドレスを変えても、ここは変えなくてよい |
| 鍵 | クライアント ID・シークレットは、Supabase の Authentication → Providers → Google に登録してある |

### 3-2. Google AdSense(広告)
| 項目 | 内容 |
| --- | --- |
| サイト運営者の番号 | `ca-pub-3259688166266003`(ページに載る公開の値)。`worker/config.production.json` の `adsenseClient` |
| サイト | `furusato-momoyoki.pages.dev`(2026-11-09 に審査をリクエスト。**審査中**) |
| 広告ユニット | まだ作っていない。審査に通ったら、ディスプレイ広告(レスポンシブ)を作り、番号を `adsenseSlot` に入れる → ゲームの下に1枠出る |
| ads.txt | ビルドが自動で作る(`google.com, pub-3259688166266003, DIRECT, f08c47fec0942fa0`) |
| 決めたこと | 自動広告はオフにする(画面に張り付く広告が、長老の案内やボタンに重なるため)。ヨーロッパ向けの同意画面は、AdSense の「プライバシーとメッセージ」で作る |
| 支払い | 収益1,000円で住所確認のはがき(PIN)、8,000円で振り込み。契約の更新・料金はなし |
| アカウント | 1人1つ。今後のゲームは、同じアカウントの「サイト」に足していく |

### 3-3. Google Search Console(検索)
| 項目 | 内容 |
| --- | --- |
| プロパティ | URL プレフィックス `https://furusato-momoyoki.pages.dev/` |
| 持ち主の確認 | HTML タグ。値は `worker/config.production.json` の `googleSiteVerification`(ビルドが head に入れる) |
| サイトマップ | `sitemap.xml`(2026-11-12 に送信。「取得できませんでした」は Google 側の表示で、様子見) |
| 関係するファイル | `robots.txt`、`sitemap.xml`、`index.html` の構造化データ(JSON-LD)・紹介文 |

---

## 4. Stripe(決済)

| 項目 | 内容 |
| --- | --- |
| 役割 | 世帯枠 +10 の購入(100円・税込・カードのみ。最大8回で100世帯まで) |
| アカウント | **別のゲーム policy_simulator と共有**。見分け方: 商品名「ふるさと百代記 世帯枠 +10」、`metadata.game = furusato`、明細の接尾辞 `FURUSATO` |
| 鍵 | ゲームごとの**制限付きキー**。Supabase の秘密の値 `STRIPE_SECRET_KEY` に入れてある |
| 仕組み | Webhook は使わない。支払いページから戻ったとき・ログインしたときに、処理が Stripe に問い合わせて確かめる |
| 手数料 | Stripe の決済手数料がかかる(1回100円の売り上げから引かれる) |
| 過去に使ったもの | 決済リンク(Payment Links)は無効化済み |
| 気をつけること | ビジネスウェブサイトが別のゲームだけだと、入金が止まったことがある。事業内容に、売っているものをすべて書いておく |

---

## 5. 宣伝・連絡先

| サービス | 内容 |
| --- | --- |
| X(公式) | https://x.com/RitsuDXIT(ゲームの画面の下・リンクカードの `twitter:site`) |
| itch.io | ゲームのページ(起動画面だけを上げ、押すと本物のゲームを新しいタブで開く)。素材は `itch/` |
| 問い合わせ | nrwebgame@gmail.com(プライバシーポリシー・特定商取引法の表記に載せている専用の Gmail) |
| リンクカード | `og.png`(1200×630)と `index.html` のメタタグ |

---

## 6. 開発に使っているもの(遊ぶ人には関係しない)

| もの | 内容 |
| --- | --- |
| GitHub | `N-Ritsu/village_game`。**作業履歴などの文書(`docs/`)だけ**を送る。コードは送らない(`.gitignore` で除外) |
| Node.js | v24(`~/.local/node/bin`。PATH には入っていない) |
| wrangler | 4.149.0(Cloudflare への公開) |
| Supabase CLI | 2.120.0 |
| Docker Desktop | 手元の Supabase を動かすため(WSL 連携をオン) |
| Playwright | 実ブラウザでの確認(使い捨ての環境に 1.64.1 を入れる) |
| Remotion | 4.0.536。PV(宣伝動画)を作るため。個人なら商用でも無料 |
| Claude(生成AI) | コード・絵(コードで描いた絵)・文章・曲(コードで作った曲)を一緒に作った。itch.io では生成AIの使用を申告する |

---

## 7. お金のかかるもの・入るもの

| | 内容 |
| --- | --- |
| 毎月かかるお金 | なし(Cloudflare・Supabase・Google はすべて無料の範囲) |
| 独自ドメイン | 使っていない(更新の費用もなし) |
| 入るお金 | 世帯枠の購入(Stripe の手数料を引いた額)、広告(AdSense。審査に通ってから) |
| 無料の範囲を超えそうなとき | Supabase の無料プランは、使われない状態が続くとプロジェクトが一時停止することがある。データベースの容量・通信量の上限もある。人が増えたら、管理画面の使用量を見る |
