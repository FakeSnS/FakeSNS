# コーディング規約・開発運用ルール v0.1

## 目的

React Native/Expo/Firebaseで開発する架空SNSアプリを、安全に継続開発するための規約。

品質、レビュー、ブランチ運用、テスト、リリース手順を明確にし、機能追加時の事故を減らす。

**対象プラットフォームは iOS のみ**（Android は対象外）。

## 技術スタック

- React Native（iOS のみ）
- Expo
- TypeScript
- Firebase
  - Auth
  - Firestore
  - Storage
  - Cloud Functions
  - FCM
- React Navigation
- Zustand
- TanStack Query
- Jest
- React Native Testing Library
- ESLint
- Prettier

## ブランチ運用

### 基本ブランチ

- main: 本番相当。直接push禁止。
- develop: 開発統合ブランチ。
- feature/*: 機能追加。
- fix/*: バグ修正。
- chore/*: 設定、依存関係、CIなど。
- refactor/*: 振る舞いを変えない内部改善。
- docs/*: ドキュメント更新。

### ブランチ命名

形式:

```text
feature/short-description
fix/short-description
chore/short-description
```

例:

```text
feature/post-composer
feature/bot-notifications
feature/firebase-auth
fix/image-upload-error
docs/requirements-v02
```

## マージルール

- mainへの直接pushは禁止。
- developへの直接pushは禁止。
- すべてPull Request経由でマージする。
- PRは最低1人以上のApproveが必要。
- PR作成者本人のApproveはカウントしない。
- CIが成功していないPRはマージ禁止。
- レビュー未対応のコメントが残っているPRはマージ禁止。
- 仕様変更を含むPRは要件定義または関連ドキュメントを更新する。
- AIエージェントによるcommitは禁止。
- AIエージェントによるpushは禁止。
- commit/pushは人間の開発者が差分を確認したうえで実行する。

GitHub branch protectionで設定する項目:

- Require a pull request before merging
- Require approvals: 1以上
- Dismiss stale pull request approvals when new commits are pushed
- Require status checks to pass before merging
- Require branches to be up to date before merging
- Require conversation resolution before merging
- Do not allow bypassing the above settings

## PRルール

PR本文には以下を書く。

```md
## 概要

## 変更内容

## 動作確認

## 影響範囲

## スクリーンショット

## 未対応・懸念
```

スクリーンショットが必要なPR:

- 画面追加
- UI変更
- 投稿、通知、プロフィールなど主要導線の変更
- ダークモード/ライトモードに影響する変更

## コミットルール

コミットメッセージは必ず日本語で書く。

Conventional Commitsの種別は使ってよいが、説明文は日本語にする。

AIエージェントはコミットメッセージ案の提示までに留め、実際の`git commit`と`git push`は実行しない。

例:

```text
feat: 投稿作成画面を追加
fix: 画像アップロード失敗時の処理を修正
docs: Firebase前提の要件定義に更新
chore: ESLint設定を追加
refactor: 通知生成処理を分離
test: BOTイベント生成のテストを追加
```

禁止例:

```text
feat: add post composer screen
fix image upload
update docs
```

## TypeScript規約

- anyは禁止。必要な場合は理由をコメントに残す。
- 型はできるだけドメイン名で表現する。
- API/Firebaseから取得したデータは型変換層を通す。
- undefined/nullの扱いを曖昧にしない。
- 画面コンポーネントにFirebase処理を直書きしない。

推奨構成:

```text
src/
  app/
  components/
  features/
    posts/
    comments/
    notifications/
    bots/
    auth/
  lib/
    firebase/
  navigation/
  theme/
  types/
  utils/
```

## React Native規約

- 画面単位のコンポーネントは`features/*/screens`に置く。
- 汎用UIは`components`に置く。
- 1ファイルが大きくなりすぎる場合は、表示、状態、データ取得を分離する。
- スタイル値はできるだけthemeから参照する。
- 文字列は後から多言語化できるよう、主要文言を集約しやすい構成にする。
- 画像選択、通知、権限リクエストは専用モジュールに分離する。

## Firebase規約

- Firestoreのパスは定数化する。
- セキュリティルールを前提に実装する。
- クライアントだけを信頼してカウント更新しない。
- 投稿数、いいね数、コメント数などの集計はCloud Functionsで補正できる設計にする。
- BOT生成・通知生成は最終的にCloud Functionsへ寄せる。
- StorageのアップロードパスはユーザーID/投稿ID単位で整理する。

例:

```text
users/{userId}
posts/{postId}
posts/{postId}/comments/{commentId}
users/{userId}/notifications/{notificationId}
bot_events/{eventId}
```

## BOT実装規約

- BOTは`users`上では`is_bot: true`で区別する。
- 実在人物、著名人、ブランドになりすます名前は禁止。
- BOT名、ID、アイコン、bio、口調は生成後に固定する。
- BOT反応はユーザー体験を壊さない範囲で段階的に発生させる。
- BOTコメントテンプレートは不快、攻撃、性的、差別的な内容を含めない。
- AI生成を使う場合は安全フィルタとログ確認を入れる。

## 通知実装規約

- 通知は必ずFirestoreに保存する。
- 未読/既読を管理する。
- 通知タイプと強度をenum相当で管理する。
- 通知文言はテンプレート化する。
- 同じ内容の通知を短時間に大量作成しない。
- プッシュ通知はアプリ内通知と別レイヤーとして扱う。

## テスト方針

最低限必要なテスト:

- 投稿作成
- 画像投稿
- コメント作成
- いいね処理
- 通知生成
- BOTイベント生成
- Firebaseデータ変換

PR前に実行するコマンド:

```text
npm run lint
npm run typecheck
npm test
```

UI変更がある場合:

```text
npm run start
```

iOS シミュレータまたは iOS 実機で確認する。

## レビュー観点

- 要件とズレていないか
- iOS（シミュレータ／実機）で動くか
- Android 向け分岐や不要なクロスプラットフォーム対応を増やしていないか
- Firebaseの読み書き回数が過剰でないか
- 通知が過剰に発火しないか
- BOTが実在人物に見えないか
- エラー時の表示があるか
- ローディング/空状態があるか
- テストが必要な箇所に追加されているか
- セーフエリア（ノッチ／ホームインジケーター）を考慮しているか

## 禁止事項

- main/developへの直接push
- レビューなしマージ
- AIエージェントによるcommit
- AIエージェントによるpush
- anyの乱用
- Firebase処理の画面直書き
- 実在人物風BOTの作成
- X/Twitterのロゴ、名称、配色、アイコンをそのまま使うこと
- ユーザーの投稿画像を無制限にアップロードすること
- セキュリティルール未整備のまま本番公開すること

## Definition of Done

機能は以下を満たしたら完了とする。

- 要件を満たしている
- iOS（シミュレータまたは実機）で確認済み
- lint/typecheck/testが通っている
- 必要なスクリーンショットがPRにある
- Firebaseのデータ構造に破綻がない
- エラー、ローディング、空状態がある
- 1人以上のApproveがある
- CIが成功している
