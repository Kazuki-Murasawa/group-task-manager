# 開発メモ（引き継ぎ書）

このファイルは、作業の中断・再開をスムーズにするための開発メモ。セッションが変わっても
ここを読めば現状と次にやることが分かるようにする。詳細な実装の経緯より「今の構造がどうなっているか」を優先して書く。

**現状（2026-07-08時点）: `index.html` 実装済み・動作確認完了。テーブル型UI不具合修正、リスト⇔カンバン切り替え、
カテゴリ分け・担当者（どちらも事前登録制）まで実装・動作確認済み。**
**本番URL: https://kazuki-murasawa.github.io/group-task-manager/ （GitHub Pages、`main` ブランチのルートから配信）**

### 2026-07-08 実装・動作確認済み：カテゴリ・担当者を事前登録制に統一

タスクの「カテゴリ」「担当者」はどちらも自由入力ではなく **事前登録制**（グループで共有する
一覧から選ぶ方式）で統一した。実装がほぼ同一のため、共通ヘルパーにまとめてある。

- 新規Firestoreコレクション `categories`（`name` / `creatorUid` / `createdAt`）、
  `assignees`（同じ形）を追加。どちらもグループ全員で共有する一覧
- ツールバーに「カテゴリ管理」「担当者管理」ボタンを追加（`#/categories` / `#/assignees` ルート）。
  管理モーダルで追加・削除ができる（`window.addCategory` / `deleteCategory` / `addAssignee` / `deleteAssignee`）
- タスク作成・編集フォームのカテゴリ・担当者欄はどちらも登録済み一覧から選ぶセレクト
  （`renderNamedSelectOptions()`）
- 購読・追加・削除・フィルタ描画・管理モーダル描画のロジックはカテゴリ／担当者で共通化
  （`watchNamedList()` / `addNamedItem()` / `deleteNamedItem()` / `renderNamedFilterOptions()` /
  `renderManageModal()` など。`{ id, name }` の配列を扱う汎用関数として実装）
- カテゴリ・担当者を削除しても、既存タスク側は `category` / `assigneeName` に名前の文字列を
  そのまま保存している（非正規化）ため、削除後もタスク側の表示は残る
- ツールバーの絞り込みセレクト、テーブル行・ボードカード・詳細モーダルへのバッジ表示も対応済み
- **担当者を事前登録制にしたことに伴い、`task_users` コレクション関連（ログイン時の
  プロフィール自己登録・購読、`assigneeUid`）は使われなくなったため削除した**。
  今は担当者はログインの有無に関係なく任意の名前を登録・アサインできる
- **Firestoreセキュリティルールに `categories` / `assignees` の追加が必要**（下記「解決済み」節に追記）

### 2026-07-08 実装・動作確認済み：リスト⇔カンバン切り替え

- ツールバー右端に `[☰ リスト | ▦ ボード]` トグルを追加。選択状態は `localStorage`
  （キー: `gtm_view_mode`）に保存し次回訪問時も維持。ハッシュルーティングには含めない
- ボードは未着手／進行中／完了の3列（列ヘッダーに件数バッジ）。カードはタイトル・
  カテゴリバッジ・担当者・期限バッジを表示、クリックで既存の詳細モーダルを開く
- PCはドラッグ&ドロップ（HTML5 Drag and Drop API）で列間移動→ステータス更新。
  スマホはD&D非対応のため、カード内にも行内と同じステータスセレクトを併設
- 検索・担当者フィルタ・カテゴリフィルタ・KPIタイルの絞り込みは両表示で共通。
  ボード表示中は列自体がステータスを表すため、重複するステータスチップは非表示にする
- ステータス更新処理は `updateTaskStatus(id, status)` に共通化し、テーブルのセレクト・
  ボードのセレクト・D&D dropの3箇所から呼び出す

### 2026-07-08 コミット済み：不具合修正2件

UIリニューアル後の動作確認で見つかった不具合を修正（コミット `e739b76`）。

- **期限バッジの超過判定**: `dueDate` は期限日の0時で保存されるため、そのまま`Date.now()`と比較すると
  期限当日のうちから「超過」表示になってしまっていた。期限日が丸ごと過ぎてから（翌日0時以降）
  「超過」にするよう修正（`dueInfo()`内）
- **編集モーダルでの入力消失**: Firestoreの`onSnapshot`は他ユーザーの操作等でも発火するため、
  タスク編集モーダルを開いている間にスナップショットが更新されると`renderModal()`が
  `card.innerHTML`を丸ごと書き直し、入力中の内容が消えてしまっていた。同じフォームを表示し
  続けている間は再描画をスキップし、担当者リスト（`refreshAssigneeOptions()`）だけを
  最新化する方式に変更

### 新UIの構成（2026-07-07リニューアル、動作確認済み）

- KPI行（未着手/進行中/完了/要注意、クリックで絞り込み）、ツールバー（検索・チップ・担当者フィルタ）、
  テーブル4列（タスク/ステータス/担当者/期限）、詳細・編集・新規はモーダル（ルーティングは従来通り
  `#/task/<id>` / `#/new`）、720px以下でカード表示に切替
- バッジ配色はWCAGコントラスト4.5:1以上に調整済み（ライトモードの文字色を暗く変更：
  danger-text #bb2033 / warning-text #8f5e05 / success-text #0f6e46 / muted #5b6472、
  ダークのprimary-text #84acff）。Firebaseロジック・ルーティングは変更なし
姉妹プロジェクトの社内Wiki（`Kazuki-Murasawa.github.io`リポジトリ）と同じ技術スタック・
実装ノウハウを流用して構築した。Firebaseプロジェクトは新規作成せずWikiと共用（`my-sync-todo`）。

### ✅ 解決済み（2026-07-07）：Firestoreセキュリティルールの追加

2026-07-06にタスク作成が **`permission-denied`** エラーで失敗していた件は、Firebase Console →
Firestore Database → 「ルール」タブに以下のルールを追加・公開して解決した（タスク作成の成功を
確認済み）。ルールはFirebase Console側でのみ管理されており、このリポジトリには
ルールファイルが存在しない点に注意。現在公開中のルールは既存の `wiki_pages` / `wiki_folders` に
加えて以下の2ブロック：

```
match /tasks/{taskId} {
  allow read: if request.auth != null;
  allow create: if request.auth != null;
  allow update, delete: if request.auth != null;
}

match /task_users/{uid} {
  allow read: if request.auth != null;
  allow write: if request.auth != null && request.auth.uid == uid;
}
```

### ✅ 解決済み（2026-07-08）：`categories` / `assignees` コレクションのセキュリティルール追加

カテゴリ・担当者の事前登録制対応に伴い、上記2ブロックに加えて以下を追加・公開済み（追加前は
それぞれの追加・削除が `permission-denied` で失敗していた）：

```
match /categories/{categoryId} {
  allow read: if request.auth != null;
  allow create, delete: if request.auth != null;
}

match /assignees/{assigneeId} {
  allow read: if request.auth != null;
  allow create, delete: if request.auth != null;
}
```

なお `task_users` のルールはアプリ側で使わなくなったが、Firestore Console側のルールは
そのまま残っている（実害はないため未削除。整理したい場合はConsole側で手動削除）。


## プロジェクト概要

- グループ（チーム）内で使うタスク管理ツール
- Firebaseを使い、`index.html` 1ファイル構成で実装する（Wikiプロジェクトと同一方針）
- 対象は少人数の身内グループでの共同タスク管理。マルチテナント（グループごとの分離）は
  現時点ではスコープ外。ログインした全員が同じタスク一覧を共有する（Wikiの「全員の投稿が
  同じFeedに載る」のと同じ考え方）

## 技術構成（Wikiプロジェクトを流用）

- Firebase Authentication（Googleログイン）＋ Firestore（リアルタイムDB）
- `index.html` 単一ファイル、`<script type="module">` でロジックを書く
- Markdownライブラリ（`marked`）は今回のタスク管理では必須ではない。もしタスクの詳細説明欄で
  Markdown対応する場合のみ、Wikiと同じ理由（CORS制限回避）で `<head>` 内の通常
  `<script src="...">` によるグローバル読み込み＋ `const marked = window.marked;` の
  ブリッジパターンを流用する
- Firestoreのコレクションはリアルタイム購読（`onSnapshot`）を基本方針とする。取得データは
  モジュール変数にキャッシュし、画面状態（フィルタ条件・編集中タスクなど）もモジュール変数で
  管理する（Wikiの `currentPages` 相当）
- ルーティングは `location.hash` で管理し、タスク詳細URLを直接共有・ブックマークできるように
  する（Wikiの `#/page/<id>` 相当。今回は `#/task/<id>` を想定）

## コレクション設計（実装済み）

- `tasks`
  - `title`（タスク名）
  - `category`（カテゴリ名の文字列。`categories`コレクションから選択、null許容＝未分類。
    非正規化しているため、カテゴリ削除後もタスク側の表示はそのまま残る）
  - `description`（詳細、プレーンテキストのみ。Markdown未対応）
  - `status`（`todo` / `in_progress` / `done`）
  - `assigneeName`（担当者名の文字列。`assignees`コレクションから選択、null許容＝未アサイン。
    カテゴリと同じ非正規化方式。ログインの有無に関係なく任意の名前を登録・アサインできる）
  - `dueDate`（期限。Firestore Timestamp、null許容）
  - `creatorUid` / `creatorName` / `createdAt` / `updatedAt`
- `categories`（ドキュメントID＝自動採番）— `name` / `creatorUid` / `createdAt`。
  グループで共有するカテゴリの事前登録リスト。「カテゴリ管理」モーダル（`#/categories`）
  から追加・削除する
- `assignees`（ドキュメントID＝自動採番）— `categories`と同じ形（`name` / `creatorUid` / `createdAt`）。
  グループで共有する担当者名の事前登録リスト。「担当者管理」モーダル（`#/assignees`）から追加・削除する
- ~~`task_users`~~（廃止・2026-07-08）— 旧実装ではログイン時に自分のプロフィールを自己登録し、
  担当者ドロップダウンの選択肢にしていたが、担当者を`assignees`による事前登録制に変更したため
  アプリ側では使わなくなった。Firestore Console側のルール・既存データは残存

## 中核機能（実装済み）

1. **タスクのCRUD＋担当者アサイン** — 作成・編集・削除、担当者は`assignees`コレクションに
   事前登録した名前から選択（ログイン不要な人も担当者にできる）
2. **ステータス管理** — 未着手／進行中／完了の3ステータス。KPIタイル・ステータスチップで絞り込み。
   リスト表示とカンバンボード表示を切り替え可能（トグルはツールバー右端、選択状態は
   `localStorage`に保存）
3. **カテゴリ分け** — `categories`コレクションに事前登録したカテゴリから1つを選んでタスクに設定。
   絞り込み・バッジ表示に対応（詳細は上記「2026-07-08 実装・動作確認済み」節）
4. **期限の可視化** — 期限日の設定と、期限超過（赤）／2日以内（黄）のバッジ表示、
   一覧は期限が近い順にソート。ホーム画面に「要注意」リストを表示。
   **プッシュ通知（ブラウザ通知やメール）は未実装** — 静的サイト+Firebaseだけでは実現コストが
   高い（FCM/Cloud Functionsの追加セットアップが必要）ため、当面はこの可視化のみで運用する方針
5. **権限方針** — ログイン済みなら誰でも編集・削除・ステータス変更可能
   （Wikiのタグ/フォルダと同じ「共同編集」ポリシー。担当者以外もステータス更新できる方が
   実用的なため）

## 重要な制約・注意点（Wikiプロジェクトからの申し送り）

- `file://` で直接開いてもGoogleログインはできない（OAuthポップアップはhttp/httpsオリジンが
  必須）。開発確認は `python -m http.server 8080` を起動し `http://localhost:8080/index.html`
  で行う
- 本番公開ドメインでログインさせるには、Firebase Console → Authentication → Settings →
  承認済みドメイン に該当ドメインを追加登録する必要がある
- Firestoreのセキュリティルールに各コレクション（`tasks` / `categories` / `assignees`）を
  追加登録するまでは、そのコレクションへの書き込みが `permission-denied` で失敗する
  （`tasks`は2026-07-06に発覚、2026-07-07にルール追加で解決済み。`categories` / `assignees`は
  2026-07-08にルール追加で解決済み）
- ローカル動作確認は `python -m http.server` を **Wikiリポジトリとは別ポート**で起動すること。
  同じ8080で両方を試すと、後から起動した側がポート競合で立ち上がらず、先に起動していた
  Wiki側のサーバーの応答を誤って掴んでしまう（2026-07-06に実際に発生）。本プロジェクトは
  8082番などWikiと被らないポートを使う

## 次にやりたいこと（候補・未着手）

- コメント機能・ページ内目次など、コア機能が安定した後の拡張は要件が出てから検討

## 再開時のチェックポイント

1. このファイルで現状把握（コア機能は実装・動作確認済み）
2. `git log` で直近のコミットを確認
3. 動作確認方法・承認済みドメインの注意点は上記「重要な制約・注意点」を参照
