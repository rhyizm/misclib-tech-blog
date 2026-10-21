---
title: '「国内保存」と「国内処理」は別物 — 生成AIのデータレジデンシーと ISMAP の現実'
description: '日本に保存できても、推論は海外というケースが多い。チャット製品と API で取れる経路も違う。国内要件と ISMAP 要件を、どの製品・経路で満たせるかを整理する。'
pubDate: 2026-10-09
order: 7
category: 'データ保護'
tags: ['データレジデンシー', 'ISMAP', 'Azure OpenAI', 'Amazon Bedrock', 'Agent Platform']
vendors: ['OpenAI', 'Anthropic', 'Google']
---

「日本リージョン対応」と聞いて、データが国内で完結すると考えるのは早計です。データレジデンシーには**保存**と**推論処理**の2つがあり、日本に保存できても処理は海外、という構成が多くあります。

## チャット製品で取れる経路

| 要件 | OpenAI | Anthropic | Google |
|---|---|---|---|
| 国内保存 | ChatGPT Enterprise（保存のみ。推論は海外） | 不可 | Gemini Enterprise Standard / Plus |
| 国内処理 | 不可 | claude.ai は不可。例外はデスクトップアプリを Bedrock につなぐ構成のみ | Gemini Enterprise Standard / Plus（Gemini 3.5 Flash のみ保証） |
| ISMAP | 登録を確認できず | 登録を確認できず | Gemini Enterprise。Workspace は本体が登録済みだが、Gemini アプリが範囲内かは未確認 |

補足です。

- **OpenAI** は Enterprise と API で日本保存に対応していますが、推論は米国・欧州・UAE で行われます。社内説明で「国内完結」と言わないことが大切です。また、MCP サーバーに送ったデータはレジデンシーの対象外です。
- **Anthropic** の直販は米国のみです。
- **Google Workspace** のデータリージョンは米国か EU しか選べず、Gemini アプリの保存・処理リージョンを指定できるのは Enterprise Plus と Frontline Plus のみ。日本は選べません。Gemini Notebook（旧 NotebookLM）のデータはデータリージョン設定の対象外で、グローバルに保存されます。
- **Gemini Enterprise** の日本リージョンは許可リスト制で、国内処理が保証されるのは Gemini 3.5 Flash のみです。ほかのモデルや画像・動画生成はグローバルに転送されます。国内処理が要件なら、機能管理で利用可能モデルを絞ります。Google 検索グラウンディングを有効にすると所在地保証の対象外になる点にも注意が必要です。

## クラウド経由は「API としての経路」

Azure OpenAI（Microsoft Foundry）や Amazon Bedrock で国内処理や ISMAP を満たす方法がありますが、これは **API としての経路**です。ChatGPT や claude.ai のチャット製品を、これらのクラウド上で動かすことはできません。

クラウド経由で国内処理や ISMAP を満たせるのは、次の範囲に限られます。

- **自社で開発するアプリケーション**（API を直接呼ぶもの）
- **コーディングエージェント**：Codex の CLI と VS Code 拡張は Azure OpenAI に、Claude Code は Bedrock に接続できます。どちらもクラウド実行などサブスクリプション前提の機能は使えません。
- **Claude デスクトップアプリの「サードパーティ推論」構成**：推論先を Bedrock などに向けた状態で、デスクトップアプリのチャット・Cowork・Code を使えます。会話履歴は端末に保存され、Web 版やモバイルは含まれません。新機能の提供は通常版より遅く、Anthropic 側の監査ログや Compliance API の対象にもなりません。

Bedrock の日本向け推論、Agent Platform 東京とも、対応モデルは限られます。

## API で取れる経路

| 要件 | OpenAI | Anthropic | Google |
|---|---|---|---|
| 国内保存・国内処理 | 直販 API は日本保存のみ。国内処理は Azure OpenAI を別途検討 | Amazon Bedrock の日本向け推論（対応モデル限定） | Agent Platform 東京（対応モデルのみ） |
| ISMAP | 直販 API は確認できず。Azure OpenAI は登録済み（二次情報） | 直販 API は確認できず。Bedrock は登録済みだが、Claude が範囲内かは要確認 | Agent Platform（Vertex AI として登録） |

Agent Platform では、`global` エンドポイントは処理場所の保証がありません。所在地要件があるなら地域エンドポイントを指定し、組織ポリシーで利用できるモデルとエンドポイントを制限します。

## チャット製品で国内処理・ISMAP が必須なら

選択肢は実質3つです。

1. **Gemini Enterprise を選ぶ。**
2. **API の上に載るチャット画面を自社で構築・調達する。**
3. **要件の対象範囲を見直す。** 機密度の高い業務だけ API 経路に分ける、など。

## 見落としやすい「新しい委託先」

ネットワーク側のゲートウェイでログを取る場合、会話本文がゲートウェイ事業者を通り、保存されることがあります。国内保存が要件なら、**ログの保存先も対象**になります。

## まとめ

- 「保存」と「処理」を分けて要件を書く。
- 国内処理や ISMAP の要件が、チャット製品と API のどちらに掛かるのかを切り分ける。
- ISMAP の登録状況は、ポータルのサービスリストで直接確認する。

> 本記事は2026年10月9日時点の公式情報にもとづきます。ISMAP の登録範囲や国内で使えるモデルは変わるため、契約判断の前に ISMAP ポータルと各社の公式ページで確認してください。
