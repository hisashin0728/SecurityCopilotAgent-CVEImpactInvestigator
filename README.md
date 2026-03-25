# CVE Impact Investigator

> CVE ID を入力すると、Microsoft Defender の脆弱性データを基に自社環境への影響を調査し、影響端末・ソフトウェア・リスク評価を含む日本語レポートを生成する Security Copilot Interactive Agent ソリューション。

## アーキテクチャ

```
┌──────────────┐     HTTP POST      ┌──────────────────┐     Security Copilot API     ┌────────────────────┐
│  Web フロント │ ──────────────────▶ │   Azure Logic App │ ──────────────────────────▶ │  Security Copilot  │
│  (HTML/JS)   │ ◀────────────────── │   (ARM Template)  │ ◀────────────────────────── │  Interactive Agent │
└──────────────┘     JSON レポート    └──────────────────┘     評価結果 (Markdown)       └────────────────────┘
                                                                                              │
                                                                                   ┌──────────┼──────────┐
                                                                                   ▼          ▼          ▼
                                                                             KQL Skills   KQL Skills   DTI Skill
                                                                            (Defender)   (DeviceInfo) (ThreatIntel)
```

## 含まれるファイル

| ファイル | 説明 |
|---------|------|
| `CVEImpactInvestigator.yaml` | Security Copilot Agent マニフェスト（YAML） |
| `CVEImpactInvestigator_LogicApp_ARM.json` | Logic App ARM テンプレート（Web → Security Copilot ブリッジ） |
| `CVEImpactInvestigator_PowerApp.html` | Web フロントエンド（HTML/CSS/JS 単一ファイル） |
| `CVEImpactInvestigator_card.html` | プラグインカード（ビジュアル概要） |

## 前提条件

- **Microsoft Security Copilot** へのアクセス
- **Microsoft Defender for Endpoint** が有効（Advanced Hunting テーブルが必要）
- **Azure サブスクリプション** — Logic App デプロイ用
- **Entra ID アプリ登録** — Security Copilot API 呼び出し用

### 必要な Entra ID API 権限

| 権限 | 種類 | 用途 |
|------|------|------|
| `SecurityCopilot.Sessions.ReadWrite` | Application | セッション作成・評価実行 |

## セットアップ手順

### 1. Security Copilot Agent のアップロード

1. [Security Copilot ポータル](https://securitycopilot.microsoft.com) にアクセス
2. **Settings** → **Custom plugins** → **Add plugin**
3. `CVEImpactInvestigator.yaml` をアップロード
4. プラグインを有効化

### 2. Entra ID アプリ登録

1. [Azure Portal](https://portal.azure.com) → **Entra ID** → **App registrations** → **New registration**
2. 名前: `CVEImpactInvestigator-App`
3. **API permissions** → **Add a permission** → **APIs my organization uses**
4. `Microsoft Security Copilot` を検索 → `SecurityCopilot.Sessions.ReadWrite` を追加
5. **管理者の同意を付与**
6. **Certificates & secrets** → 新しいクライアントシークレットを作成
7. **テナント ID**、**クライアント ID**、**クライアントシークレット** をメモ

### 3. Logic App のデプロイ

Azure CLI でデプロイ:

```powershell
az deployment group create `
  --resource-group <リソースグループ名> `
  --template-file CVEImpactInvestigator_LogicApp_ARM.json `
  --parameters `
    tenantId="<テナント ID>" `
    clientId="<クライアント ID>" `
    clientSecret="<クライアントシークレット>" `
    securityCopilotApiBase="https://api.medeina-dev.defender.microsoft.com" `
    securityCopilotGeoRegion="eastus"
```

> **リージョン選択肢**: `eastus`, `westeurope`, `uksouth`, `australiaeast`, `eastasia`, `japaneast`

デプロイ後、Logic App の **HTTP POST URL** を取得:

```powershell
az rest --method POST `
  --uri "https://management.azure.com/subscriptions/<サブスクリプションID>/resourceGroups/<リソースグループ名>/providers/Microsoft.Logic/workflows/CVEImpactInvestigator-LogicApp/triggers/HTTP要求の受信時/listCallbackUrl?api-version=2016-10-01" `
  --query value -o tsv
```

### 4. Web フロントエンドの設定

`CVEImpactInvestigator_PowerApp.html` を開き、`CONFIG` セクションを編集:

```javascript
const CONFIG = {
  FLOW_URL: '<Logic App の HTTP POST URL をここに貼り付け>',
  DEMO_MODE: false
};
```

ブラウザで HTML ファイルを開けば利用可能です。

## Agent の構成

### スキル一覧

| スキル名 | 形式 | 説明 |
|---------|------|------|
| `InvestigateCVEImpact` | Agent (Interactive) | メインオーケストレーター — ワークフロー全体を制御 |
| `GetDevicesVulnerableToCve` | KQL (Defender) | CVE に脆弱なデバイス一覧を取得 |
| `GetDeviceDetailsForCve` | KQL (Defender) | 影響デバイスの詳細情報（露出レベル、インターネット公開等）を取得 |
| `GetAffectedSoftwareInventory` | KQL (Defender) | 影響ソフトウェアとインストール端末数を取得 |
| `GetCvesByIdsDti` | 外部 (ThreatIntelligence.DTI) | CVE の脅威インテリジェンス情報を取得 |

### ワークフロー

```
1. CVE 詳細情報の取得        → GetCvesByIdsDti
2. 影響デバイスの特定         → GetDevicesVulnerableToCve
3. デバイス詳細情報の取得     → GetDeviceDetailsForCve
4. 影響ソフトウェアの特定     → GetAffectedSoftwareInventory
5. 日本語レポートの生成       → Agent が統合してレポート作成
```

### 出力レポート構成

- **CVE 概要** — CVSS スコア、深刻度、公開日、エクスプロイト有無
- **自社環境への影響サマリ** — 影響端末数、深刻度別分布
- **影響端末一覧** — テーブル形式（デバイス名、OS、露出レベル等）
- **影響ソフトウェア一覧** — テーブル形式（ソフトウェア名、バージョン、端末数）
- **リスク評価** — 総合リスクレベルと判定根拠
- **推奨アクション** — パッチ情報、優先対応端末、緩和策

## Logic App のフロー

```
HTTP POST (UserRequest) 
  → OAuth トークン取得 (Entra ID)
  → Security Copilot セッション作成
  → Evaluation 送信 (InvestigateCVEImpact)
  → ポーリング (10秒間隔、最大10分)
  → レポート返却 (JSON)
```

### Logic App トリガーの JSON スキーマ

```json
{
  "type": "object",
  "properties": {
    "UserRequest": {
      "type": "string",
      "description": "調査対象の CVE ID（カンマ区切りで複数指定可能）"
    }
  },
  "required": ["UserRequest"]
}
```

### Logic App レスポンス例

```json
{
  "status": "Completed",
  "report": "## CVE 概要\n- **CVE ID**: CVE-2024-38014\n..."
}
```

## Web フロントエンド機能

- **CVE ID 入力** — テキスト入力 + クイックチップ（よく使う CVE をワンクリック追加）
- **CVE 形式バリデーション** — `CVE-YYYY-NNNNN` 形式を自動検証
- **進捗表示** — 5 段階プログレスバー
- **Markdown レンダリング** — テーブル、見出し、リスト、太字、コードをHTML変換
- **レポート操作** — クリップボードコピー / Markdown ダウンロード
- **調査履歴** — localStorage に最大 10 件保存
- **デモモード** — `DEMO_MODE: true` で API 不要のサンプルレポート表示
- **レスポンシブ** — モバイル対応レイアウト

## トラブルシューティング

| 症状 | 原因 | 対処法 |
|------|------|--------|
| `NoResponse` エラー | OAuth audience URL が不正 | API ベース URL にリージョンパス（`/geo/eastus`）を含めない |
| `UserRequest` 必須エラー | パラメータ名の不一致 | Logic App スキーマとHTML両方で `UserRequest` を使用 |
| タイムアウト | Security Copilot の処理時間超過 | HTML の timeout 値を調整、Logic App のポーリング上限を確認 |
| 英語で出力される | Agent の言語ルール不足 | YAML の Instructions 先頭に日本語出力ルールが記載されていることを確認 |
| Logic App 認証失敗 | アプリ権限未付与 | Entra ID で管理者の同意が付与されているか確認 |

## ライセンス

MIT
