# frontend

このフォルダは、Google Apps Script 画面ではなく、GitHub Pages、Vercel、Firebase Hosting などに置く外部LIFFフロントエンドです。

## 目的

まず確認することは、LIFF SDK の初期化と LINE userId の取得です。

現在の `frontend/index.html` は、スプレッドシート保存APIにはまだ接続しません。チェックインボタンを押しても保存は行わず、次の段階で Apps Script API へ保存する予定だと表示するだけです。

## GitHub Pagesで公開する想定

1. この `frontend/index.html` を GitHub Pages などで公開します。
2. 公開されたURLを控えます。
3. LINE Developers の LIFF エンドポイントURLを、その公開URLに変更します。
4. LINEトークで `https://liff.line.me/2010326382-0mJkoibO` を開きます。

## 確認ポイント

画面に次の状態が出ることを確認します。

- `liff.init 成功`
- `liff.isInClient()` の結果
- `liff.isLoggedIn()` の結果
- `profile.userId` の先頭6文字
- `storage key 予定値`

成功時は、`storage key 予定値` が `checkinHistory:U` で始まる形になります。

## 次の段階

保存API接続は次の段階で行います。

次に作る予定の流れは次の通りです。

1. Apps Script 側に外部フロントエンドから呼べる保存APIを用意する
2. `frontend/index.html` から Apps Script API へ `customer_code` とチェックイン情報を送る
3. Googleスプレッドシートにチェックイン履歴を保存する
4. 本格運用前に ID token または access token のサーバー側検証を検討する
