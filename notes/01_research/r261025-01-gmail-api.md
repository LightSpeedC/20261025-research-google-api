# Gmail API 調査

Google API 調査の第 1 回。Gmail API の機能・認証・クォータ・通知・送信を公式ドキュメントで確かめた

> 📅 作成: 2026-10-25 / 更新: 2026-10-25

[README へ戻る](../../README.md)

1. [概要とリソース](#1-概要とリソース)
2. [認証とスコープ](#2-認証とスコープ)
3. [クォータ](#3-クォータ)
4. [プッシュ通知と送信](#4-プッシュ通知と送信)

## 1. 概要とリソース

Gmail API は、Gmail のメールボックスの読み取り・送信・整理・設定変更を行う RESTful API。ユーザーの認可を得て Gmail データへ触れる Web アプリの、多くの場面で最適な選択肢とされている。

### 主なリソース

| リソース | 内容 |
|---|---|
| `messages` | 送信者・宛先・件名・本文を持つメール。作成後は内容を変更できない |
| `threads` | 関連するメッセージを会話としてまとめたもの |
| `labels` | 整理用の印。システムラベル (INBOX・TRASH・SPAM 等) は作成・削除できない。ユーザーラベルは作成・変更・削除できる |
| `drafts` | 未送信のメッセージ。内容を差し替えられる。送信すると下書きは消え、`SENT` ラベルのメッセージができる |
| `history` | メールボックスの変更履歴。`historyId` を起点に差分を取る (「プッシュ通知と送信」を参照) |
| `settings` | 送信元エイリアスと署名・転送・フィルタ・不在通知・S/MIME・POP/IMAP・委任・言語 |

### できること

- 読み取り専用でのメールの抽出・インデックス化・バックアップ
- プログラムからのメール送信
- フィルタによる振り分け・ラベル付け
- 署名・転送などの設定管理 (組織全体での署名の標準化を含む)
- メールボックスの変更通知 (Cloud Pub/Sub 連携)
- 同じ Google Workspace 組織内での、別ユーザーのメールボックスへの委任

出典: [Gmail API の概要 (Google for Developers)](https://developers.google.com/workspace/gmail/api/guides)

## 2. 認証とスコープ

### 認証情報の種類

| 種類 | Gmail での使いどころ |
|---|---|
| API キー | 公開データ用。Workspace の大半のデータは OAuth 2.0 が必要で、メールには通常使えない |
| OAuth 2.0 クライアント ID | ユーザー所有のメールに触れる主要な手段。同意画面での承認が要る。長期の利用にはリフレッシュトークンを取る |
| サービスアカウント | 単独ではユーザーのメールに触れない。ドメイン全体の委任を設定すると組織内のユーザーのメールに触れる。有効化できるのはスーパー管理者だけ |

クライアントシークレットは、アプリのコードに平文で含めず安全に保管する。

出典: [認証と認可の概要 (Google for Developers。Workspace API 全般の説明)](https://developers.google.com/workspace/gmail/api/auth/about-auth)

### スコープ

区分は Non-sensitive / Sensitive / Restricted の 3 段階。要る範囲だけを求める。

| スコープ (`https://www.googleapis.com/auth/` 以下) | 区分 | 概要 |
|---|---|---|
| `gmail.labels` | ✅ **Non-sensitive** | ラベルの閲覧と編集 |
| `gmail.addons.current.action.compose` | ✅ **Non-sensitive** | アドオン操作時に下書きの管理とメール送信 |
| `gmail.addons.current.message.action` | ✅ **Non-sensitive** | アドオン操作時にメッセージを閲覧 |
| `gmail.addons.current.message.metadata` | ⚠️ **Sensitive** | アドオン実行時にメタデータを閲覧 |
| `gmail.addons.current.message.readonly` | ⚠️ **Sensitive** | アドオン実行時にメッセージを閲覧 |
| `gmail.send` | ⚠️ **Sensitive** | ユーザーの代理でメールを送信 |
| `gmail.readonly` | ❌ **Restricted** | メールと設定の閲覧 |
| `gmail.compose` | ❌ **Restricted** | 下書きの管理とメール送信 |
| `gmail.insert` | ❌ **Restricted** | メールボックスへのメール追加 |
| `gmail.modify` | ❌ **Restricted** | 閲覧・作成・送信。ゴミ箱を経由した即時の完全削除はできない |
| `gmail.metadata` | ❌ **Restricted** | ラベルやヘッダーの閲覧 (本文は不可) |
| `gmail.settings.basic` | ❌ **Restricted** | メール設定とフィルタの閲覧・変更 |
| `gmail.settings.sharing` | ❌ **Restricted** | 重要な設定の管理。管理者用で、ドメイン全体の委任を受けたサービスアカウントのみ |
| `https://mail.google.com/` | ❌ **Restricted** | 閲覧・作成・送信に加え完全削除も可能 |

> [!NOTE]
> **IMPORTANT**<br>メールを読むスコープ (`gmail.readonly` 等) は、本文を読めないはずの `gmail.metadata` まで含めて Restricted。送信だけなら Sensitive の `gmail.send` で足りる。`https://mail.google.com/` は、即時かつ完全な削除が要るときだけ求めるよう公式が注記している。

出典: [Gmail API のスコープ (Google for Developers)](https://developers.google.com/workspace/gmail/api/auth/scopes)

### リフレッシュトークンが失効する条件

- ユーザーがアプリのアクセスを取り消した
- 6 か月間使われなかった
- ユーザーがパスワードを変え、そのトークンに Gmail のスコープが含まれている
- ユーザーアカウントが付与済みのトークンの上限を超えた。OAuth クライアント ID ごとに 1 つの Google アカウントあたり 100 個で、超えると最も古いものが警告なく無効になる (サービスアカウントは対象外)
- 時間制限付きアクセスの期限が切れた
- 管理者が、アプリのスコープに含まれるサービスを Restricted にした (エラー `admin_policy_enforced`)
- 同意画面が外部ユーザー向けで、公開状態が「テスト」のプロジェクトは、有効期間が **7 日**になる (`openid`・`userinfo.email`・`userinfo.profile` だけを求める場合を除く)

出典: [OAuth 2.0 の概要 (Google for Developers)](https://developers.google.com/identity/protocols/oauth2)

### 制限付きスコープの審査

- 除外に当たらなければ、アプリ審査が要る。スコープの必要性の説明、Cloud Console での宣言、デモ動画の提出などで構成される
- 制限付きスコープのデータを**サーバーに保存または送信する**と、セキュリティ評価 (CASA) が要る。評価は Google が認定した独立の機関が行う
- 評価の承認から**最低 12 か月ごと**に再評価が要る。新しい制限付きスコープを足したときも、前の評価に含まれていなければ再評価になることがある
- 審査の除外: 個人利用、テスト用ユーザーだけに公開している開発・テスト段階、サービスアカウントで自社データだけを扱う場合、Workspace 組織内だけで使うアプリ (Internal)
- 目安は数週間。先にブランド審査 (通常 2〜3 営業日) を終える

出典: [Restricted scope verification (Google for Developers)](https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification)

## 3. クォータ

### 上限

| 種類 | 上限 |
|---|---:|
| 1 分あたり・プロジェクト | 1,200,000 ユニット |
| 1 分あたり・ユーザー / プロジェクト | 6,000 ユニット |
| 1 日あたり・プロジェクト (課金のしきい値) | 80,000,000 ユニット |

- ユーザー単位・プロジェクト単位の上限は「毎秒」ではなく**毎分**
- 1 日 80,000,000 は速度制限ではなく、超えた分に課金が発生するしきい値。引き上げの申請はできない
- 1 通あたりの受信者数は 500 人まで
- 2026 年 5 月 1 日以降に作ったプロジェクトには新しいクォータが適用される。それ以前のプロジェクトは従来の設定のまま

### メソッドごとの消費 (主なもの)

| メソッド | ユニット |
|---|---:|
| `messages.list` / `messages.modify` / `threads.list` | 5 (threads.list は 10) |
| `messages.get` | 20 (format が full・raw・html なら 60) |
| `threads.get` | 40 (format が full なら 120) |
| `messages.send` / `drafts.send` | 100 |
| `messages.batchModify` / `batchDelete` | 50 |
| `history.list` | 2 |
| `watch` | 100 |
| `labels.list` / `labels.get` / `getProfile` | 1 |

> [!NOTE]
> **NOTE**<br>本文まで読む `messages.get` は 60 ユニット。ユーザーあたり毎分 6,000 ユニットなら、本文付きで毎分およそ 100 通が目安になる。ヘッダーだけの取得 (20 ユニット) なら 3 倍。全ユニット表は出典を参照。

出典: [Gmail API の使用制限 (Google for Developers)](https://developers.google.com/workspace/gmail/api/reference/quota)

## 4. プッシュ通知と送信

### プッシュ通知 (watch)

1. Cloud Pub/Sub にトピックとサブスクリプションを作り、`gmail-api-push@system.gserviceaccount.com` にトピックへの publish 権限を与える
2. `users/me/watch` にトピックと対象ラベルを指定して呼ぶ。現在の `historyId` と `expiration` が返る
3. 通知の `message.data` (Base64URL の JSON) にメールアドレスと新しい `historyId` が入る
4. 保存しておいた前回の `historyId` で `history.list` を呼び、差分を取る。処理後に新しい値を保存する
5. Webhook は HTTP 200 などで確認応答になる。pull は acknowledge を明示的に呼ぶ。応答が無いと再送される

> [!NOTE]
> **WARNING**<br>`watch` は 7 日以内に再実行しないと、そのユーザーの更新が届かなくなる。公式は 1 日 1 回の再実行を勧める。通知は通常数秒で届くが、まれに遅延・欠落するため、通知が来ないときの備えとして `history.list` の定期呼び出しを用意する。

- 通知レートはユーザーごとに最大 1 秒あたり 1 件。超過分は破棄される
- 通知の処理中に新しい通知を起こすと通知ループになりうる
- インストール型アプリやモバイル端末などユーザー所有のデバイスには、公式はポーリングによる同期を勧める

出典: [Gmail のプッシュ通知 (Google for Developers)](https://developers.google.com/workspace/gmail/api/guides/push)

### メール送信

```text
1. RFC 2822 準拠の MIME メッセージを作る (添付はマルチパート)
2. base64url でエンコードして、メッセージの raw に入れる
3. messages.send (下書きなら drafts.send) を呼ぶ
```

- 返信を同じスレッドにまとめる条件は、`Subject` の一致と、`References`・`In-Reply-To` の RFC 2822 どおりの設定

出典: [メールの送信 (Google for Developers)](https://developers.google.com/workspace/gmail/api/guides/sending)

### メッセージサイズの上限

公式のガイド・リファレンス (送信ガイド、アップロードのガイド、`messages.send` のリファレンス、クォータ) には、数値の記載が無かった。Google が配布する API 定義 (Discovery ドキュメント) の media upload の `maxSize` には、次の値がある。

| メソッド | maxSize (バイト) | 換算 |
|---|---:|---:|
| `messages.send` | 36,700,160 | 35 MiB |
| `drafts.create` / `drafts.update` / `drafts.send` | 36,700,160 | 35 MiB |
| `messages.import` / `messages.insert` | 157,286,400 | 150 MiB |

> [!NOTE]
> **NOTE**<br>これは media upload (`/upload/...`) の上限。JSON の `raw` に base64url で入れる経路に同じ上限が掛かるかは、確かめられていない。base64url で約 3 分の 1 大きくなるため、添付できる実サイズはこれより小さい。

出典: [Gmail API の Discovery ドキュメント (Google)](https://gmail.googleapis.com/$discovery/rest?version=v1)

[README へ戻る](../../README.md)
