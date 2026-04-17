# Accord - Claude Code Context

## プロジェクト概要

| 項目 | 内容 |
|------|------|
| プロジェクト名 | Accord |
| GitHub | https://github.com/ieneko1122/Accord |
| 開発者 | ieneko1122 |
| 目的 | 練習・試作品 |

### コンセプト
> Discord（不和）の対義語。  
> PCゲーマー向けのマッチングサービス。  
> ゲームの趣味でマッチングして、Discordフレンドを増やす。  
> チャットはDiscordに任せ、このアプリはマッチングに特化する。  
> **このアプリはあくまでDiscordへの「導線」である。**

---

## 技術スタック

### バックエンド（`/backend`）
| 役割 | 技術 |
|------|------|
| フレームワーク | Spring Boot 3.x |
| 認証 | Spring Security OAuth2 Client（Discord） |
| DB | H2（試作用）→ 後でPostgreSQLに移行可 |
| テスト | JUnit 5 + Mockito + AssertJ |
| ビルドツール | Gradle |
| 環境変数 | spring-dotenv（.envファイル） |

### フロントエンド（`/frontend`）
| 役割 | 技術 |
|------|------|
| フレームワーク | Vue 3（Composition API） |
| ビルドツール | Vite |
| ルーティング | Vue Router |
| 状態管理 | Pinia |
| HTTP通信 | Axios |
| UIライブラリ | Bootstrap 5 |

### 外部API
| API | 用途 |
|-----|------|
| Discord OAuth2 | ログイン・DiscordID・アバター自動取得 |
| IGDB API | ゲーム検索・公式名・カバー画像取得（表記ゆれ防止） |

---

## フォルダ構成

```
accord/
├── CLAUDE.md
├── requirements.md        # 要件定義書（詳細はこちら）
├── backend/               # Spring Boot REST API
│   ├── src/main/java/com/github/ieneko1122/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── domain/model/
│   │   └── config/
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── .env               # 環境変数（gitignore済み）
│   └── build.gradle
└── frontend/              # Vue 3 SPA
    ├── src/
    │   ├── components/
    │   ├── views/
    │   ├── stores/        # Pinia
    │   ├── router/        # Vue Router
    │   └── main.js
    └── vite.config.js
```

---

## 環境構築手順

### 前提条件
- Java 21 LTS（`C:\Program Files\Java\jdk-21.0.10`）
- Node.js
- Git Bash

### Java 21に切り替え（毎回Git Bash起動時に実行）
```bash
export JAVA_HOME="/c/Program Files/Java/jdk-21.0.10"
export PATH="$JAVA_HOME/bin:$PATH"
```

### バックエンド起動
```bash
cd backend
./gradlew bootRun
```

### フロントエンド起動（別ターミナル）
```bash
cd frontend
npm run dev
```

### .envファイル作成（新しいPCでの初回のみ）
```
backend/.env に以下を作成：
DISCORD_CLIENT_ID=（Discord開発者ポータルから取得）
DISCORD_CLIENT_SECRET=（Discord開発者ポータルから取得）
IGDB_CLIENT_ID=（後で設定）
IGDB_CLIENT_SECRET=（後で設定）
```

### Discord開発者ポータル設定
```
https://discord.com/developers/applications
→ Accord アプリ
→ OAuth2 → Redirects：
   http://localhost:8080/login/oauth2/code/discord
```

---

## 重要な設計上の決定事項

### 認証
- メールアドレス登録なし・Discord OAuthのみ
- DiscordID・アバターはOAuthで自動取得
- パスワード不要

### チャット
- チャット機能なし（削除済み）
- マッチング成立後はDiscordIDを開示してDiscordへ誘導

### マッチングロジック
- お互いにいいね → マッチング成立
- いいね取り消し：マッチング前のみ可能
- 再いいね：可能
- マッチング解除：試作品はなし・本番はあり
- 「Discordで繋がった！」ボタン → matches.is_connected=TRUE → 一覧から消える

### ゲーム検索（目玉機能）
- IGDB API連携で公式ゲーム名を取得
- 表記ゆれ完全防止
- ゲームカバー画像も取得
- 入力時debounce 300ms

### 検索・絞り込み
- ゲームタグ複数選択AND・プレイ時間帯複数選択OR
- 自分自身は一覧から除外

### DiscordID開示
- プロフィール詳細画面でマッチング状態をAPI側で判定
- マッチング済みのみDiscordIDを返す（セキュリティ）

---

## 画面一覧（6画面）

| # | 画面名 | 主な目的 |
|---|--------|---------|
| 1 | ログイン | Discordログイン |
| 2 | プロフィール設定 | 初回登録・編集（使い回し） |
| 3 | ユーザー一覧 | 仲間を探していいねを送る |
| 4 | プロフィール詳細 | 相手の詳細（マッチング状態で表示変化） |
| 5 | 受け取ったいいね | いいねを返してマッチングを成立させる |
| 6 | マッチング一覧 | DiscordIDを確認・「繋がった！」ボタン |

---

## データモデル（テーブル一覧）

```
users          : discord_id, last_login_at
profiles       : username, avatar_url, bio, play_style, party_style, skill_level
play_times     : profile_id, time_slot（MORNING/AFTERNOON/NIGHT/LATE_NIGHT）
games          : igdb_id, name, cover_url（IGDBから取得）
profile_games  : profile_id, game_id（中間テーブル）
likes          : from_user_id, to_user_id
matches        : user1_id, user2_id, is_connected
```

---

## 現在の開発状況

### 完了
- [x] 要件定義
- [x] GitHubリポジトリ作成
- [x] Spring Bootバックエンド初期構築
- [x] Vue 3フロントエンド初期構築
- [x] Discord OAuth2設定・動作確認
- [x] application.yml設定
- [x] .env設定

### 次のステップ
- [ ] Discord OAuthコールバック処理・ユーザー保存
- [ ] Entityクラス・Repository作成
- [ ] IGDB APIアカウント登録・動作確認
- [ ] Vue Router・Piniaの初期設定
- [ ] 各画面の実装

---

## 将来的な拡張（試作品以降）

- プロフィールに長い自己紹介欄
- プロフィール設定の完了率表示
- ユーザー一覧の並び順変更
- ブロック機能
- マッチング解除機能
- Steam連携
- 通報機能
- 検索の関連度スコアリング
- DB を PostgreSQL に移行
- デプロイ（Render / Railway など）
