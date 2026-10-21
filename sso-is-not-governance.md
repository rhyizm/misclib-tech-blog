---
title: '「SSO がある」と「統制できる」は別物 — 法人向け生成AIプランの層をどう選ぶか'
description: 'ChatGPT Business と Claude Team は SSO まで。SCIM、監査ログ、接続元制限はプランの機能としてはエンタープライズ層に限られる。IdP やゲートウェイで補える範囲と、補えない範囲を整理する。'
pubDate: 2026-10-09
order: 1
category: 'プラン選定'
tags: ['SSO', 'SCIM', '監査ログ', 'プラン比較']
vendors: ['OpenAI', 'Anthropic', 'Google']
---

生成AIの法人導入でよく聞くのが「Business（Team）プランなら SSO があるので、会社として管理できている」という判断です。半分は正しく、半分は誤りです。SSO で締まるのは**入口**だけで、退職者のアカウントが自動で止まるか、誰が何をしたか追えるか、社外からの利用を防げるかは、別の機能が担っています。

この記事では、OpenAI・Anthropic・Google の3社について、プランを「個人」「チーム・中小」「エンタープライズ」「API」の4層に分けて、どの層で何ができるようになるかを整理します。

## 4つの利用層

| 利用層 | 意味 | 該当プラン |
|---|---|---|
| 個人 | 個人が自分で契約する。管理者という概念がない | ChatGPT Free・Go・Plus・Pro、Claude Free・Pro・Max、Google AI Plus・Pro・Ultra |
| チーム・中小 | 法人契約だがセルフサーブ。SSO までは付くことが多い | ChatGPT Business、Claude Team、Workspace Business 各種、Gemini Enterprise Business |
| エンタープライズ | 統制機能が揃う層 | ChatGPT Enterprise、Claude Enterprise、Workspace Enterprise 各種、Gemini Enterprise Standard / Plus |
| API | 自社システムに組み込む開発者向け | OpenAI API、Claude API、Gemini API、Agent Platform（旧 Vertex AI） |

Google だけは製品系統が3つ（Workspace、Gemini Enterprise、Gemini API / Agent Platform）あり、Workspace と Gemini Enterprise は別契約である点に注意が必要です。

## 原則1：個人プランは業務利用の対象外にする

個人向けプランは**有料でも学習利用が既定でオン**です。3社とも設定でオフにできますが、それは利用者任せで、管理者機能は一切ありません。Google は個人向けで有料・無料のデータの扱いを区別していません。

「有料プランだから大丈夫」は通用しない、というのが出発点です。

## 原則2：全社導入はエンタープライズ層を基準にする

Team / Business プラン（Claude Team、ChatGPT Business など）は「入口（SSO）は締められるが、SCIM・監査ログ・接続元制限をプランの機能としては持たない」層です。IdP やネットワーク側のゲートウェイで補える部分も多いので、補完策と、それでも残る差を承知したうえで選びます。

たとえば ChatGPT Business には次のものがありません。

- SCIM（退職者の自動停止）
- 監査ログ、Compliance API
- IP 許可リスト、カスタムロール
- ISO 認証、出力の著作権補償

Claude Team も同様で、SSO と JIT プロビジョニングまで。SCIM、監査ログ、Compliance API、保持期間設定、カスタムロール、ドメインキャプチャ、IP 許可リストは Enterprise 限定です。

### SSO と SCIM と JIT の違い

- **SSO**：会社の認証基盤（Entra ID、Okta など）でログインさせる。パスワード管理と多要素認証を会社側に寄せられる。
- **JIT プロビジョニング**：初回ログイン時にアカウントを自動作成する。**作成は自動だが、停止は自動にならない**。
- **SCIM**：人事異動・退職を AI サービス側のアカウントに自動反映する。退職者のアカウントが自動で止まるかはここで決まる。

SCIM がなくても、**SSO を必須にしておけば**、IdP 側でアカウントを無効化するかアプリの割り当てを外すだけで、AI サービスへの新たなログインは止められます。Anthropic も、Team プランで IdP の割り当てを外したユーザーはログインできなくなると明記しています。

ただし止まるのはログインまでです。メンバー一覧には残り、シートも消費したままなので、削除は管理者の手作業になります。すでに開いているセッションが即座に切れるかは、公式記事に記載がありません。SCIM が使えないプランでは、IdP での停止に加えて、AI サービス側のメンバー削除を退職・異動の人事フローに組み込みます。

接続元の制限も同じ考え方で補えます。Entra ID の条件付きアクセスや Google Workspace のコンテキストアウェア アクセスを SSO のアプリに適用すれば、ログイン時の場所や端末を制限できます。ただし評価はログインのタイミングだけで、ログイン後に場所が変わっても再評価されません。

## 要件別の最低ライン

| 要件 | OpenAI | Anthropic | Google |
|---|---|---|---|
| 学習に使われない＋SSO | ChatGPT Business | Claude Team | Workspace Business Standard 以上 |
| SCIM、監査ログ、カスタムロール、接続元制限 | ChatGPT Enterprise | Claude Enterprise | Workspace Enterprise Standard 以上 |
| 会話本文の取得・保全 | Enterprise（Compliance API） | Enterprise（Compliance API） | Workspace Business Plus 以上（Vault） |
| コーディングエージェントの設定を組織で強制 | Enterprise（Business は端末側配布のみ） | Team 以上 | Gemini Enterprise Standard / Plus |

Workspace は Business エディションでも学習には使われませんが、Gemini 向け DLP・条件付きアクセス・ログの外部出力は Enterprise エディションからです。

## プラン比較の前に決める5つの要件

マトリックスを眺める前に、次の5つを決めておくと、見るべき行が絞れます。

1. **入力してよい情報の上限**：公開情報のみ／社内情報まで／個人情報・顧客情報・ソースコードまで。
2. **国内保存・国内処理の要否**：「保存が国内」で足りるのか、「処理も国内」が必要か。
3. **ISMAP の要否**：官公庁案件や、調達基準で引用されているか。
4. **監査の深さ**：操作記録だけでよいか、会話本文まで取得・保全する必要があるか。
5. **開発者利用の有無**：コーディングエージェントを使わせるか。使わせるなら端末管理（MDM）があるか。

## まとめ

- 個人プランは有料でも業務利用の対象外。
- SSO は入口の統制であって、統制の全体ではない。
- Team / Business プランを選ぶなら「できないこと」を明文化し、手動運用か別の仕組みで補う。

Team / Business プランで会話の記録や入力の検査を補う方法は、[Enterprise プランなしで生成AIの会話を記録・検査する方法](/blog/team-tier-network-logging/)で解説しています。3社27プランを26項目で並べた[比較マトリックス](/secure-ai-install/matrix.html)も公開しています。

> 本記事は2026年10月9日時点の各社公式ドキュメントにもとづきます。各社とも既定値やプラン構成の変更が頻繁なため、契約・設定の直前に公式ページで再確認してください。
