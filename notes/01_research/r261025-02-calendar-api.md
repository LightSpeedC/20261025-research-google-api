# Google Calendar API 調査

Google API 調査の第 2 回。Calendar API のリソース・スコープ・クォータ・通知と同期・繰り返し予定を公式ドキュメントで確かめた

> 📅 作成: 2026-10-25 / 更新: 2026-10-25

[README へ戻る](../../README.md)

1. [概要とリソース](#1-概要とリソース)
2. [スコープ](#2-スコープ)
3. [クォータ](#3-クォータ)
4. [プッシュ通知と増分同期](#4-プッシュ通知と増分同期)
5. [繰り返し予定](#5-繰り返し予定)

## 1. 概要とリソース

Google Calendar API は RESTful API。HTTP を直接呼ぶか、Google のクライアントライブラリで使う。カレンダーの Web 画面にある機能の大部分を、API から扱える。

| リソース | 内容 |
|---|---|
| `Events` | タイトル・開始と終了・参加者などを持つ予定。単発と繰り返しがある |
| `Calendars` | イベントの集まり。説明文や既定のタイムゾーンなどのメタデータを持つ |
| `CalendarList` | ユーザーのカレンダー一覧に出るカレンダーの集まり。色や新規イベントの通知設定など、ユーザーごとの属性を持つ (`CalendarListEntry`) |
| `Settings` | タイムゾーンなど、画面上のユーザー設定 |
| `Acl` | ユーザーやグループに、カレンダーへの特定のアクセス権を与えるルール |

出典: [Google Calendar API の概要 (Google for Developers)](https://developers.google.com/workspace/calendar/api/guides/overview)

### Colors

- メソッドは `get` だけ (`list` は無い)。`GET https://www.googleapis.com/calendar/v3/colors`
- 返るのは `calendar` と `event` の 2 つのパレットと、`updated`。いずれも読み取り専用
- パレットは色 ID をキーにし、各色は `background` (背景色) と `foreground` (その上に文字を描く色) を持つ
- `calendarListEntry` の `colorId` は `calendar` パレット、イベントの `colorId` は `event` パレットを指す
- 認証は任意。認証するなら、`calendar`・`calendar.readonly`・`calendar.app.created`・`calendar.calendarlist`・`calendar.calendarlist.readonly`・`calendar.events.freebusy`・`calendar.events.owned`・`calendar.events.owned.readonly`・`calendar.events.public.readonly` のどれか

出典: [Colors (Google for Developers)](https://developers.google.com/workspace/calendar/api/v3/reference/colors) ・ [colors.get (Google for Developers)](https://developers.google.com/workspace/calendar/api/v3/reference/colors/get)

### Freebusy

`POST https://www.googleapis.com/calendar/v3/freeBusy` で、複数のカレンダーやグループの、予定が入っている時間帯 (busy) を返す。

| 項目 | 内容 |
|---|---|
| `timeMin` / `timeMax` | 検索範囲の開始と終了 (RFC3339) |
| `timeZone` | 応答のタイムゾーン。既定は UTC |
| `items[].id` | 照会するカレンダーまたはグループの ID |
| `groupExpansionMax` | 1 グループに含めるカレンダー ID の最大数。上限 100。超えるとエラー |
| `calendarExpansionMax` | 空き状況を返すカレンダーの最大数。上限 50 |

- 必要なスコープは、`calendar.readonly`・`calendar`・`calendar.events.freebusy`・`calendar.freebusy` のどれか
- 応答の `calendars` に、カレンダーごとの `busy[]` (`start` は開始を含み、`end` は終了を含まない) と `errors[]` が入る。`groups` にはグループの展開結果が入る
- エラーの `reason` は `groupTooBig`・`tooManyCalendarsRequested`・`notFound`・`internalError`。今後増える可能性があるため、未知の値も扱えるようにする

出典: [Freebusy.query (Google for Developers)](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query)

## 2. スコープ

公式ページに 20 件が載っている。接頭辞はすべて `https://www.googleapis.com/auth/`。公式は「できるだけ範囲の狭いスコープを選ぶ」よう求めている。

| 用途 | スコープ (接頭辞を省略) | 概要 |
|---|---|---|
| 全体 | `calendar` | 閲覧できるすべてのカレンダーの閲覧・編集・共有、完全削除 |
| 全体 | `calendar.readonly` | 閲覧できるカレンダーの閲覧・ダウンロード |
| イベント | `calendar.events` | すべてのカレンダーのイベントの閲覧・編集 |
| イベント | `calendar.events.readonly` | すべてのカレンダーのイベントの閲覧 |
| イベント | `calendar.events.owned` | 所有カレンダーのイベントの作成・変更・削除 |
| イベント | `calendar.events.owned.readonly` | 所有カレンダーのイベントの閲覧 |
| イベント | `calendar.events.public.readonly` | 公開カレンダーのイベントの閲覧 |
| 空き状況 | `calendar.freebusy` | 予定の空き状況の確認 |
| 空き状況 | `calendar.events.freebusy` | アクセスできるカレンダーの空き状況の閲覧 |
| カレンダー | `calendar.calendars` | カレンダーのプロパティの変更、セカンダリカレンダーの作成 |
| カレンダー | `calendar.calendars.readonly` | カレンダーのタイトル・説明・タイムゾーンなどの閲覧 |
| カレンダー | `calendar.app.created` | セカンダリカレンダーの作成と、そのイベントの操作 |
| 一覧 | `calendar.calendarlist` | 購読中のカレンダーの閲覧・追加・削除 |
| 一覧 | `calendar.calendarlist.readonly` | 購読中のカレンダー一覧の閲覧 |
| 共有 | `calendar.acls` | 所有カレンダーの共有権限の閲覧・変更 |
| 共有 | `calendar.acls.readonly` | 所有カレンダーの共有権限の閲覧 |
| 設定 | `calendar.settings.readonly` | カレンダー設定の閲覧 |
| アドオン | `calendar.addons.execute` | カレンダーアドオンとしての実行 |
| アドオン | `calendar.addons.current.event.read` | 画面で開いたイベントの閲覧 |
| アドオン | `calendar.addons.current.event.write` | 画面で開いたイベントの編集 |

> [!IMPORTANT]
> 各スコープが sensitive か non-sensitive かを示す公式の一覧表は、見つからなかった。公式で確かめられたのは、次の 2 点だけ。<br>・Google の検証ガイドが、sensitive の例として「Google カレンダーに保存されたイベントの閲覧」を挙げている<br>・区分は、Cloud Console の Google Auth platform > Data Access で、スコープを追加する画面に non-sensitive・sensitive・restricted の 3 つの節として出る (静的な一覧表ではない)

### 区分を探した範囲

- **公式で探したが、スコープごとの区分は書かれていなかったページ:** Calendar API のスコープ、OAuth 2.0 Scopes for Google APIs (Calendar の節)、Sensitive scope verification、OAuth 同意画面の設定ガイド
- **非公式 (参考。公式ではない):** Nylas の開発者向けガイドは「Google は Calendar のスコープを sensitive に分類するため、本番のアプリは Google の OAuth 検証が要る」と書く。スコープごとの区分は示していない

出典 (公式): [Sensitive scope verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification) ・ [OAuth 同意画面の設定](https://developers.google.com/workspace/guides/configure-oauth-consent) ・ [OAuth 2.0 Scopes for Google APIs](https://developers.google.com/identity/protocols/oauth2/scopes)<br> 出典 (非公式): [How to use OAuth for Google Calendar (Nylas)](https://developer.nylas.com/docs/cookbook/calendar/google-calendar-oauth/)

範囲の選び方 (この資料の解釈): 閲覧だけなら `.readonly` 付きにする。自分の所有カレンダーだけを扱うなら、`calendar.events` より `calendar.events.owned` が狭い。空き状況だけなら `calendar.freebusy` で足りる可能性がある。公式の文面ではなく、スコープ名と概要からの推測。

出典: [Google Calendar API のスコープ (Google for Developers)](https://developers.google.com/workspace/calendar/api/auth)

## 3. クォータ

| 種類 | 上限 |
|---|---:|
| 1 分あたり・プロジェクト | 10,000 リクエスト |
| 1 分あたり・プロジェクト × ユーザー | 600 リクエスト |
| 1 日あたり・プロジェクト (課金のしきい値) | 1,000,000 リクエスト |

- 1 分ごとのスライディングウィンドウで集計する。1 分間に上限を超えるバーストがあると、次のウィンドウで制限がかかることがある
- 1 日 1,000,000 は、これを下回る範囲では課金が発生しないしきい値。引き上げの申請はできない。標準的な利用は追加料金なしだが、超えた分は 2026 年中に課金される予定とページにある。変更の前に少なくとも 90 日前の通知があるとされている
- 超過すると `403 usageLimits` または `429 usageLimits` が返る

### 対処

- 指数バックオフで再試行する。待ち時間は `min((2^n) + ランダムミリ秒, maximum_backoff)`。ランダムは 1,000 ミリ秒以下で、再試行のたびに計算し直す。`maximum_backoff` は 32〜64 秒が目安。再試行の回数にも上限を設ける
- 同じカレンダーへの短時間の連続書き込みは避ける。運用上の制限は予告なく変わることがある。間隔を空けるか、バッチリクエストを検討する
- 定期同期の時刻をクライアントごとにずらし、間隔に ±25% ほどの揺らぎを持たせる
- ポーリングの代わりにプッシュ通知を使う
- ドメイン全体の委任を使うサービスアカウントでは、`quotaUser` パラメータ (または `x-goog-quota-user` ヘッダー) で実際のユーザーを指定する

出典: [Calendar API のクォータの管理 (Google for Developers)](https://developers.google.com/workspace/calendar/api/guides/quota)

## 4. プッシュ通知と増分同期

### プッシュ通知 (watch)

対象は `Acl`・`CalendarList`・`Events`・`Settings` の 4 種類。通知を受ける HTTPS のサーバーが要る。

| watch の項目 | 内容 |
|---|---|
| `id` (必須) | プロジェクト内で一意。最大 64 文字。UUID を推奨 |
| `type` (必須) | `web_hook` |
| `address` (必須) | 通知を受ける HTTPS の URL。有効な SSL 証明書が要る (自己署名・信頼されない発行元・失効・ホスト名の不一致は不可) |
| `token` | 最大 256 文字。通知の検証や振り分けに使う。機密情報は入れない |
| `expiration` | ミリ秒の Unix タイムスタンプ |

- 作成の直後に `X-Goog-Resource-State: sync` の通知が届く。メッセージ番号は常に 1。watch の応答より先に届くことがある
- 通知に本文は無い。変更の中身は、API を呼んで取る
- `X-Goog-Resource-State` は `sync`・`exists`・`not_exists` のいずれか
- 成功とみなす応答は 200・201・202・204・102。クライアントライブラリでは 500・502・503・504 で指数バックオフの再送がある
- 止めるときは `https://www.googleapis.com/calendar/v3/channels/stop` に `id` と `resourceId` を POST する
- `Events` と `Acl` は**カレンダーごと**、`Settings` と `CalendarList` は**ユーザーごとに 1 回**の購読で足りる

> [!WARNING]
> チャネルに自動更新は無い。期限が近づいたら、新しい `id` で watch をもう一度呼ぶ (新旧が並行して有効な時間帯がある)。通知は一定の割合で欠落することがあり、届かなくても同期が進む作りにする。

出典: [Calendar API のプッシュ通知 (Google for Developers)](https://developers.google.com/workspace/calendar/api/guides/push)

### 増分同期 (syncToken)

1. **初回の全同期:** list で全件を取り、最後のページの `nextSyncToken` を保存する
2. **増分同期:** 保存した値を `syncToken` に入れて list する。結果には削除されたエントリも入る。`pageToken` が返ったら、最初と同じクエリに `pageToken` を足して順に取り、最後のページの新しい `syncToken` を保存する
3. **410 GONE:** トークンの期限切れや関連 ACL の変更で無効になると返る。ローカルのデータを全部消して、全同期からやり直す

- 増分同期で使えるクエリパラメータには制限がある。初回を含めて、すべてのリクエストで同じパラメータの組を使う。許されない指定は `400`
- 期間で絞るなら初回の全同期で使う
- イベントの古い方式 (`updated` と `modifiedSince`) は取りこぼしやすく、非推奨

出典: [Calendar API の増分同期 (Google for Developers)](https://developers.google.com/workspace/calendar/api/guides/sync)

## 5. 繰り返し予定

- **作成:** イベントの `recurrence` に RRULE を入れる (例: `RRULE:FREQ=WEEKLY;UNTIL=20110701T170000Z`)
- **`events.list()` の既定:** 単発の予定・繰り返しの親・例外だけを返す。例外ではないインスタンスは返さない
- **`singleEvents=true`:** 各インスタンスが個別に返るが、親は返らない。空き状況だけを見る権限のユーザーも、同じ動きになる
- **`events.instances()`:** 特定の繰り返し予定のインスタンスを全部取る

### インスタンスの見分け方

| フィールド | 内容 |
|---|---|
| `recurrence` | 親だけが持つ。インスタンスは持たない |
| `recurringEventId` | 所属する親予定の ID |
| `originalStartTime` | 親の繰り返しルール上の本来の開始時刻。日時が変えられていても、系列の中の位置を特定できる |

インスタンスの変更・削除は、取得したインスタンスへの PUT で行う。取り消すには `status` を `cancelled` にする。この操作で例外ができる。

> [!CAUTION]
> 系列全体や「以降すべて」を変えたいときに、インスタンスを 1 つずつ変更しない。例外が大量にでき、カレンダーが煩雑になり、アクセスが遅くなり、通知も大量に送られる。

出典: [繰り返しイベント (Google for Developers)](https://developers.google.com/workspace/calendar/api/guides/recurringevents)

[README へ戻る](../../README.md)
