# 開発メモ（引き継ぎ書）

このファイルは、作業の中断・再開をスムーズにするための開発メモ。セッションが変わっても
ここを読めば現状と次にやることが分かるようにする。詳細な実装の経緯より「今の構造がどうなっているか」を優先して書く。

**現状（2026-07-07時点）: `index.html` 実装済み・動作確認完了（タスク作成成功）。**
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
  - `description`（詳細、プレーンテキストのみ。Markdown未対応）
  - `status`（`todo` / `in_progress` / `done`）
  - `assigneeUid` / `assigneeName`（担当者。null許容＝未アサイン）
  - `dueDate`（期限。Firestore Timestamp、null許容）
  - `creatorUid` / `creatorName` / `createdAt` / `updatedAt`
- `task_users`（ドキュメントID＝uid）— ログイン時に `setDoc(merge)` で自分のプロフィール
  （`uid` / `displayName` / `email` / `updatedAt`）を自己登録し、担当者ドロップダウンの
  選択肢として全員分を読み取る（Wikiには存在しない新規コレクション）

## 中核機能（実装済み）

1. **タスクのCRUD＋担当者アサイン** — 作成・編集・削除、担当者を`task_users`から選択
2. **ステータス管理** — 未着手／進行中／完了の3ステータス。サイドバーでバッジ絞り込み
   （カンバン化はまだ未着手、必要になれば拡張）
3. **期限の可視化** — 期限日の設定と、期限超過（赤）／2日以内（黄）のバッジ表示、
   一覧は期限が近い順にソート。ホーム画面に「要注意」リストを表示。
   **プッシュ通知（ブラウザ通知やメール）は未実装** — 静的サイト+Firebaseだけでは実現コストが
   高い（FCM/Cloud Functionsの追加セットアップが必要）ため、当面はこの可視化のみで運用する方針
4. **権限方針** — ログイン済みなら誰でも編集・削除・ステータス変更可能
   （Wikiのタグ/フォルダと同じ「共同編集」ポリシー。担当者以外もステータス更新できる方が
   実用的なため）

## 重要な制約・注意点（Wikiプロジェクトからの申し送り）

- `file://` で直接開いてもGoogleログインはできない（OAuthポップアップはhttp/httpsオリジンが
  必須）。開発確認は `python -m http.server 8080` を起動し `http://localhost:8080/index.html`
  で行う
- 本番公開ドメインでログインさせるには、Firebase Console → Authentication → Settings →
  承認済みドメイン に該当ドメインを追加登録する必要がある
- Firestoreのセキュリティルールに `tasks` / `task_users` を追加登録するまではタスク作成が
  `permission-denied` で失敗する（2026-07-06に発覚、2026-07-07にルール追加で解決済み）
- ローカル動作確認は `python -m http.server` を **Wikiリポジトリとは別ポート**で起動すること。
  同じ8080で両方を試すと、後から起動した側がポート競合で立ち上がらず、先に起動していた
  Wiki側のサーバーの応答を誤って掴んでしまう（2026-07-06に実際に発生）。本プロジェクトは
  8082番などWikiと被らないポートを使う

## 次にやりたいこと（候補・未着手）

- コメント機能・ページ内目次など、コア機能が安定した後の拡張は要件が出てから検討
- 本番公開（GitHub Pages等）する場合は、公開ドメインをFirebaseの承認済みドメインに追加すること

## 再開時のチェックポイント

1. このファイルで現状把握（コア機能は実装・動作確認済み）
2. `git log` で直近のコミットを確認
3. 動作確認方法・承認済みドメインの注意点は上記「重要な制約・注意点」を参照
