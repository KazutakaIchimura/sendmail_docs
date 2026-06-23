# 総合テストにおけるSelenium方針

作成日: 2026-06-23

## 実装状況（2026-06-23）

`/e2e` ディレクトリに、Docker完結型のSelenium総合テスト環境を構築した。

- `e2e/docker-compose.yml`: MySQL → backend → frontend(nginx) → Selenium(`selenium/standalone-chrome`) を一括起動する構成。
- `e2e/frontend.Dockerfile` + `e2e/nginx.conf`: frontendは本番（Vercelの`vercel.json`によるrewrite）と同等に、nginxで `/api/` を backend へリバースプロキシする構成で配信する。
- `e2e/pom.xml` + `e2e/src/test/java/com/example/e2e/`: Java + JUnit5 + selenium-java（独立したMavenプロジェクト。backendとは別モジュール）。`SeleniumTestBase` が `RemoteWebDriver` のセットアップ/破棄を共通化し、各テストクラスがこれを継承する。
- `e2e/run.sh`: `docker compose up --build --wait` → `./mvnw test` → `docker compose down -v` を一括実行するエントリーポイント。ローカル・CIのどちらでも `cd e2e && ./run.sh` 一発で再現できる。
- 初回シナリオとして `LoginE2ETest`（正しい認証情報でのログイン成功／誤った認証情報でのエラー表示）を実装し、実際にブラウザ経由で動作確認済み。

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
    ├── SeleniumTestBase.java   # RemoteWebDriverの生成・破棄を共通化
    └── LoginE2ETest.java       # ログイン成功／失敗シナリオ
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
- テストシナリオの拡充: 現状はログインのみ。送付レコードのCRUD等、主要な業務フローへの拡張が必要。
- デプロイ済み環境（Vercelプレビュー／Railway）に対するスモークテストの要否。
