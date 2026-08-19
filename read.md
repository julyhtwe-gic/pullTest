# GIC CRM MCP Server
# this is for June_Branch

Claude（Desktop / claude.ai / Claude Code）から GIC CRM を自然言語で操作するための
MCP サーバー。CRM の REST API（STG: `https://crm-stg-api.gicjp.org/api/v1`）を
JWT 認証付きでラップし、厳選した 28 ツールを公開する。

```
Claude UI ──(MCP)── このサーバー ──(REST + JWT)── GIC CRM API (Cloud Run)
```

## ツール一覧

| 分類 | ツール |
|------|--------|
| 認証 | `login_with_microsoft` `auth_status` |
| 基本 | `whoami` `list_master_data` `list_users` `get_user_activity_report` |
| 取引先 | `search_accounts` `get_account` `create_account` `update_account` |
| 担当者 | `search_contacts` `get_contact` `create_contact` `update_contact` |
| リード | `search_leads` `get_lead` `create_lead` `update_lead` `convert_lead` |
| 商談 | `search_opportunities` `get_opportunity` `create_opportunity` `update_opportunity` |
| 活動 | `list_activities` `log_activity` |
| 分析 | `get_dashboard_summary` `get_pipeline_report` `list_forecasts` `get_forecast_summary` `upsert_forecast` |

削除系（DELETE）ツールは安全のため意図的に公開していない。

## 認証モード

| モード | 設定 | 仕組み |
|--------|------|--------|
| **Microsoft SSO（推奨）** | `GIC_CRM_AUTH=azure` | 初回のみ `login_with_microsoft` ツール → ブラウザでデバイスコードサインイン。リフレッシュトークンを `%LOCALAPPDATA%\gic-crm-mcp\msal_cache.json` にキャッシュし、以降は自動更新。ID トークンを `POST /auth/login/azure` で CRM JWT に交換する。パスワード不要（SSO 専用アカウントでも使える）。 |
| パスワード | `GIC_CRM_EMAIL` + `GIC_CRM_PASSWORD` | `POST /auth/login`。CRM 側にパスワードが設定されているアカウントのみ。 |

`GIC_CRM_AUTH` 未設定時は EMAIL/PASSWORD があればパスワード、なければ azure。
どちらのモードでも**そのユーザーとして全操作が実行される**（オーナー権限チェック適用）。

注意: SSO モードはデバイスコードフローを使うため、Azure ポータルのアプリ登録
（CRM フロントエンドと同じ `a276073e-…`）で「パブリック クライアント フローを許可する」
が有効である必要がある。無効の場合 `login_with_microsoft` がその旨のエラーを返す。

## 環境変数

| 変数 | 説明 |
|------|------|
| `GIC_CRM_API_URL` | CRM API ベースURL（既定: STG） |
| `GIC_CRM_AUTH` | `azure`（SSO）または `password` |
| `GIC_CRM_EMAIL` / `GIC_CRM_PASSWORD` | パスワードモード時のログイン情報 |
| `GIC_CRM_AZURE_TENANT_ID` / `GIC_CRM_AZURE_CLIENT_ID` | SSO モードの上書き用（既定は STG の値を内蔵） |
| `GIC_CRM_TOKEN_CACHE` | MSAL トークンキャッシュの保存先の上書き |
| `MCP_TRANSPORT` | `stdio`（既定）または `http` |
| `MCP_AUTH_TOKEN` | HTTP モード時の静的 Bearer トークン（未設定なら認証なし・警告ログ） |
| `PORT` | HTTP モードのポート（既定 8080。Cloud Run が自動設定） |

## 使い方 1: Claude Desktop（ローカル, stdio）

```bash
cd mcp-server
pip install -r requirements.txt
```

`claude_desktop_config.example.json` を参考に、Claude Desktop の設定ファイル
（Windows: `%APPDATA%\Claude\claude_desktop_config.json`）へ追記して再起動:

```json
{
  "mcpServers": {
    "gic-crm": {
      "command": "python",
      "args": ["C:\\path\\to\\mcp-server\\server.py"],
      "env": {
        "GIC_CRM_AUTH": "azure"
      }
    }
  }
}
```

初回は Claude に「CRMにMicrosoftでログインして」と頼むと URL とコードが表示される。
ブラウザでサインイン後、すべてのツールが使えるようになる（`auth_status` で確認可）。

※ 設定ファイルは **Claude Desktop を終了してから**編集すること。起動中に編集すると
アプリが古い設定で上書きし戻すことがある。

## 使い方 2: claude.ai / Desktop の Connector（リモート, HTTP）

### Cloud Run へデプロイ

```bash
cd mcp-server
gcloud run deploy gic-crm-mcp \
  --source . \
  --region asia-northeast1 \
  --allow-unauthenticated \
  --set-env-vars "MCP_TRANSPORT=http,GIC_CRM_API_URL=https://crm-stg-api.gicjp.org/api/v1" \
  --set-secrets "GIC_CRM_EMAIL=gic-crm-mcp-email:latest,GIC_CRM_PASSWORD=gic-crm-mcp-password:latest"
```

MCP エンドポイントは `https://<service-url>/mcp`。ヘルスチェックは `/health`。

### Claude 側の設定

- **claude.ai (web/mobile)**: Settings → Connectors → *Add custom connector* に
  `https://<service-url>/mcp` を登録（Pro/Max/Team プランが必要）。
- **Claude Desktop**: 同じく Settings → Connectors から登録可能。
- **Claude Code**:
  ```bash
  claude mcp add --transport http gic-crm https://<service-url>/mcp \
    --header "Authorization: Bearer <MCP_AUTH_TOKEN>"
  ```

### セキュリティ上の注意（重要）

claude.ai の custom connector は静的トークンヘッダーを送れない（OAuth または
認証なしのみ）。STG 用に公開する場合の現実的な選択肢:

1. **Claude Code / API 経由のみ** → `MCP_AUTH_TOKEN` を設定（推奨）
2. **claude.ai から使う** → `MCP_AUTH_TOKEN` なしで公開し、
   - Cloud Run の URL を推測困難なサービス名にする
   - 操作権限が最小の専用 CRM ユーザー（role_type=sales）を `GIC_CRM_EMAIL` に使う
   - 本番 API には向けない（STG 限定）
3. 将来的に OAuth 対応（MCP Authorization 仕様）を実装すれば claude.ai でも安全に利用可能

## 動作確認

```bash
# stdio モードでツール一覧が出るか（MCP Inspector）
npx @modelcontextprotocol/inspector python server.py

# HTTP モード起動
set MCP_TRANSPORT=http && python server.py
curl http://localhost:8080/health
```

## 仕様メモ（CRM API の挙動）

- 商談の `status` は直接設定不可。`stage_id` から自動導出（受注→won、失注→lost、
  保留→on_hold、他→open）。失注には `loss_reason_id` 必須。
- 商談の作成・更新時に `expected_order_date` の年月で売上予測（expected/confirmed）が
  自動同期される。
- `log_activity` で `opportunity_id` を渡すと商談の `last_activity_at` /
  `next_action` / `next_action_due_date` が自動更新される。
- リードの `status=closed` には `close_reason`（converted/not_converted）必須。
  商談化済みリードのステータスは変更不可。
- 書き込み系はレコードオーナー本人か admin/executive のみ（403）。
- `/auth/login` は 5回/分のレート制限があるため、トークンは期限までキャッシュされる。
