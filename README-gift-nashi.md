# 差し入れ・お手紙・のし紙印刷システム README

小林タカラヅカ倶楽部の観劇会で、ジェンヌさんへの差し入れ・お手紙の協力募集から、のし紙印刷までを担うシステムです。`entry-form.html`系とは別テーブル・別フローで動いています。

---

## 1. 全体の流れ

1. **`jenne-gift-form.html`**（まり管理者用・要ログイン）でジェンヌさんを登録し、今回の観劇会に紐づけて確定。お菓子担当（当番）も選択。
2. 観劇会開催の**14日前に自動で**（Supabase Edge Function）、参加決定者へ`gift-letter-form.html`のURLを含む案内メールが一斉送信される（詳細は10章）。手動で送りたい場合は`admin-list.html`の「🎁 差し入れ・お手紙 ご案内メール」からも送信可能。
3. 参加者が**`gift-letter-form.html`**（参加者向け・ログイン不要）で協力を申し込む（メールアドレス必須）。
4. **`gift-letter-list.html`**（集計用・ログイン不要）で申込状況・合計金額・当番を確認。一番下から御礼メール（写真付き）を送信。
5. **`nashi-print.html`**（まり管理者用・要ログイン）でのし紙の背景を登録・位置調整。
6. 当番の担当者が**`gift-staff-dashboard.html`**（ログイン不要・スタッフ用統合ページ）で、申込状況の確認・差し入れ完了報告（写真アップロード）・のし紙画像の作成を行う。

---

## 2. 関連ファイル

| ファイル | 役割 | ログイン |
|---|---|---|
| `jenne-gift-form.html` | ジェンヌ登録・観劇会への紐づけ・当番設定 | 必要 |
| `gift-letter-form.html` | 参加者向け 協力申込フォーム(メール必須) | 不要 |
| `gift-letter-list.html` | 申込集計・当番確認・CSV出力・御礼メール送信(写真アップロード付き) | 不要 |
| `nashi-print.html` | のし紙背景の登録・位置調整・**画像として書き出し**(印刷機能は廃止) | 必要 |
| `nashi-staff-print.html` | 担当者用 のし紙画像作成(背景選択のみ、位置調整も可) | 不要 |
| `gift-staff-dashboard.html` | 担当者向け統合ページ(申込状況・完了報告・のし紙画像作成を1画面に) | 不要 |
| `supabase-js.min.js` | 上記すべてが読み込む自ホスト版ライブラリ本体。**全ファイルと同じ階層に必須** | - |

---

## 3. Supabaseテーブル構成

| テーブル | 主な項目 | 備考 |
|---|---|---|
| `jennes` | name, period, photo_url, profile_url | ジェンヌのマスタ。使い回し前提 |
| `event_gifts` | event_id, jenne_id | 1回に複数人可(花組など) |
| `event_duty` | event_id(PK), duty_staff, duty_partner | 当番は1回1人。パートナーは固定ペアから自動計算 |
| `gift_letter_submissions` | event_id, jenne_id, name, period, name_display, has_companion, companion_name, companion_name_display, email | 住所欄は廃止済み(過去の名残でaddress列は残存するが未使用)。emailは必須項目 |
| `nashi_backgrounds` | image_url, overlay_top, overlay_font_size | 背景ごとに名前欄の位置・文字サイズを保存(overlay_rightは廃止・左右自動中央寄せに変更) |
| `event_gift_report` | event_id(PK), staff_name, receipt_photo_url, gift_photo_url, candy_photo_url | 差し入れ完了報告(写真3枚)。`gift-staff-dashboard.html`から担当者が登録 |
| `events.gift_invite_sent_at` | timestamptz | 案内メール自動送信済みフラグ(重複送信防止) |

---

## 4. 当番・パートナーのペア(固定)

お菓子購入・楽屋口届け担当と、金銭回収担当は固定ペアです。`jenne-gift-form.html`内の`DUTY_PAIRS`で管理しています。

- 原田 ⇔ 太田
- 伊庭 ⇔ 岩本
- 西中 ⇔ 水無瀬

当番を選ぶと、パートナーは自動で決まり、`event_gifts`ではなく`event_duty`テーブルに保存されます(観劇会1回につき1件)。

---

## 5. 組ごとのテーマカラー(参加者向けページ)

`gift-letter-form.html`・`gift-letter-list.html`は、対象観劇会の`troupe`列を見て自動でテーマカラーが変わります(`applyTroupeTheme()`)。

| 組 | イメージカラー |
|---|---|
| 花組 | ピンク系 |
| 月組 | ゴールド系 |
| 雪組 | グリーン系 |
| 星組 | ブルー系 |
| 宙組 | パープル系 |

**教訓(2026年9月)**：ボタンや背景色を新しく追加する際、CSS変数(`var(--green)`等)を使わず直接色コード(`#9ec089`など)を書いてしまうと、組カラーへの切り替えが反映されず「緑だけ残る」不具合になる。新しい色を足すときは必ず`var(--green)` / `var(--green-dark)` / `var(--green-soft)`を使うこと。

---

## 6. supabase-jsライブラリについて(重要・過去にハマった点)

- 当初は`unpkg`/`jsdelivr`からCDN経由で読み込んでいたが、**まりさんの回線環境ではCDN読み込みが不安定で「読み込み中」のまま固まる不具合が頻発**したため、supabase-js v2.0.0のUMDバンドルをダウンロードし、`supabase-js.min.js`として自サイトに同梱する方式に変更した。
- **ファイル名は厳密に`supabase-js.min.js`(ピリオド区切り)**。GitHubアップロード時に`supabase-js_min.js`や`supabase-js-min.js`に自動変換されてしまったことがあり、その場合404で全ページが動かなくなる。アップロード後は`https://takarazuka-mikokoro.vercel.app/supabase-js.min.js`に直接アクセスしてコードが表示されるか確認するのが確実。
- **anonキーは新形式(`sb_publishable_...`)ではなく、旧JWT形式(`eyJhbGci...`)を使うこと**。自ホストした古いバージョンのライブラリ(v2.0.0)は新形式キーとの組み合わせで認証がうまく通らず、ログイン済みのはずなのに書き込みがRLSエラーになる不具合が発生した。`admin-list.html`のように最新版ライブラリをCDNから読み込んでいるページは新形式キーで問題ない。
- **`gift-letter-form.html`の送信(INSERT)だけは、上記ライブラリを使わず生の`fetch()`でSupabase REST APIを直接叩く実装にしてある**。ライブラリ経由のINSERTだけがRLSエラーになる原因不明の不具合があり、fetchに切り替えたら解消した。他のページの読み取り(SELECT)はライブラリ経由のままで問題なし。

---

## 7. 観劇会選択のデフォルト値について

ページによって意図的に基準が違う。

- `jenne-gift-form.html` / `gift-letter-form.html` / `gift-letter-list.html` / `nashi-print.html` / `nashi-staff-print.html`：**今日から一番近い未来の回**をデフォルト選択(準備作業がまだの回を優先して表示するため)。
- `blog-report.html`：**直近の過去回**をデフォルト選択(終わった公演の報告を書くページのため)。

新しくページを作る際、この基準を混同すると「フォームで選んだ回と集計ページで見える回がズレる」不具合になるので注意。

---

## 8. 過去に発生した不具合と解決策

- **`gift-letter-list.html`・`nashi-print.html`で、実際にはデータが存在するのに一覧が0件と表示される不具合(解決済み)**：原因は`gift_letter_submissions`のSELECTポリシーが`to anon`のみで、ログインが必要なページ(`authenticated`ロールで通信)からは読み取れなかったこと。`gift_letter_submissions_select_authenticated`ポリシーを追加して解決。**新しくテーブルを作る際は、ログイン有無どちらのページからも読まれる可能性があるなら、SELECTポリシーを`anon`だけでなく`public`(両方)にしておくこと。**
- `gift_letter_submissions`の`address`列は使われなくなったが、DBには残したまま(実害なし)。
- **プレビュー(CSS)と実際の画像書き出し(Canvas)でのし紙の文字サイズが食い違う不具合(解決済み)**：iOS Safariには小さすぎるフォントを自動的に見やすく拡大表示する機能があり、CSSプレビューではそれで実際より大きく綺麗に見えていた。Canvas書き出しにはその補正が効かないため、入力値通りの小さいサイズで描画され、ズレが生じていた。**対策として、プレビュー自体もCanvasで描画する方式に変更**（8章参照）。

---

## 9. のし紙画像の作り方(印刷機能は廃止・画像書き出し方式)

- 当初はブラウザの印刷機能(`window.print()`)を使っていたが、**iOS Safariで印刷すると背景色がおかしくなる・余計な2ページ目ができる等の不具合が解消できなかった**ため、印刷機能は完全に廃止し、**Canvasで背景画像＋名前を1枚のJPG画像として合成し、その場でダウンロード/長押し保存**する方式に変更した。
- 用紙はA4横向き固定(B5・向き選択は廃止)。書き出し解像度は2079×1470px(A4横 297×210mmを約7px/mmで換算)。
- **プレビューと書き出しは全く同じCanvas描画ロジックを共有**しており、`PREVIEW_WIDTH`(700px)を基準にプレビュー用canvasを描画し、書き出し時は`EXPORT_WIDTH / PREVIEW_WIDTH`の倍率でフォントサイズをスケールして同じ見た目を高解像度で再現する。この仕組みのおかげでプレビューと実際の画像がズレない。
- 名前は1人1列の縦書き。列は右から左へ並び、全体を横方向に自動で中央寄せする(`drawVerticalNames()`)。名前ごとに列が分かれる仕組みなので、CSS版で使っていた「名前欄の高さ」設定は実質意味を持たず廃止した。
- 調整できるのは「上からの位置(%)」「文字サイズ(px、デフォルト20)」の2つのみ。
- 画像のダウンロードは、iOS Safariが`<a download>`属性を無視することが多いため、**「新しいタブで開く」ボタン→長押しで保存**という方式にしている。
- 新しく登録した背景の初期値(`overlay_top`・`overlay_font_size`)は、DBのカラムdefault値を使うので、UI側のデフォルト値を変える時はDB側のdefaultも合わせて変更すること(片方だけ変えるとズレる)。

---

## 10. 差し入れ案内メールの自動送信(Supabase Edge Function)

観劇会の**14日前**に、参加決定者へ自動でご案内メールを送信する仕組み。

- **Edge Function名**：`send-gift-invites`(Supabaseの「Edge Functions」に配置)
- **トリガー**：`pg_cron`で毎日 UTC 0:00(日本時間 朝9:00)に自動実行。`select cron.job`で登録状況を確認できる。
- **処理内容**：
  1. 「今日からちょうど14日後」が開催日の`events`を検索(`gift_invite_sent_at`が未設定のもののみ)
  2. `event_gifts`+`jennes`から差し入れ先の名前(複数可)を取得
  3. `registrations`で`status = '参加決定'`かつ`answers.email`が入っている人だけを対象に、重複を除いてメール送信
  4. 全員への送信に成功した場合のみ`events.gift_invite_sent_at`を更新(**一部失敗した場合はフラグを立てず、翌日以降に自動リトライされる**)
- **メール本文**：まりさん作成のHTMLテンプレートを使用。〔開催日〕〔公演名〕〔差し入れ先のお名前〕〔締切日〕を自動で埋め込む。差し入れ先が複数人いる場合は改行して全員分表示。締切日は「送信日(=開催14日前)の7日後」で自動計算。
- **送信方法**：EmailJSのREST API(`https://api.emailjs.com/api/v1.0/email/send`)を、Public KeyとPrivate Keyを使ってサーバーサイドから直接呼び出している(ブラウザを経由しないため、通常のEmailJS SDKではなくAPIを直接叩く形)。

### EmailJS側で必要な設定(重要・ハマりポイント)

- **EmailJSの「Account」→「Security」で「API access from non-browser environments」を有効にしておく必要がある**。これが無効だと`403 API access from non-browser environments is currently disabled`エラーになり、サーバーサイドからの送信が全て失敗する。
- **メールをHTMLとして正しく表示するには、テンプレートの本文(Content)を「Edit Content」で開き、`{{message}}`ではなく`{{{message}}}`(波かっこ3つ)にする必要がある**。波かっこ2つだと、送った内容がそのまま「文字列」として表示されてしまい、写真が画像として表示されず`<img src=...>`という文字列がそのまま出てしまう。
- Supabase Edge Functionには`SUPABASE_URL`・`SUPABASE_SERVICE_ROLE_KEY`が自動で環境変数として渡されるため、RLSを気にせず読み書きできる(service roleを使用)。EmailJSのPrivate Keyはコード内に直接埋め込んでいる(Supabase側にシークレット管理機能を使わず埋め込む方式。他の анонキー等と同程度のセキュリティレベル)。

### テスト方法

本番の参加者にメールを誤送信しないよう、テスト時は以下の手順を踏むこと。

1. `events`に開催日=今日から14日後の架空イベントを仮登録(`title`は「【テスト】」などと分かる名前に)
2. `participants`と`registrations`(`status='参加決定'`, `answers.email`にテスト用アドレス)を仮登録
3. SQLで`select net.http_post(url := 'https://[project].supabase.co/functions/v1/send-gift-invites', headers := '{"Content-Type": "application/json"}'::jsonb);`を実行して手動トリガー
4. `select status_code, content from net._http_response where id = [直前のrequest_id];`で結果を確認
5. テスト用に作った`events`・`registrations`・`participants`を必ず削除する

---

## 11. スタッフ用ダッシュボード周りの整理

同じような名前のページが複数あるので整理。

| ファイル名 | 内容 | 備考 |
|---|---|---|
| `staff-dashboard.html` | 観劇会全体のスタッフ用ダッシュボード(スケジュール確認・申込状況・食事・請求など) | 差し入れ関連システムとは別に元々存在していたページ |
| `gift-staff-dashboard.html` | **差し入れ・のし紙専用**のスタッフページ | 上記`staff-dashboard.html`の一番下からリンク |

`staff-dashboard.html`という名前は、差し入れ用に新規作成した際に一度誤って重複してしまったことがあるので、**新しくスタッフ向けページを作る際はファイル名の重複に要注意**。

`gift-staff-dashboard.html`の内容：
- ①申込状況(件数・人数・合計金額・お菓子購入担当者・金銭回収担当・楽屋口届け担当者(2名まとめ)・ジェンヌ別名簿)
- ②差し入れ完了報告(担当者名は「お菓子購入担当者」から自動入力、写真3枚をアップロードして`event_gift_report`に保存。**送信ボタンは置いていない**、御礼メールの確認・送信は`gift-letter-list.html`側でまりさんが行う)
- ③のし紙を作る(背景を選ぶだけ。位置調整は不可。まりさんが`nashi-print.html`で保存した位置設定を自動で使う)

---

## 12. 観劇会ごとの「やることリスト」への差し入れタスク追加

`task-tracker.html`(私用・要ログイン)のタスク一覧(`TASK_DEFS`)に、以下を追加済み。

| タスク | 期限(開催日からのオフセット) | 自動 |
|---|---|---|
| 差し入れフォーム作成 | -15日 | 手動 |
| 差し入れ募集開始 | -14日 | **自動**(10章のEdge Function) |
| 差し入れ集計 | -3日 | 手動 |

自動送信のタスクには「自動」バッジを表示し、期限欄も「あと◯日」ではなく実際の送信日(例:「9/20 自動送信」)を表示するようにしている。全タスクは観劇会ごとにグループ化せず、日付順(`due`でソート)にフラットに並ぶ仕様はそのまま。

`admin-event-detail.html`の「運営スケジュール」にも同様に以下を追加し、日付順に自動で並び替わるよう修正済み(以前は固定順で表示していたため、日付が前後する不具合があった)。

| 項目 | オフセット | 自動 |
|---|---|---|
| 差し入れ募集開始 | -14日 | **自動** |
| 差し入れ募集締切、集計開始 | -7日 | 手動 |
| 差し入れ報告 | +2日 | 手動 |
| ホームページで報告アップ | +3日 | 手動 |

※ スタッフ向けの`event-schedule-view.html`(staff-dashboard.htmlからリンク)は別ファイルのため、内容が同じかどうかは要確認。同じ内容を表示する場合は「自動」バッジを外すことも検討。

---

## 13. 旅行代理店の確定枚数と定員の自動連携

`admin-agency.html`で確定枚数(`agency_confirmations.confirmed_count`)を登録・修正すると、**同じ組・開催日・公演時間の`events.capacity`(定員)に自動反映**されるようにした(`syncCapacityToEvent()`)。

- マッチング条件：`troupe` + `event_date` + `show_time`が完全一致する`events`行
- 該当する観劇会が見つからない場合は、その旨をメッセージで表示し、定員は更新されない(登録自体は成功する)
- 複数件マッチした場合は全件に同じ値を反映する(通常は起こらない想定)
