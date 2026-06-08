# GitHub Pages LIFF フロントエンド

このフォルダは、Google Apps Script の画面ではなく、GitHub Pages、Vercel、Firebase Hosting などに置く外部 LIFF フロントエンドです。

## 現在の構成

```text
GitHub Pages の index.html
-> LIFF SDK
-> LINE userId 取得
-> JSONP で Apps Script Webアプリ /exec API を呼ぶ
-> Code.gs の doGet(e)
-> saveCheckin(checkinData)
-> Googleスプレッドシートへ appendRow
```

役割は次のように分けています。

- GitHub Pages `index.html`: LINE userId 取得、画面表示、ユーザー操作
- Apps Script `Code.gs`: 入力検証、状態取得、同日重複防止、特典二重使用防止、スプレッドシート保存
- Googleスプレッドシート: 簡易DB、イベントログ

## なぜ Apps Script 画面をやめたか

Apps Script HtmlService の画面では、実際の実行URLが `script.googleusercontent.com` 側になることがあります。
LIFF では、`liff.init()` を実行するURLが LINE Developers で設定したエンドポイントURLと一致、またはその配下である必要があります。

この条件と Apps Script 画面の実行URLが合わず、`liff.init()` が完了しない問題が出ました。
そのため、画面は GitHub Pages へ移し、Apps Script は保存APIとして使う構成にしています。

この分離により、画面だけを後から作り替えたり、保存APIだけを別バックエンドへ移行したりしやすくなります。

## JSONP を使う理由

GitHub Pages のような外部HTMLから Apps Script を呼ぶ場合、`google.script.run` は使えません。
`google.script.run` は Apps Script HtmlService 内だけで使える仕組みです。

通常の `fetch` POST は CORS で詰まる可能性があります。
MVPでは、まず疎通を優先して JSONP GET 方式を使っています。

JSONP GET で保存する方式は、本格運用向けとしては強くありません。
本番化する場合は、次のような方式を検討します。

- ID token または access token のサーバー側検証
- Cloud Run
- Firebase Functions
- Cloudflare Workers
- Supabase Edge Functions
- Apps Script 以外の通常APIバックエンド

## v0.1 でできること

- LINE内で LIFF URL を開く
- LIFF SDK で LINE userId を取得する
- チェックインボタンで Apps Script JSONP API へ保存する
- Googleスプレッドシートに `customer_code` として LINE userId を保存する
- API ping で Apps Script API の疎通を確認する
- サーバー側で同日チェックイン重複を防ぐ

## v0.2 で追加したこと

v0.2では、localStorageを正本にせず、スプレッドシート上のイベントログから現在状態を復元する方針に進めています。

- `api=getCustomerState` を追加
- 起動時に LINE userId ごとのサーバー状態を取得
- +3日特典判定をサーバー側で実行
- チェックイン保存時の `benefit_status` と `memo` をサーバー側で決定
- 今日のチェックインが特典対象なら、特典使用ボタンを表示
- 特典使用を `action = benefit_used` のイベントとして追記
- 同じ `checkin_id` の特典二重使用をサーバー側で防止

設計思想として、フロントエンドは表示と操作を担当し、業務ルールの正本は Apps Script 側に寄せています。

## v0.3 で追加したこと

v0.3では、`APP_MODE` を追加し、本番用UIと開発用UIを切り替えられるようにしました。

- `APP_MODE = 'customer'` では、お客様向け本番表示になります
- `APP_MODE = 'dev'` では、LIFFログ、APIログ、state確認用のデバッグカードを表示します
- customerモードでは、userId、APIレスポンス、payload、checkin_id などの開発情報を画面に出しません
- devモードでは、LINE内ブラウザで原因調査しやすいように従来通り画面内ログを確認できます
- 保存、状態取得、+3日判定、特典使用保存のロジック自体は v0.2 から変えていません

本番では `customer` を使います。
開発や検証では `dev` に切り替えます。

`APP_MODE` は画面表示だけの切り替えです。
サーバー側の同日重複防止、特典二重使用防止、保存処理には影響しません。

## まだできないこと

- localStorageによる画面補助
- LINE ID token または access token のサーバー側検証
- 本番用プライバシー説明
- 店舗ID、キャンペーンID、特典種別の保存
- デザイン最終調整
- JSONP以外のより安全なAPI方式への移行

## 設定する場所

Apps Script 側の Webアプリ URL `/exec` を、`index.html` の次の設定へ入れます。

```js
const APPS_SCRIPT_API_URL = 'APPS_SCRIPT_API_URL';
```

実際のファイルでは、この値を Apps Script Webアプリの `/exec` URL に差し替えます。
Apps Script を新規デプロイしてURLが変わった場合は、ここも差し替えます。
同じデプロイを新バージョンで更新するだけなら、URLは基本的に維持されます。

## Apps Script Webアプリの公開設定

GitHub Pages から呼べるように、Apps Script Webアプリのデプロイ設定を確認します。

- 実行ユーザー: 自分
- アクセスできるユーザー: 全員

公開範囲が狭いと、GitHub Pages から JSONP API を呼べず、API接続に失敗します。

## 確認ポイント

画面では次の状態を確認します。

- devモードでは `liff.init 成功`
- devモードでは `LINE userId 取得成功`
- devモードでは `storage key 予定値` が `checkinHistory:U` で始まる
- devモードでは `API ping response.ok: true`
- devモードでは `getCustomerState response.ok: true`
- customerモードではデバッグカードが表示されない
- customerモードではuserIdやAPIログが画面に出ない
- チェックイン後に `保存成功` と表示される

スプレッドシートでは、`customer_code` に `U` で始まる LINE userId が入っていれば成功です。

## 同日重複防止

同じ LINE userId で同じ日に2回チェックインすると、サーバー側同日重複防止により `ALREADY_CHECKED_IN_TODAY` になります。
これは通信エラーではなく、業務上の正常ブロックです。

localStorageを消しても、スプレッドシートに今日の `checkin` 行が残っていれば、サーバー側で止まります。

## 特典使用防止

特典使用は、チェックイン行を上書きせず、`action = benefit_used` の別イベントとして追記します。
`checkin_id` により、どのチェックインに対する特典使用かを紐づけます。

同じ `customer_code` と同じ `checkin_id` の `benefit_used` 行がすでにある場合、Code.gs 側で `BENEFIT_ALREADY_USED` として止めます。

## 次の魔改造候補

- localStorageをフルの LINE userId キーで画面補助に使う
- APIレスポンスで前回チェックイン詳細をさらに返す
- スプレッドシート列に `store_id` や `campaign_id` を追加する
- `benefit_type` を追加して特典種別を保存する
- ID token検証を `Code.gs` に追加する
- JSONPから別API方式へ移行する
- customerモードの文言とレイアウトをさらに店頭向けに整える

## テスト手順

### devモード確認

1. `current/github-pages/index.html` の `APP_MODE` を `dev` にする
2. GitHubへcommitする
3. GitHub Pagesの反映を待つ
4. LINEで `https://liff.line.me/2010326382-0mJkoibO` を開く
5. デバッグカードが表示されるか確認する
6. `liff.init`、API ping、getCustomerState のログが見えるか確認する
7. チェックインと特典使用が従来通り動くか確認する

### customerモード確認

1. `current/github-pages/index.html` の `APP_MODE` を `customer` に戻す
2. GitHubへcommitする
3. GitHub Pagesの反映を待つ
4. LINEで `https://liff.line.me/2010326382-0mJkoibO` を開く
5. デバッグカードが非表示になっているか確認する
6. userIdやAPIログが画面に出ないことを確認する
7. チェックインと特典使用が従来通り動くか確認する
8. お客様向け文言だけが表示されることを確認する

成功条件は、devでは開発ログが見え、customerでは開発ログが見えず、どちらのモードでも保存、状態取得、特典使用が壊れていないことです。

## 失敗時に見る場所

- GitHub Pages 画面内のログ
- Apps Script の実行数
- `APPS_SCRIPT_API_URL` が最新の `/exec` URLか
- Apps Script Webアプリの公開範囲が「全員」になっているか
- スプレッドシートに `checkins` タブがあるか

## 本番化前の注意

LINE userId は個人識別子に近い情報です。
本番公開前には、保存目的、保存項目、利用期間、削除対応を説明できるようにします。

MVP段階では小規模検証として扱い、デバッグUIや画面内ログは本番ユーザーに見せない形へ整理します。
