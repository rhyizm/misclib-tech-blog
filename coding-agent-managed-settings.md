---
title: 'コーディングエージェントの既定値は安全側とは限らない — Claude Code・Codex・Antigravity の管理設定'
description: 'Claude Code は自動承認の auto モードが既定、Antigravity はターミナル自動実行が「Always proceed」でサンドボックスは既定オフ。開発者向けエージェントを MDM で統制するための設定の勘所をまとめる。'
pubDate: 2026-10-09
order: 6
category: 'コーディングエージェント'
tags: ['Claude Code', 'Codex', 'Antigravity', 'MDM', 'サンドボックス']
vendors: ['OpenAI', 'Anthropic', 'Google']
---

コーディングエージェントは、チャットとはリスクの質が違います。**端末でコマンドが動く**からです。開発者には、チャット利用とは別のポリシーを用意する必要があります。

しかも、既定値が安全側とは限りません。

- **Claude Code**：自動承認の auto モードが既定の開始モード。
- **Antigravity（Gemini Enterprise 経由）**：ターミナル自動実行の既定が「Always proceed」で、サンドボックスは既定オフ。

組織で設定を強制できるのは、Codex（Enterprise）、Claude Code（Team 以上）、Antigravity（Gemini Enterprise Standard / Plus）です。

## 大原則：管理画面から配る設定は「境界」ではない

コーディングエージェントの設定を管理画面から配信する方式は、会社管理でない端末では利用者が回避できます。Anthropic も Google も、公式に「セキュリティ境界ではない」と書いています。**強制力が必要なら MDM で端末に配ります。**

## Claude Code

設定項目ごとの勘所です。

- **auto モードの禁止**：`permissions.disableAutoMode` を `"disable"` にします。
- **承認を全部省略するモードの禁止**：`permissions.disableBypassPermissionsMode` を `"disable"` にします。
- **コマンド禁止では通信は止まらない**：`Bash(curl *)` の禁止は、書かれたコマンド文字列にしか一致しません。スクリプト経由や別コマンドでの通信は通ります。通信の制限はサンドボックスのドメイン許可リストで行います。
- **サンドボックスの通信先は空から始まる**：`sandbox.network.allowManagedDomainsOnly` を true にすると、管理設定に書いたドメイン以外は確認なしで遮断され、利用者は追加できません。
- **サンドボックスの外で動くものがある**：ファイル読み書きや Web 取得の組み込みツール、MCP サーバー、フックは対象外です。それぞれ権限ルール、MCP 許可リスト、`allowManagedHooksOnly` で締めます。
- **失敗したコマンドをサンドボックス外で再実行する抜け道**：`allowUnsandboxedCommands` を false にして塞ぎます。
- **ネイティブ Windows ではサンドボックスが動かない**：`failIfUnavailable` を true にすると Windows 端末では起動しなくなります。Windows の開発者は WSL2 で使わせます。
- **MCP サーバーは名前ではなく URL で許可する**：名前は利用者が自由に付けられます。`allowManagedMcpServersOnly` を true にします。
- **接続先の差し替えで配信を回避できる**：独自の接続先（`ANTHROPIC_BASE_URL`）を設定すると、管理画面からの配信の取得自体が行われません。MDM 配布と併用し、`forceRemoteSettingsRefresh` で取得失敗時に起動させないようにします。
- **リポジトリにコミットされた設定**：`allowManagedPermissionRulesOnly` を true にすると、リポジトリや利用者の許可ルールを無視できます。

`managed-settings.json` の例（値は自社に合わせます）：

```json
{
  "forceLoginMethod": "claudeai",
  "forceLoginOrgUUID": ["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"],
  "forceRemoteSettingsRefresh": true,
  "permissions": {
    "deny": ["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"],
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable"
  },
  "allowManagedPermissionRulesOnly": true,
  "allowedMcpServers": [{ "serverUrl": "https://mcp.example.co.jp/*" }],
  "allowManagedMcpServersOnly": true,
  "allowManagedHooksOnly": true,
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowedDomains": ["registry.npmjs.org", "github.com"],
      "allowManagedDomainsOnly": true
    }
  }
}
```

配布先は、macOS が `/Library/Application Support/ClaudeCode/managed-settings.json` か構成プロファイル（`com.anthropic.claudecode`）、Linux と WSL が `/etc/claude-code/managed-settings.json`、Windows が `C:\Program Files\ClaudeCode\managed-settings.json` かレジストリ（`HKLM\SOFTWARE\Policies\ClaudeCode`）です。管理画面からの配信はグループ別に出し分けられないので、部門ごとに変えるなら MDM のプロファイルを分けます。

## Codex

設定強制は `requirements.toml` で行います。配布経路の優先順位は、macOS の MDM が最も強く、次に旧形式の管理設定、管理画面からの配信、システムファイルの順です。Business は管理画面からの配信の対象か確認できていないので、MDM かシステムファイル（`/etc/codex/requirements.toml`、Windows は `%ProgramData%\OpenAI\Codex\requirements.toml`）で配ります。

```toml
# 承認なしの実行と、全権限のサンドボックス解除を選べなくする
allowed_approval_policies = ["on-request"]
allowed_sandbox_modes = ["read-only", "workspace-write"]
allowed_web_search_modes = ["cached"]
allow_managed_hooks_only = true

# 許可した MCP サーバーだけを使わせる（名前ではなく URL で照合）
[mcp_servers.internal_docs]
identity = { url = "https://mcp.example.co.jp/mcp" }

# 通信先を許可リストに固定（実験的機能）
[experimental_network]
enabled = true
managed_allowed_domains_only = true

[experimental_network.domains]
"api.openai.com" = "allow"
"registry.npmjs.org" = "allow"

# ブラウザ操作と PC 操作を無効化
[features]
browser_use = false
computer_use = false
```

- フックのスクリプト本体は配信されません。MDM で端末に置く必要があります。
- 新しい Codex では `allowed_sandbox_modes` より権限プロファイル（`allowed_permission_profiles`）の利用が推奨されています。配布前に公式ページで書式を確認します。
- 「Work Cloud からのローカル PC アクセス」は ZDR に対応しません。ZDR が要件なら有効にしません。

## Antigravity（Gemini Enterprise）

管理コンソールの既定値を必ず変えます。

| 設定 | 既定 | 推奨 |
|---|---|---|
| ターミナルの自動実行 | Always proceed（常に実行） | Require review、または Proceed in sandbox |
| サンドボックス | オフ | オン |
| ワークスペース外のファイルアクセス | Deny | そのまま |
| ブラウザアクセス | オフ | そのまま。使うなら許可 URL を設定 |
| MCP サーバー | オフ | そのまま。使うなら許可リストを設定 |
| プロンプト・応答のログ | オフ | 監査要件があればオン |
| 他社モデル | オフ | 必要時のみ。超過課金の有効化が前提 |

- Gemini Enterprise 内の Antigravity は、SOC・ISO・Access Transparency の対象外です。
- 日本リージョンでは開発者ツールの所在地保証がありません。
- Gemini CLI と Gemini Code Assist の個人利用は2026年6月に終了し、Code Assist Standard / Enterprise は2026年10月9日で新規販売を終了しました。これから導入するなら Antigravity が前提になります。

## 開発者向けポリシーのチェックリスト

- [ ] 管理設定を MDM で配布した
- [ ] 自動承認・承認省略モードの扱いを固定した
- [ ] サンドボックスを有効にし、通信先を許可リストにした
- [ ] MCP サーバーとプラグインの入手元を許可リストにした
- [ ] 個人アカウントでのログインを端末側で塞いだ
- [ ] Windows 端末の扱いを決めた

> 本記事は2026年10月9日時点の各社公式ドキュメントにもとづきます。設定キーの名前や書式は頻繁に変わるため、配布前に公式ページで確認してください。
