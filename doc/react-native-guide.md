# React Native 画面実装ガイド（UI のみ）

[frontend-design.md](./frontend-design.md) の画面・見た目・遷移を、React Native（Expo / **iOS のみ**）の**フロントコード**としてどう書くかのまとめ。

**本ドキュメントの範囲**

- 画面レイアウト、コンポーネント、ナビ、スタイル
- 一覧の UI 挙動（スクロール、pull-to-refresh の置き方）
- ローディング / 空 / エラーの**見た目**

**対象外（書かない）**

- API / Firebase / データ取得
- TanStack Query / Zustand などの状態ライブラリ選定
- バックエンド連携

表示用のデータは、当面 **モック（固定配列）** で十分。後から差し替える前提で、画面は「渡されたデータを描画する」だけにする。

関連: [frontend-design.md](./frontend-design.md) / [coding-standards.md](./coding-standards.md)

---

## 1. 使うもの（UI に必要な範囲）

| 用途 | 採用 |
| --- | --- |
| アプリ | Expo + React Native + TypeScript |
| 画面遷移 | React Navigation（Native Stack + Bottom Tabs） |
| スタイル | `StyleSheet` + `theme/` |
| 画像表示 | `Image` / `expo-image` |
| 画像選択 UI | `expo-image-picker`（ピッカーを開くまで。アップロード処理は対象外） |
| セーフエリア | `react-native-safe-area-context` |
| 対象 OS | **iOS のみ** |

---

## 2. フォルダ構成（画面だけ）

```text
src/
  app/                 # エントリ、SafeArea / Navigation の組み立て
  navigation/          # AuthStack / MainTabs / 各 Stack
  features/
    auth/screens/
    home/screens/
    posts/screens/
    notifications/screens/
    profile/screens/
    dm/screens/
    search/screens/
    settings/screens/
  components/          # PostCard、Avatar、EmptyState、ErrorState など
  theme/               # colors, spacing, typography
  mocks/               # 画面確認用のダミーデータ（任意）
```

- 画面 → `features/*/screens`
- 使い回す UI → `components`
- 色・余白 → `theme`（画面にカラーコード直書きしない）

---

## 3. 基本の書き方

### 3.1 画面コンポーネント

```tsx
// features/home/screens/HomeScreen.tsx
import { View, StyleSheet } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

export function HomeScreen() {
  const insets = useSafeAreaInsets();

  return (
    <View style={[styles.container, { paddingTop: insets.top }]}>
      {/* タイムライン UI */}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
});
```

- 関数コンポーネント + TypeScript
- スタイルは `StyleSheet.create`、色は `theme` 参照

### 3.2 Web との対応

| Web | React Native |
| --- | --- |
| `div` | `View` |
| `span` / `p` | `Text`（文字は必ず `Text` 内） |
| `img` | `Image` |
| `button` | `Pressable` |
| `input` | `TextInput` |
| CSS | `StyleSheet`（デフォルトで縦方向 flex） |
| `onClick` | `onPress` |
| スクロール | 短い画面は `ScrollView`、一覧は `FlatList` |

---

## 4. ナビゲーション（画面遷移）

`frontend-design.md` の遷移イメージどおり、**画面をつなぐだけ**。

```text
RootNavigator
├── AuthStack
│   ├── Splash
│   ├── Login
│   ├── SignUp
│   ├── ResetPassword
│   └── ProfileSetup
└── MainTabs
    ├── HomeStack（Home / PostDetail / OtherProfile）
    ├── SearchStack
    ├── NotificationsStack
    └── ProfileStack（MyProfile / EditProfile / Settings / FollowList）

モーダル or 別 Stack:
  ComposePost / ImagePreview / DmStack（一覧 → トーク）
```

- タブ: `@react-navigation/bottom-tabs`
- 詳細: `@react-navigation/native-stack`
- 投稿作成は `presentation: 'modal'` でも Stack でも可（方針は1つに）
- iOS: ヘッダー戻る + 端スワイプ戻る

```tsx
export type HomeStackParamList = {
  Home: undefined;
  PostDetail: { postId: string };
  OtherProfile: { userId: string };
};
```

通知行タップ → `PostDetail` に `postId` を渡す、までをナビの型に含めておく（実際の通知データ取得は対象外）。

---

## 5. 一覧 UI（ホーム / 通知 / DM）

`frontend-design.md` の「無限スクロール」「pull-to-refresh」は、**リスト部品の props** として用意する。中身の取得処理は書かない。

```tsx
const MOCK_POSTS = [
  { id: '1', displayName: 'Riku', username: 'riku_notes', body: 'こんにちは', liked: false },
  // ...
];

<FlatList
  data={MOCK_POSTS}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <PostCard post={item} />}
  onEndReached={() => {
    // UI 用: 後で「続きを読む」処理を差し込む枠だけ残す
  }}
  onEndReachedThreshold={0.4}
  refreshing={false}
  onRefresh={() => {
    // UI 用: 引っ張ったときの見た目確認用
  }}
  ListEmptyComponent={<EmptyState message="まだ投稿がありません" />}
  ListFooterComponent={null}
/>
```

| デザイン要件 | UI での対応 |
| --- | --- |
| 下までスクロールで続き | `onEndReached` を置く |
| 上に引っ張って更新 | `refreshing` / `onRefresh` |
| 0件 | `ListEmptyComponent` |
| 追加読み込み中の見た目 | `ListFooterComponent` に小さな Loading |

長い一覧は `ScrollView` + `map` にしない。`FlatList` を使う。

---

## 6. 画面ごとの UI メモ

### 認証

- `TextInput` でメール / パスワード欄
- 送信ボタン押下中の **disabled + スピナー見た目**
- 「パスワードを忘れた」→ `ResetPassword` 画面へのリンク
- 実際のログイン処理は書かない（ボタン `onPress` は画面遷移や console で可）

### ホーム

- `FlatList` + `PostCard`
- カード: アイコン、表示名、ID、時刻、本文、画像、アクション列（返信 / 再投稿相当 / いいね / 共有）
- アクションは `Pressable`。押したときの見た目（色・塗りつぶし）まで作る

### 投稿作成

- `TextInput`（`multiline`）+ 文字数カウンター表示
- 画像追加ボタン → ピッカーで選んだ画像を画面にプレビュー表示するまで
- `KeyboardAvoidingView`（iOS: `behavior="padding"`）
- 投稿ボタンの enabled / disabled（文字数・枚数オーバー時の見た目）

### 投稿詳細

- 上: 元投稿、下: コメント一覧 `FlatList`
- 返信入力欄のレイアウト
- スレッドの見た目（インデントなど）はデザイン寄せで調整

### 通知

- 通知行コンポーネント + `FlatList`
- タップで `navigation.navigate('PostDetail', { postId })`
- タブの未読バッジは `tabBarBadge` の見た目確認（数字は仮でよい）

### プロフィール

- ヘッダー画像・bio・フォロー数は `ListHeaderComponent`
- 投稿一覧は同じ `FlatList`（入れ子 Scroll を避ける）

### DM

- 一覧: スレッド行の `FlatList`
- トーク: 吹き出し一覧 + 下部入力バー
- `KeyboardAvoidingView` + 下部セーフエリア余白

### 設定

- `ScrollView` + 設定行（ラベル、スイッチ、右矢印）
- 利用規約などは行タップの見た目だけ（`Linking` は後で可）
- ブロック / ミュート導線の入口 UI

---

## 7. 共通 UI（画面ごとに必ず用意）

`frontend-design.md` のとおり、正常系以外もセットで作る。

| 状態 | コンポーネント例 |
| --- | --- |
| 読み込み中 | `LoadingState`（スピナー or スケルトン） |
| 0件 | `EmptyState`（文言 + 任意 CTA） |
| エラー | `ErrorState`（「再試行」ボタンの見た目） |

画面切り替えは、当面フラグや story 用 props で切り替えて確認すればよい。

```tsx
type Props = { state?: 'loading' | 'empty' | 'error' | 'ready' };

export function HomeScreen({ state = 'ready' }: Props) {
  if (state === 'loading') return <LoadingState />;
  if (state === 'error') return <ErrorState onRetry={() => {}} />;
  // FlatList（empty は ListEmptyComponent）
}
```

その他共通:

- 下部タブナビ
- 投稿作成ボタン（FAB or タブ中央）
- アクションシート（削除・共有などのメニュー見た目）
- 画像プレビュー（全画面モーダル）

---

## 8. iOS 向け UI

| 注意点 | 画面側でやること |
| --- | --- |
| ノッチ / ホームインジケーター | `SafeAreaProvider` + insets |
| 戻る | Stack ヘッダーの戻るを使う |
| キーボード | 投稿作成・DM で `KeyboardAvoidingView` |
| 通知許可ダイアログ | 文言・「設定アプリへ」誘導画面の UI だけ先に用意可 |
| ライト / ダーク | `useColorScheme()` + theme の2パレット |

```tsx
import { SafeAreaProvider } from 'react-native-safe-area-context';

export function App() {
  return (
    <SafeAreaProvider>
      <NavigationContainer>{/* ... */}</NavigationContainer>
    </SafeAreaProvider>
  );
}
```

---

## 9. テーマ（後差し替え用）

```tsx
// theme/colors.ts
export const colors = {
  light: {
    background: '#FFFFFF',
    text: '#0F1419',
    border: '#EFF3F4',
    primary: '#1D9BF0', // 仮
  },
  dark: {
    background: '#000000',
    text: '#E7E9EA',
    border: '#2F3336',
    primary: '#1D9BF0',
  },
};
```

`spacing` / `typography` も同様に集約。ブランド・文言は後から変える前提。

---

## 10. MVP で先に作る UI 順

1. ナビ骨格（Auth / MainTabs）の枠
2. ログイン・登録の**見た目**
3. ホーム + `PostCard` + モック `FlatList`
4. 投稿作成画面のレイアウト
5. 投稿詳細のレイアウト
6. 通知一覧のレイアウト
7. プロフィールのレイアウト
8. DM 一覧・トークのレイアウト
9. 設定、空 / エラー / ローディングの見た目揃え

検索本格 UI・再投稿の複雑挙動・プッシュの作り込みは後回し。

---

## 11. やらないこと

- 画面から API / Firebase を呼ぶコードを書く
- データ取得ライブラリの使い方をこのガイドに増やす
- Android 分岐
- 正常系だけの画面（空・エラー・ローディング無し）
- 色のベタ書き乱立
- 長い一覧を `ScrollView` + `map`

---

## 12. 関連

- [frontend-design.md](./frontend-design.md) … 画面・デザイン要件（本ガイドの元）
- [coding-standards.md](./coding-standards.md) … 規約・PR
