# Google Tasks API 調査

Google API 調査の第 3 回。Tasks API (ToDo リスト) のリソース・フィールド・一覧と移動・スコープとクォータを公式ドキュメントで確かめた

> 📅 作成: 2026-10-25 / 更新: 2026-10-25

[README へ戻る](../../README.md)

1. [概要とエンドポイント](#1-概要とエンドポイント)
2. [Task と TaskList のフィールド](#2-task-と-tasklist-のフィールド)
3. [一覧・移動と上限](#3-一覧移動と上限)
4. [スコープとクォータ](#4-スコープとクォータ)

## 1. 概要とエンドポイント

Google Tasks API は、Google Tasks のコンテンツとメタデータを、プログラムから検索・読み取り・更新するための REST API。自社サービスを Google Tasks につなぐ使い方と、Apps Script での自動化 (メールや文書の操作をきっかけにタスクを作る、外部システムのデータでリストを定期更新する) が、公式の概要に挙がっている。

ベース URL は `https://tasks.googleapis.com`。リソースは `tasklists` (リスト) と `tasks` (タスク) の 2 つ。

### tasklists

| メソッド | HTTP | パス | 内容 |
|---|---|---|---|
| `list` | `GET` | `/tasks/v1/users/@me/lists` | リストの一覧 |
| `get` | `GET` | `/tasks/v1/users/@me/lists/{tasklist}` | 指定のリストを取得 |
| `insert` | `POST` | `/tasks/v1/users/@me/lists` | リストを作成 |
| `patch` | `PATCH` | `/tasks/v1/users/@me/lists/{tasklist}` | 部分更新 |
| `update` | `PUT` | `/tasks/v1/users/@me/lists/{tasklist}` | 更新 |
| `delete` | `DELETE` | `/tasks/v1/users/@me/lists/{tasklist}` | リストを削除 |

### tasks

| メソッド | HTTP | パス | 内容 |
|---|---|---|---|
| `list` | `GET` | `/tasks/v1/lists/{tasklist}/tasks` | タスクの一覧 |
| `get` | `GET` | `/tasks/v1/lists/{tasklist}/tasks/{task}` | 指定のタスクを取得 |
| `insert` | `POST` | `/tasks/v1/lists/{tasklist}/tasks` | タスクを作成 |
| `patch` | `PATCH` | `/tasks/v1/lists/{tasklist}/tasks/{task}` | 部分更新 |
| `update` | `PUT` | `/tasks/v1/lists/{tasklist}/tasks/{task}` | 更新 |
| `delete` | `DELETE` | `/tasks/v1/lists/{tasklist}/tasks/{task}` | タスクを削除 |
| `move` | `POST` | `/tasks/v1/lists/{tasklist}/tasks/{task}/move` | 別の位置 (別のリストを含む) へ移動 |
| `clear` | `POST` | `/tasks/v1/lists/{tasklist}/clear` | 完了済みのタスクをクリア (非表示にする) |

出典: [Google Tasks API の概要](https://developers.google.com/workspace/tasks) ・ [Tasks API REST リファレンス (Google for Developers)](https://developers.google.com/workspace/tasks/reference/rest)

## 2. Task と TaskList のフィールド

### Task

| フィールド | 型 | 読み取り専用 | 内容 |
|---|---|---|---|
| `id` / `etag` | string | 明記なし | 識別子と ETag |
| `title` | string | いいえ | タスク名。最大 1024 文字 |
| `notes` | string | いいえ | メモ。最大 8192 文字。Google ドキュメントから割り当てられたタスクには設定できない |
| `status` | string | いいえ | `needsAction` または `completed` |
| `due` | RFC 3339 | いいえ | 予定日 (下の IMPORTANT を参照) |
| `completed` | RFC 3339 | 明記なし | 完了日時。未完了なら省略される |
| `deleted` | boolean | いいえ | 削除済みか。既定 `false`。割り当てられたタスクでは読み取り専用 |
| `hidden` | boolean | はい | リストをクリアした時点で完了していたタスクが `true` |
| `parent` | string | はい | 親タスクの ID。トップレベルでは省略される。変更は `move` |
| `position` | string | はい | 同じ階層での並び順。辞書順で比べる。変更は `move` |
| `links[]` | object[] | はい | 関連リンク (`type`・`description`・`link`)。`type` の例は `email`・`generic`・`chat_message`・`keep_note` |
| `webViewLink` | string | はい | Web UI で開く絶対 URL |
| `assignmentInfo` | object | はい | 割り当てられたタスクの出所 (`linkToTask`・`surfaceType`・`driveResourceInfo`・`spaceInfo`) |
| `updated` / `selfLink` / `kind` | string | はい | 最終更新日時、操作用の URL、`tasks#task` |

> [!IMPORTANT]
> `due` は期限ではなく、**そのタスクを行う予定日** (カレンダーに表示される日)。RFC 3339 の形だが、設定時に時刻は捨てられ日付だけが残る。公式に「API では予定の時刻を読み書きできない」とある。時刻つきの期限を前提に設計すると誤る。

### TaskList

- `id`・`etag`・`title` (最大 1024 文字)・`updated`・`selfLink`・`kind` (`tasks#taskList`)。`updated`・`selfLink`・`kind` は出力専用

出典: [Tasks リソース](https://developers.google.com/workspace/tasks/reference/rest/v1/tasks) ・ [TaskList リソース (Google for Developers)](https://developers.google.com/workspace/tasks/reference/rest/v1/tasklists)

## 3. 一覧・移動と上限

### tasks.list の主なパラメータ

| パラメータ | 既定・上限 | 内容 |
|---|---|---|
| `maxResults` | 既定 20、上限 100 | 1 ページの件数 |
| `pageToken` | - | 前回の `nextPageToken` を渡す |
| `showCompleted` | 既定 true | 完了済みを含めるか |
| `showHidden` | 既定 false | 非表示を含めるか |
| `showDeleted` | 既定 false | 削除済みを含めるか |
| `showAssigned` | 既定 false | 自分に割り当てられたタスク (Docs・Chat のスペース由来) を含めるか |
| `dueMin` / `dueMax` | 指定なしなら絞らない | 予定日の範囲 (RFC 3339) |
| `completedMin` / `completedMax` | 指定なしなら絞らない | 完了日の範囲 |
| `updatedMin` | 指定なしなら絞らない | 最終更新日時の下限 |

> [!WARNING]
> Web UI やモバイルアプリで完了したタスクは、`showHidden` も `true` にしないと一覧に出ない。「完了済みが取れない」ときは、まずここを疑う。

### tasks.move

- `parent` (新しい親。省略でトップレベル)・`previous` (直前の兄弟。省略で先頭)・`destinationTasklist` (移動先のリスト。省略で同じリスト内) の 3 つを、クエリで渡す。リクエストボディは空
- 親にできるのは、同じリストにあり、非表示でないタスク。割り当てタスクと繰り返しタスクは、親にも子にもできない
- 完了済みで非表示のタスクは入れ子にできず、位置 0 にしか移せない (`parent` と `previous` を空にする)
- 繰り返しタスクは、リストをまたぐ移動に未対応

### 上限

| 項目 | 上限 | 出典 |
|---|---:|---|
| 1 ユーザーのタスクリスト数 | 2,000 | .NET クライアントのリファレンス |
| 1 ユーザー・1 リストあたりの非表示でないタスク数 | 20,000 | `tasks.list` のリファレンス |
| 1 ユーザーの全タスク数 | 100,000 | `tasks.list` のリファレンス |
| 1 タスクあたりのサブタスク数 | 2,000 | `tasks.move` のリファレンス |

リストの上限 2,000 は、公式の REST リファレンスのページには見つからず、Google の .NET クライアントライブラリのリファレンス (`googleapis.dev`) の `TasklistsResource` に「A user can have up to 2000 lists at a time.」とある。

出典: [tasks.list](https://developers.google.com/workspace/tasks/reference/rest/v1/tasks/list) ・ [tasks.move (Google for Developers)](https://developers.google.com/workspace/tasks/reference/rest/v1/tasks/move) ・ [Google.Apis.Tasks.v1 (.NET クライアントのリファレンス)](https://googleapis.dev/dotnet/Google.Apis.Tasks.v1/latest/api/Google.Apis.Tasks.v1.html)

## 4. スコープとクォータ

### スコープ

| スコープ (`https://www.googleapis.com/auth/` 以下) | 内容 |
|---|---|
| `tasks` | すべてのタスクの作成・編集・整理・削除 |
| `tasks.readonly` | タスクの閲覧のみ |

- 公式は、できるだけ範囲の狭いスコープを選ぶよう求めている
- `tasks.move` は `tasks` だけを要求する。`tasks.list` は `tasks` と `tasks.readonly` のどちらでもよい
- 公開アプリで該当するスコープを使うときは、アプリの検証が要る

> [!IMPORTANT]
> 各スコープが sensitive か non-sensitive かは、公式の Tasks のスコープのページに書かれていなかった。Cloud Console の Data Access で見る形で、静的な一覧表は見つからなかった。非公式の情報にも、区分を示すページは見つからなかった。

探した範囲: Tasks API のスコープのページ、`Sensitive scope verification`。後者は、Calendar の調査で、スコープごとの区分表が無いことを確かめたページ。

出典: [Tasks API のスコープ](https://developers.google.com/workspace/tasks/auth) ・ [Sensitive scope verification (Google for Developers)](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification)

### クォータ

- 1 日 50,000 クエリ (courtesy の上限)
- プロジェクトごとに異なる。引き上げは Cloud Console の「Quotas & System Limits」から申請できるが、承認は保証されず、大幅な増加は時間がかかることがある
- サービスアカウントの呼び出しは、単一のアカウントとして扱われる
- 1 分あたりの上限、エラーの種類と対処は、公式のこのページに書かれていなかった

探した範囲: 上のページと、毎分の上限・429 の検索。他の API (Health API・Admin SDK 等) の毎分の値は出たが、Tasks 固有の値を示す公式・非公式の情報は見つからなかった。

出典: [Tasks API の割り当てと使用量上限 (Google for Developers)](https://developers.google.com/workspace/tasks/limits)

[README へ戻る](../../README.md)
