# frontend

このフォルダは、Google Apps Script の画面ではなく、GitHub Pages、Vercel、Firebase Hosting などに置く外部LIFFフロントエンドです。

## 現在の構成

```text
GitHub Pages index.html
→ LIFF SDK
→ LINE userId取得
→ JSONPで Apps Script /exec API
→ Code.gs の saveCheckin(checkinData)
→ Googleスプレッドシート
```

役割は次のように分けています。

- GitHub Pages `index.html`: LINE userId取得、画面表示、チェックインボタン操作
- Apps Script `Code.gs`: 入力検証、同日重複防止、特典二重使用防止、保存API
- Googleスプレッドシート: 簡易DB、イベントログ

## なぜApps Script画面をやめたか

Apps Script HtmlService の画面では、実際の実行URLが `script.googleusercontent.com` になることがありました。

LIFFでは、`liff.init()` を実行するURLが LINE Developers に設定したLIFFエンドポイントURLと一致、またはその配下である必要があります。

Apps Script画面ではこの条件を満たしにくく、`liff.init()` が完了しませんでした。

そのため、画面はGitHub Pagesへ移し、Apps Scriptは保存APIに役割を絞りました。

この分離により、将来フロントだけを作り替えたり、保存APIだけを別バックエンドへ移したりしやすくなります。

## JSONPを使う理由

GitHub Pages のような外部HTMLから Apps Script を呼ぶ場合、`google.script.run` は使えません。

`google.script.run` は Apps Script の HtmlService 内専用です。

通常の `fetch` POST は CORS で詰まる可能性があります。

そのため、MVPでは JSONP GET で疎通を優先しています。

JSONP GETで保存する方式は、本格運用向けとしては強くありません。

本番化する場合は、次のような方式を検討します。

- ID token または access token のサーバー側検証
- Cloud Run
- Firebase Functions
- Cloudflare Workers
- Supabase Edge Functions
- その他の通常APIバックエンド

## 現在できること

- LINE内でLIFF URLを開く
- LIFF SDKで LINE userId を取得する
- チェックインボタンで Apps Script JSONP API を呼ぶ
- Googleスプレッドシートに `customer_code = U...` を保存する
- `memo` に `GitHub Pages LIFFフロントからのチェックイン保存テスト。` を保存する
- サーバー側で同日チェックイン重複を防ぐ
- API ping で疎通確認する

## まだできないこと

- +3日特典判定の外部フロント移植
- 特典使用保存
- localStorage の本格移植
- サーバー側の LINE ID token 検証
- 本番用プライバシー説明
- デバッグUI非表示化

## 設定する場所

Apps Script側の Webアプリ URL `/exec` を、`index.html` の次の設定へ入れてください。

```js
const APPS_SCRIPT_API_URL = 'APPS_SCRIPT_API_URL';
```

`APPS_SCRIPT_API_URL` の部分を、実際の `/exec` URL に差し替えます。

Apps Script を新規デプロイしてURLが変わった場合は、ここを新しいURLへ差し替えます。

同じデプロイを新バージョンで更新しただけなら、URLは基本的に維持されます。

## Apps Script Webアプリの公開設定

GitHub Pages から呼べるように、Apps Script Webアプリのデプロイ設定を確認してください。

- 実行ユーザー: 自分
- アクセスできるユーザー: 全員

公開範囲が狭いと、GitHub Pages からJSONPで呼べず、API接続に失敗します。

## 確認ポイント

画面で次の状態を確認します。

- `liff.init 成功`
- `LINE userId 取得成功`
- `storage key 予定値` が `checkinHistory:U` で始まる
- `API ping response.ok: true`
- チェックインボタンを押した後に `保存成功` と表示される

スプレッドシート側では、`customer_code` に `U` で始まる LINE userId が入っていれば成功です。

## 同日重複防止

同じLINEアカウントで同じ日に2回チェックインすると、サーバー側同日重複防止により `ALREADY_CHECKED_IN_TODAY` になります。

その場合、画面には「本日はすでにチェックイン済みです」と表示されます。

これは通信エラーではなく、業務上の正常ブロックです。

## 次の魔改造候補

- +3日判定を `index.html` へ移す
- `benefit_used` 送信を追加する
- localStorageをフルの LINE userId キーで使う
- APIレスポンスで前回チェックイン状態を返す
- スプレッドシート列に `store_id` や `campaign_id` を追加する
- `benefit_type` を追加して特典種別を保存する
- ID token検証を `Code.gs` へ追加する
- JSONPから別API方式へ移行する

## テスト手順

1. `current/app-script/Code.gs` をApps Scriptへ反映する
2. Apps Scriptを新バージョンでデプロイする
3. WebアプリURL `/exec` をコピーする
4. `current/github-pages/index.html` の `APPS_SCRIPT_API_URL` に `/exec` URL を入れる
5. GitHubへcommitする
6. GitHub Pagesの反映を待つ
7. LINE DevelopersのLIFFエンドポイントURLはGitHub Pages URLのままにする
8. LINEで `https://liff.line.me/2010326382-0mJkoibO` を開く
9. LINE userId 取得成功を確認する
10. API ping成功をログで確認する
11. チェックインボタンを押す
12. スプレッドシートに `customer_code = U...` の行が増えるか確認する
13. もう一度押した場合、`ALREADY_CHECKED_IN_TODAY` になるか確認する

## 失敗時に見る場所

- GitHub Pages画面内のログ
- Apps Scriptの実行数
- `APPS_SCRIPT_API_URL` が最新の `/exec` URLか
- Apps Script Webアプリの公開範囲が「全員」になっているか
- スプレッドシートの `checkins` タブが存在するか

## 本番化前の注意

LINE userId は個人識別子に近い情報です。

本番公開前には、保存目的、保存項目、利用期間、削除対応を説明できるようにします。

MVP段階では、小規模検証として扱い、デバッグUIや画面内ログは本番ユーザーに見せない形へ整理します。
