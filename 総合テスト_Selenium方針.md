# 総合テストにおけるSelenium方針

作成日: 2026-06-23

## 実装状況（2026-06-23）

`/e2e` ディレクトリに、Docker完結型のSelenium総合テスト環境を構築した。

- `e2e/docker-compose.yml`: MySQL → backend → frontend(nginx) → Selenium(`selenium/standalone-chrome`) を一括起動する構成。
- `e2e/frontend.Dockerfile` + `e2e/nginx.conf`: frontendは本番（Vercelの`vercel.json`によるrewrite）と同等に、nginxで `/api/` を backend へリバースプロキシする構成で配信する。
- `e2e/pom.xml` + `e2e/src/test/java/com/example/e2e/`: Java + JUnit5 + selenium-java（独立したMavenプロジェクト。backendとは別モジュール）。`SeleniumTestBase` が `RemoteWebDriver` のセットアップ/破棄を共通化し、各テストクラスがこれを継承する。
- `e2e/run.sh`: `docker compose up --build --wait` → `./mvnw test` → `docker compose down -v` を一括実行するエントリーポイント。ローカル・CIのどちらでも `cd e2e && ./run.sh` 一発で再現できる。
- 初回シナリオとして `LoginE2ETest`（正しい認証情報でのログイン成功／誤った認証情報でのエラー表示）を実装し、実際にブラウザ経由で動作確認済み。

## テストシナリオの拡充（2026-06-24）

ログインのみだった初回実装から、主要な業務フロー（事業所管理・スタッフ管理・利用者管理・送付物管理）のCRUDおよびバリデーション・権限系シナリオまで拡充した。`./run.sh` で全クラス・全件成功を確認済み（計50件）。

| テストクラス | 件数 | 主なシナリオ |
|---|---|---|
| `LoginE2ETest` | 5 | ログイン成功／失敗／無効化アカウントでのログイン拒否など |
| `PasswordChangeE2ETest` | 4 | 初回強制パスワード変更フロー |
| `OfficeManagementE2ETest` | 9 | 事業所の新規登録（全項目／名称のみ）、必須・郵便番号・電話番号のバリデーション、編集、有効化／無効化、名称検索によるフィルタ |
| `StaffManagementE2ETest` | 9 | スタッフ登録とバリデーション（必須・メール形式・パスワード強度・ロール必須・メール重複409）、編集（パスワード欄が編集時は非表示であることの確認込み）、自分自身の行に無効化ボタンが出ないこと、ADMINが最後の1人でない場合のみ無効化可能なことの動的カウント検証 |
| `UserManagementE2ETest` | 13 | 利用者登録・編集・検索フィルタ・無効化／有効化と一覧表示の出し分け、利用者詳細での事業所紐付け追加／解除 |
| `MailSendE2ETest` | 10 | 単一／複数事業所×送付種別での送付物作成、STEPごとの「次へ」ボタンの有効化制御、利用者検索フィルタ、送付済み一括処理（メモ付き・500文字超のバリデーション）、ステータスフィルタ変更時の選択クリア、事業所・利用者での履歴フィルタ、CSV出力 |

**拡充の過程で見つけた問題と対応（いずれもテストコード側の不具合。本番のフロントエンド／バックエンドには影響しないことを確認済み）:**
- DADSの `Checkbox` コンポーネントは `{...rest}` を素通しするだけで、呼び出し側（`CreatePage.tsx` STEP2の送付種別選択）が `value` を明示的に渡していないため、`input[value=...]` セレクタでは選択できなかった → ラベルのテキスト一致で要素を選ぶように変更。
- テストデータの「ふりがな」固定文字列に長音符 `ー` を使っていたが、`userSchema.ts` のひらがな正規表現バリデーション（`/^[ぁ-ん\s]*$/`）の範囲外で、フォーム送信が静かに失敗していた → `ー` を使わない固定文字列に変更。
- **XPathの `contains(text(), x)` は要素の最初のテキストノードしか見ない**という仕様の罠。`🏢 {office.name}` のようにJSXの文字列リテラルと埋め込み式が隣接していると、描画後はリテラル部分と式部分が別々のテキストノードになり、`text()` ではリテラル部分（絵文字など）しか評価されず、本来あるはずの要素が「見つからない」と誤判定される。`MailSendByOfficePage`／`MailSendHistoryPage`の一覧表示確認、`UserDetailPage.isOfficeLinked`（紐付き事業所の表示確認）で発生し、いずれも子孫テキストを全て連結した文字列値を見る `contains(., x)` に修正。`UserDetailPage` 側はこのバグにより `isOfficeLinked` が常にfalseを返す決定論的な不具合で、事業所紐付け追加／解除のテストが必ずタイムアウトしていた。
- **新規作成直後の一覧初回表示が古いデータのまま止まる現象**: スタッフ／事業所／利用者の一覧画面で、作成・更新直後にその場の一覧（React Queryのキャッシュ）が新しいデータを反映しないまま表示されることがあった（バックエンドのデータ自体はAPIで直接確認すると正しく存在しており、フロントエンド側の取得タイミングに起因すると推測。根本原因は未特定）。対応として、行が見つからない場合に強制リロードして再取得するリトライを各Page Objectの `waitForRow`／`isRowVisible` に追加。ただし強制リロードは検索ボックスの入力内容や「すべて表示」選択などの画面側state（URLに乗らないフィルタ）も一緒に消してしまうため、それらが消えることで「絞り込みで除外されているはずの行が再表示されて誤って見つかる」という二次的な誤判定を引き起こした（`OfficeListPage`／`UserListPage`の検索フィルタテストで実際に発生）。リトライ時に直前の検索キーワード・フィルタ選択を再適用することで対応。一方 `MailSendHistoryPage` の絞り込みはコンポーネントstateで保持されており同様の対応ができないため、ここでは強制リロードによるリトライを行わない方針とした。

**導入過程で見つけた問題と対応:**
- `frontend` に `.dockerignore` がなく `node_modules` 等が毎回Docker daemonへ転送され、ビルドコンテキスト転送だけで数分かかっていた → `frontend/.dockerignore` を追加。
- backendの `cors.allowed-origins` に compose 内フロントエンドのオリジン（`http://frontend`）が含まれておらず、ブラウザからのログインリクエストが403（`Invalid CORS request`）になっていた → `docker-compose.yml` の backend サービスに `CORS_ALLOWED_ORIGINS=http://frontend` を追加。
- **実バグを検出**: `frontend/src/api/client.ts` のaxiosレスポンスインターセプターが、**ログインAPI自体の401（パスワード誤り）でも** `window.location.href = '/login'` の強制リロードを起こしており、`LoginForm.tsx` が表示するはずの「メールアドレスまたはパスワードが正しくありません」が画面に出ないまま無言リロードされる不具合があった。ログインエンドポイント（`/auth/login`）への401はリダイレクト対象から除外するよう修正済み。

## 背景

[結合テスト_データベース方針案.md](./結合テスト_データベース方針案.md) で導入したTestcontainers結合テスト（`*IT`）は、Spring経由でアプリケーション層〜実MySQLの整合性を検証するものであり、**実際にブラウザでUIを操作したときの挙動**（フォーム入力、画面遷移、エラー表示など）は検証できていなかった。

今回のSeleniumログインテストの初回実行で、上記の通り「ログイン失敗時にエラーメッセージが表示されない」という結合テストやUnitテストでは検出できない不具合が実際に見つかっており、フロントエンド〜backend〜MySQLを通しでブラウザから検証する総合テストの必要性が裏付けられた。

本番構成:
- フロントエンド: Vercel（静的ホスティング、`vercel.json` で `/api/*` をRailwayへrewrite）
- バックエンド: Railway上のSpring Boot（MySQL接続）

## 検討した実行方式

| 方式 | 概要 | 評価 |
|---|---|---|
| **Docker完結型（採用）** | frontend/backend/MySQL/Seleniumを全てdocker-composeでコンテナ化し、ローカル・CIどちらでも同一環境を再現する | ◎ ローカルとCIの差異がない／◎ デプロイ前のブランチでも実行できる／△ frontend用Dockerfileを別途用意する必要がある（本番はVercel/Railwayで稼働するため） |
| デプロイ済み環境に対して実行 | Vercelのプレビューデプロイ＋Railwayに対してSeleniumを向け、Dockerはブラウザ（Selenium）のみコンテナ化する | ◎ 本番に最も近い／✗ デプロイ完了を待つ必要があり、ローカルで素早く回せない／✗ プレビューデプロイがない変更（フロント未変更のブランチ等）では使えない場合がある |

**結論**: ローカル開発中に頻繁に実行できることを優先し、Docker完結型を採用した。デプロイ済み環境に対するスモークテストは別途の課題とする（後述の未決事項）。

## 採用した構成

```
e2e/
├── docker-compose.yml      # mysql / backend / frontend / selenium の4サービス
├── frontend.Dockerfile     # frontendをビルドしnginxで配信するためのDockerfile（frontendリポジトリには置かない）
├── nginx.conf              # /api/ を backend サービスへリバースプロキシ
├── pom.xml                 # Selenium総合テスト用の独立Mavenプロジェクト
├── run.sh                  # up --build --wait → mvn test → down -v を一括実行
└── src/test/java/com/example/e2e/
    ├── SeleniumTestBase.java          # RemoteWebDriverの生成・破棄を共通化
    ├── LoginE2ETest.java              # ログイン成功／失敗シナリオ
    ├── PasswordChangeE2ETest.java     # 初回強制パスワード変更
    ├── OfficeManagementE2ETest.java   # 事業所のCRUD・バリデーション・検索
    ├── StaffManagementE2ETest.java    # スタッフのCRUD・バリデーション・権限系
    ├── UserManagementE2ETest.java     # 利用者のCRUD・検索・事業所紐付け
    ├── MailSendE2ETest.java           # 送付物の作成・一括送付・履歴・CSV出力
    ├── pages/                         # Page Objectパターンの画面ごとのクラス
    ├── pages/components/              # ConfirmModal等、画面横断の共通コンポーネント
    └── support/                       # LoginHelper・TestDataFactory等のテストヘルパー
```

**ネットワークの考え方（要注意点）:**
- Seleniumが操作するブラウザは `selenium` コンテナの中で動くため、`driver.get(...)` に渡すURLは **docker-compose内部のサービス名**（`http://frontend`）で解決する。ホストにポートマッピングした `http://localhost:8090` ではブラウザ（コンテナ内プロセス）からは届かない。
- 一方、テストを実行するJavaプロセス自身（Mavenを実行するホスト側）からSeleniumサーバーへコマンドを送るための接続先（`RemoteWebDriver`に渡すURL）は、ホストにポートマッピングされた `http://localhost:4444` を使う。
- この2つのURLは役割が異なるため、`SeleniumTestBase` では `SELENIUM_REMOTE_URL`（Selenium操作用）と `APP_URL`（ブラウザのナビゲーション先）を別の環境変数として分離している。

**frontendのビルド方式:**
- 本番はVercelの `vercel.json` rewriteで `/api/*` をRailwayへ転送する「同一オリジン」構成になっている。総合テスト環境でもこれを再現するため、frontendは `VITE_USE_MOCK=false` でビルドし、nginxの `location /api/ { proxy_pass http://backend:8080/api/; }` で同一オリジンのAPIプロキシを構成している。これによりCORSを意識せず本番と同条件でテストできる（後述のCORS設定は、ブラウザがOriginヘッダーを送る場合への保険として必要だった）。

## 実行方法

```bash
cd e2e
./run.sh
```

個別にコンテナを起動したまま繰り返しテストしたい場合（デバッグ用途）:

```bash
cd e2e
docker compose up -d --build --wait
./mvnw test          # 何度でも再実行可能
docker compose down -v
```

## 今後の課題（未決事項）

- CI（GitHub Actions）への組み込み: backendの結合テスト（`ci.yml`）とは別ジョブとして追加するか、`workflow_dispatch`/nightly等で実行頻度を分けるかは未決定。
- デプロイ済み環境（Vercelプレビュー／Railway）に対するスモークテストの要否。
- 「新規作成直後の一覧初回表示が古いデータのまま止まる現象」（前述）の根本原因は未特定。強制リロードによるリトライで実用上は安定して全件成功しているが、フロントエンド側（React Queryのキャッシュ無効化タイミング等）の調査は今後の課題として残る。
