# 要件定義書 - Accord

## 1. プロジェクト概要

| 項目 | 内容 |
|------|------|
| プロジェクト名 | Accord |
| 目的 | 練習・試作品 |
| 開発人数 | 1人 |
| ターゲット | PCゲームをやってる若者 |

### コンセプト
> Discordは既存の友達・コミュニティが前提。  
> このアプリは **ゲームの趣味でマッチングして、Discordフレンドを増やす** のが目的。  
> チャットはDiscordに任せ、このアプリはマッチングに特化する。  
> **このアプリはあくまでDiscordへの「導線」である。**

---

## 2. ターゲットユーザー

- PCゲームをプレイしている若者
- 新しいゲーム仲間を探したい人
- Discordは使っているが新しい出会いの場がない人

---

## 3. 機能要件（MVP）

### Sprint 1 - ユーザー認証
- [ ] Discordアカウントでログイン（OAuth2）
- [ ] ログアウト（サーバーセッション破棄 + フロントPiniaクリアの両方）
- [ ] セッション管理（HttpOnly Cookie 方式、詳細は 4.非機能要件を参照）
- [ ] Discord snowflake ID・`@username`・`global_name`・`avatar_hash` の自動取得（ログインのたびに最新値で更新）

### Sprint 2 - プロフィール
- [ ] プロフィール作成・編集
  - ユーザー名（Discord `global_name` → 無ければ `username` を初期値に）
  - 好きなゲーム（タグ・複数選択、IGDBサジェスト付き）
  - プレイ時間帯（朝 / 昼 / 夜 / 深夜、複数選択可）
  - ボイチャ派 / テキスト派 / どちらでも（BOTH）
  - ソロ / 固定パーティ募集 / どちらでも（BOTH）
  - スキルレベル（初心者 / 中級者 / 上級者）
  - 一言コメント
- [ ] IGDB APIラッパー＋キャッシュ（Caffeine・`games` テーブルを経由・4req/sec対策）

### Sprint 3 - ユーザー探し・いいね
- [ ] 他のユーザー一覧表示（最終ログイン順・自分除外・ページング必須）
- [ ] ゲームタイトルで絞り込み（複数タグAND）
- [ ] プレイ時間帯で絞り込み（複数選択OR）
- [ ] プレイスタイル（ボイチャ／テキスト／BOTH）で絞り込み
- [ ] ユーザープロフィール詳細表示（`isLikedByMe` / `isLikedMe` / `isMatched` フラグを含む）
- [ ] いいね送信・取り消し
- [ ] 受け取ったいいね一覧表示（ページング必須）
- [ ] いいねを返す（マッチング判定）

### Sprint 4 - マッチング
- [ ] お互いにいいねしたらマッチング成立（`user1_id < user2_id` で正規化）
- [ ] マッチング一覧ページ（ページング必須）
- [ ] マッチング成立後のみ相手の `@username` を開示（コピーボタン付き）
- [ ] 「Discordで繋がった！」ボタンで **自分側のみ** 一覧から削除（相手側の表示は維持）
- [ ] 未読バッジ（受信いいね・新規マッチ件数）をナビゲーションに表示

### Sprint 5 - テスト・調整
- [ ] バグ修正
- [ ] UIの調整
- [ ] テストコード整備

---

## 4. 非機能要件

| 項目 | 内容 |
|------|------|
| 対象プラットフォーム | Webブラウザ（PC）※モバイルは Non-goal だが、Bootstrap 5 のレスポンシブで壊れない程度に留意 |
| 同時接続 | 試作品のため考慮しない |
| 認証方式 | Discord OAuth2（認可コードフロー） |
| セッション | HttpOnly Cookie セッション（`SameSite=Lax`）。開発環境では Vite の `server.proxy` でフロント・API を同一オリジン化し、本番は同ドメイン配信 or `SameSite=None; Secure` |
| Spring Security | `SessionCreationPolicy.IF_REQUIRED`。未認証は 401、認証済だがプロフィール未作成の API 呼び出しは 403（画面側でプロフィール設定画面へリダイレクト） |
| CSRF | Spring Security デフォルトの CSRF トークン方式（SPA からは `XSRF-TOKEN` Cookie 経由で `X-XSRF-TOKEN` ヘッダ送信） |
| ログアウト | サーバー側セッション破棄 + フロント側 Pinia state クリアの **両方** を実施。Discord 側 refresh_token の revoke は将来対応 |
| CORS | 本番は同オリジン配信を目指すため原則不要。開発時は Vite proxy で迂回。直接クロスオリジン配信する場合のみ allowlist で `credentials: true` 許可 |
| ページング | 一覧系 API（ユーザー一覧／受信いいね／マッチング一覧）は `?limit=20&cursor=<id>` 必須。`limit` デフォルト 20・最大 50 |
| IGDB レート対策 | バックエンドで Caffeine キャッシュ（検索結果 15分）+ `games` テーブル自体を永続キャッシュとして利用（4 req/sec 超過防止）。Twitch OAuth トークンはリフレッシュ対応 |
| H2 の運用 | 試作中もファイルモードで動かす（`jdbc:h2:file:./data/accord;AUTO_SERVER=TRUE`）。再起動でデータが消えないこと |
| 環境変数 | 開発は `.env`（spring-dotenv）、本番は PaaS の環境変数（`application.yml` の `${...}` で両対応） |
| プロフィール未設定時 | ログイン後プロフィール未設定の場合はプロフィール設定画面に強制リダイレクト |

---

## 5. 技術スタック

**バックエンド（REST API）**

| 役割 | 技術 |
|------|------|
| フレームワーク | Spring Boot 3.x |
| 認証 | Spring Security OAuth2 Client（Discord） |
| DB | H2（試作用）→ 後でPostgreSQLに移行可 |
| テスト | JUnit 5 + Mockito + AssertJ |
| ビルドツール | Gradle |

**フロントエンド（SPA）**

| 役割 | 技術 |
|------|------|
| フレームワーク | Vue 3（Composition API） |
| ビルドツール | Vite |
| ルーティング | Vue Router |
| 状態管理 | Pinia |
| HTTP通信 | Axios |
| UIライブラリ | Bootstrap 5 |

**その他**

| 役割 | 技術 |
|------|------|
| タスク管理 | GitHub Projects |
| チャット | なし（DiscordID交換で代替） |

---

## 6. テスト方針

| 優先度 | 対象 |
|--------|------|
| 高 | Service層（ビジネスロジック） |
| 中 | Repository層（DB操作） |
| 低 | Controller層（試作品なら後回しでOK。ただし OAuth2 コールバック `/login/oauth2/code/discord` と `SecurityFilterChain` 設定は例外 ― 認証バグは最重要障害につながるため WebMvcTest で最低限のスモークテストを書く） |

---

## 7. プロジェクト構成（予定）

```
backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com.example.accord/
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── domain/model/
│   │   │       └── config/
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/
│           └── com.example.accord/
│               ├── service/
│               └── repository/
frontend/
├── src/
│   ├── components/
│   ├── views/
│   ├── stores/        # Pinia
│   ├── router/        # Vue Router
│   └── main.js
├── index.html
└── vite.config.js
```

---

## 8. 開発手法

- **カンバン方式**（GitHub Projects）
- `TODO → IN PROGRESS → DONE` でタスク管理
- スプリント単位で機能を区切って開発

---

## 9. ユーザーストーリー

| ID | ストーリー | 受け入れ条件（概要） |
|----|-----------|---------------------|
| US-001 | PCゲーマーとして、Discordアカウントでログインしたい。なぜなら、新しいパスワードを覚えずにアプリを使い始めるため。 | Discordログインボタンから認証できる。DiscordID・アバターが自動取得される。 |
| US-002 | ゲーマーとして、好きなゲーム・プレイ時間帯・スタイルをプロフィールに設定したい。なぜなら、自分に合う仲間に見つけてもらうため。 | プロフィール編集画面で好きなゲーム（ゲームタグ）・時間帯・スタイルを設定できる。 |
| US-003 | ゲーマーとして、ゲームタイトルやプレイ時間帯で絞り込んで他のユーザーを探したい。なぜなら、趣味やスタイルが合う人を効率よく見つけたいため。 | ユーザー一覧で絞り込みができる。最終ログイン日時が表示される。 |
| US-004 | ゲーマーとして、気になるユーザーにいいねを送ったり取り消したりしたい。なぜなら、自分の意思でマッチング相手を選びたいため。 | いいねボタンで送信・取り消しができる。 |
| US-005 | ゲーマーとして、お互いにいいねしたらマッチング成立し、相手の Discord `@username` をマッチング一覧で確認したい。なぜなら、Discordですぐにフレンド申請して一緒に遊びたいため。 | マッチング一覧に相手の `@username`（コピーボタン付き）が表示される。snowflake は表示しない。 |
| US-006 | ゲーマーとして、マッチした相手の一覧を管理したい。なぜなら、誰とマッチしたか把握したいため。 | マッチング一覧ページでマッチ済みユーザーを確認できる。 |
| US-007 | ゲーマーとして、他のユーザーのプロフィール詳細を見たい。なぜなら、いいねを送る前に相手のことを知りたいため。 | プロフィール詳細画面でゲーム・スタイル・時間帯が確認できる。`isLikedByMe`・`isMatched` により自分のアクション状態が一目で分かる。 |
| US-008 | ゲーマーとして、マッチした相手と Discord で繋がった後に「繋がった！」を押して自分の一覧から整理したい。なぜなら、マッチング一覧を常にアクティブな状態に保ちたいため。 | 「Discordで繋がった！」ボタンを押すと **自分側の** 一覧から消える（相手側の表示は維持される）。 |
| US-009 | ゲーマーとして、新しくいいねやマッチングが来たことにすぐ気付きたい。なぜなら、毎回全画面を開いて確認するのは手間だから。 | ナビゲーションに「受信いいね」「マッチング」の未読件数バッジが表示される。画面を開くと既読になる。 |
| US-010 | ゲーマーとして、ログアウトしたら完全にアプリから離脱したい。なぜなら、共用 PC でアカウントを残したくないから。 | ログアウト後にブラウザバックしてもセッションが復活しない（サーバー・フロント両方でクリア）。 |

---

## 10. データモデル

### テーブル一覧

#### users（認証情報）
```sql
users
├── id                     BIGINT PK AUTO_INCREMENT
├── discord_user_id        VARCHAR(32) UNIQUE NOT NULL  -- Discord snowflake（内部一意キー・不変）
├── discord_username       VARCHAR(32) NOT NULL         -- @username（フレンド申請用・ログイン毎に同期）
├── discord_global_name    VARCHAR(64)                  -- 表示名（null 可・ログイン毎に同期）
├── avatar_hash            VARCHAR(64)                  -- アバターのハッシュ。URLは組み立てる（null なら default avatar）
├── last_login_at          TIMESTAMP                    -- 最終ログイン日時（OAuth コールバック時のみ更新）
├── last_checked_likes_at  TIMESTAMP                    -- 受信いいね画面を最後に開いた時刻（未読バッジ用）
├── last_checked_matches_at TIMESTAMP                   -- マッチング一覧を最後に開いた時刻（未読バッジ用）
└── created_at             TIMESTAMP
-- updated_at は last_login_at で代替（users 自体の可変項目が少ないため）
-- アバター URL は `https://cdn.discordapp.com/avatars/{discord_user_id}/{avatar_hash}.png` で都度組み立て
```

#### profiles（プロフィール情報）
```sql
profiles
├── id           BIGINT PK AUTO_INCREMENT
├── user_id      BIGINT FK(users.id) UNIQUE NOT NULL
├── username     VARCHAR(50) NOT NULL         -- 初期値: Discord global_name（無ければ username）
├── bio          VARCHAR(100)                 -- 一言コメント
├── play_style   ENUM('VOICE','TEXT','BOTH')  -- ボイチャ派 / テキスト派 / どちらでも
├── party_style  ENUM('SOLO','FIXED','BOTH')  -- ソロ / 固定パーティ / どちらでも
├── skill_level  ENUM('BEGINNER','INTERMEDIATE','ADVANCED')
├── created_at   TIMESTAMP
└── updated_at   TIMESTAMP
-- avatar_url カラムは廃止（users.avatar_hash から組み立てる）
-- プロフィール未作成状態 = profiles レコードが存在しない（LEFT JOIN で NULL 判定）
```

#### play_times（プレイ時間帯・中間テーブル）
```sql
play_times
├── profile_id  BIGINT FK(profiles.id)
└── time_slot   ENUM('MORNING','AFTERNOON','NIGHT','LATE_NIGHT')
-- PK: (profile_id, time_slot)
-- 複数選択可
-- 値の意味: MORNING=朝 / AFTERNOON=昼 / NIGHT=夜 / LATE_NIGHT=深夜
```

#### games（ゲームマスタ）
```sql
games
├── id              BIGINT PK AUTO_INCREMENT
├── igdb_id         BIGINT UNIQUE NOT NULL       -- IGDB公式ID（唯一のユニークキー）
├── name            VARCHAR(200) NOT NULL        -- IGDB公式ゲーム名（UNIQUE 制約は付けない：同名別作品あり）
├── cover_image_id  VARCHAR(64)                  -- IGDB image_id のみ保存。URL はサイズ指定で組み立てる
└── created_at      TIMESTAMP
-- カバーURL: `https://images.igdb.com/igdb/image/upload/t_{size}/{cover_image_id}.jpg`
-- IGDB APIから取得した公式名で保存 → 表記ゆれ完全防止
-- igdb_id でIGDBと紐付け → 重複防止
-- games テーブル自体を永続キャッシュとして使う（IGDB レート対策）
```

#### profile_games（プロフィールとゲームの中間テーブル）
```sql
profile_games
├── profile_id  BIGINT FK(profiles.id)
└── game_id     BIGINT FK(games.id)
-- PK: (profile_id, game_id)
```

#### likes（いいね）
```sql
likes
├── id           BIGINT PK AUTO_INCREMENT
├── from_user_id BIGINT FK(users.id)
├── to_user_id   BIGINT FK(users.id)
└── created_at   TIMESTAMP
-- UNIQUE(from_user_id, to_user_id) で重複防止
-- CHECK(from_user_id <> to_user_id) 自分自身へのいいね禁止
```

#### matches（マッチング成立）
```sql
matches
├── id                BIGINT PK AUTO_INCREMENT
├── user1_id          BIGINT FK(users.id) NOT NULL
├── user2_id          BIGINT FK(users.id) NOT NULL
├── user1_connected   BOOLEAN DEFAULT FALSE  -- user1 側が「Discordで繋がった！」を押したか
├── user2_connected   BOOLEAN DEFAULT FALSE  -- user2 側が「Discordで繋がった！」を押したか
└── created_at        TIMESTAMP
-- 正規化ルール: 常に user1_id < user2_id で挿入すること
-- CHECK(user1_id < user2_id)
-- UNIQUE(user1_id, user2_id)
-- 「自分側の *_connected フラグが TRUE」なら自分のマッチング一覧から非表示
--  → 片方が押しても相手の一覧には依然として表示される（DiscordID 再確認可能）
```

~~#### messages（チャット）~~
チャット機能は削除。マッチング成立後はDiscordID交換で代替。

### 検索・絞り込みロジック

**試作品：**
```
ゲームタグ（複数選択）AND プレイ時間帯

例：「ApexかつValorantをやってて夜アクティブな人」

-- ゲームタグのAND（両方やってる人）
SELECT p.user_id
FROM profiles p
JOIN profile_games pg ON p.id = pg.profile_id
JOIN games g ON pg.game_id = g.id
WHERE g.name IN ('Apex Legends', 'Valorant')
  AND p.user_id != 自分のID
GROUP BY p.user_id
HAVING COUNT(DISTINCT g.id) = 2  -- 選択タグ数と一致した人のみ

-- プレイ時間帯のOR（どれか1つでも合えばOK）
  AND EXISTS (
    SELECT 1 FROM play_times pt
    WHERE pt.profile_id = p.id
    AND pt.time_slot IN ('NIGHT', 'LATE_NIGHT')
  )
```

**本番：**
```
関連度スコアリング（共通ゲームが多いほど上位）
あいまい検索対応
```

---

### マッチングロジック
```
1. user_a が user_b にいいね → likes(from=a, to=b) を INSERT
2. user_b が user_a にいいね → likes(from=b, to=a) を INSERT
3. likes(from=b, to=a) が存在するか確認
4. 存在する → matches を INSERT（ただし user1_id = LEAST(a,b), user2_id = GREATEST(a,b) で正規化）
   → UNIQUE(user1_id, user2_id) と CHECK(user1_id < user2_id) により二重登録を DB 側で防止
5. いいね返しの競合（ほぼ同時に相互いいね）は INSERT … ON CONFLICT DO NOTHING で片方を無害化
```

### マッチングルール詳細

| ルール | 内容 |
|--------|------|
| いいね取り消し | マッチング前のみ可能（マッチング成立後は不可・ボタン非表示） |
| 再いいね | 取り消し後に同じ相手へ再いいね可能 |
| マッチング解除 | 試作品：なし。マッチ済の相手を「無かったことにする」手段は提供しない（本番で追加予定）。ユーザーには「マッチング後の解除はできません」と UI に明記 |
| 「Discordで繋がった」ボタン | 押した本人の `user*_connected` のみ TRUE に。相手側の表示には影響しない（= 相手はまだ `@username` を確認できる） |
| いいね上限 | なし |
| 自分が受け取ったいいねの表示 | 画面5で一覧表示（`likes.to = 自分`）。自分自身へのいいねは不可（下記バリデーションで弾く） |
| 未読バッジ | 受信いいね件数 = `COUNT(likes WHERE to=自分 AND created_at > users.last_checked_likes_at)`<br>新規マッチ件数 = `COUNT(matches WHERE 自分側 AND created_at > users.last_checked_matches_at)` |

### いいね取り消しロジック
```
マッチング前：
　likes(from=a, to=b) を DELETE → 取り消し完了
　再いいね → likes(from=a, to=b) を INSERT

マッチング成立後：
　取り消し不可（ボタン自体を非表示）
```

### 自分へのいいね表示ロジック
```
画面5（受け取ったいいね一覧）で表示
　likes(to=自分) のユーザーを一覧表示
　→ いいねを返すボタンでマッチング判定
```

### ゲーム登録ロジック（IGDB API連携）
```
1. ユーザーがゲーム名を入力（フロント側 debounce：300ms）
2. Spring Boot 経由で IGDB API に検索リクエスト
   ├─ まずバックエンドの Caffeine キャッシュ（検索クエリ→結果、TTL 15分）をチェック
   ├─ キャッシュ HIT → そのまま返す（IGDB を叩かない）
   └─ キャッシュ MISS → IGDB /v4/games を呼び出し → キャッシュ保存 → 返す
3. IGDB公式名・カバー画像（cover_image_id）でサジェスト表示
4. ユーザーが選択
　　├─ games に存在する（igdb_id で検索）→ 既存の game_id を使用（IGDB は叩かない）
　　└─ 存在しない → games に INSERT（igdb_id, name, cover_image_id）→ その id を使用
```

### IGDB API
```
- エンドポイント：api.igdb.com/v4/games
- 認証：Twitch OAuth2 の access_token（無料）
- レート制限：4 req/sec（超過で即 429）
- トークン管理：有効期限切れを検知して自動リフレッシュ（@Scheduled or lazy refresh）
- 取得データ：ゲーム名・カバー image_id・igdb_id
- Spring Boot の WebClient で呼び出し（RestTemplate は非推奨のため WebClient を推奨）

-- キャッシュ階層（上から順に参照）
-- ① フロント側 debounce（300ms）
-- ② バックエンド Caffeine キャッシュ（検索結果、TTL 15分）
-- ③ games テーブル（永続キャッシュ：一度登録したゲームは IGDB 再照会不要）
```

---

## 11. 画面設計

### 画面一覧・遷移

```
ログイン画面
　　↓ Discordログイン成功
　　├─ 初回 → プロフィール設定画面
　　└─ 登録済み → ユーザー一覧画面
　　　　　　↕
　　　プロフィール詳細画面（DiscordID非表示）

【ナビゲーション共通】
　ユーザー一覧 ／ 受け取ったいいね ／ マッチング一覧 ／ プロフィール編集

　受け取ったいいね一覧画面
　　　　↓ いいねを返す
　　マッチング一覧画面
　　　　↕
　　プロフィール詳細画面（DiscordID表示）
```

---

### 画面1：ログイン画面

| 要素 | 詳細 |
|------|------|
| アプリ名・キャッチコピー | 上部に表示 |
| Discordでログインボタン | クリックでOAuth2認証開始 |

---

### 画面2：プロフィール設定画面（初回ログイン時 ＋ 編集時に使い回す）

| 要素 | 入力形式 |
|------|---------|
| ユーザー名 | テキスト（初期値：Discord表示名） |
| 好きなゲーム | サジェスト付きタグ入力（複数選択） |
| プレイ時間帯 | チェックボックス（昼/夜/深夜/朝） |
| ボイチャ / テキスト派 | ラジオボタン |
| ソロ / 固定パーティ | ラジオボタン |
| スキルレベル | ラジオボタン（初心者/中級者/上級者） |
| 一言コメント | テキストエリア |
| 保存ボタン | クリックでAPI送信 |

---

### 画面3：ユーザー一覧画面

**仕様**
- 自分自身は一覧に表示しない（API側で除外）
- ページング必須（`?limit=20&cursor=<user_id>`、最終ログイン順）
- ヘッダーに受信いいね／マッチングの未読バッジを表示（別エンドポイント `/api/me/unread`）

**絞り込みエリア**
| 要素 | 入力形式 |
|------|---------|
| ゲームタイトル | テキスト入力・IGDBサジェスト付き |
| プレイ時間帯 | チェックボックス（複数選択 OR） |
| プレイスタイル | チェックボックス（VOICE/TEXT/BOTH、複数選択 OR） |
| 絞り込み / リセットボタン | - |

**ユーザーカード**
| 要素 | 備考 |
|------|------|
| アバター画像 | - |
| ユーザー名 | - |
| 好きなゲーム | タグ表示 |
| プレイ時間帯 | - |
| スキルレベル | - |
| 最終ログイン日時 | - |
| いいねボタン | 送信済みは色変更・再クリックで取り消し（マッチング前のみ） |
| 詳細を見るボタン | プロフィール詳細画面へ |

**非同期処理**
```
- 絞り込み → API呼び出し → カード再描画
- いいねボタン → API呼び出し → ボタン状態更新
- ゲームサジェスト → 入力のたびにAPI呼び出し（debounce）
```

---

### 画面4：プロフィール詳細画面

**表示制御：API 側でマッチング状態を判定し、表示項目を出し分ける**
```
APIレスポンスに以下フラグを含める:
  - isLikedByMe: 自分が相手にいいね送信済みか
  - isLikedMe  : 相手から自分にいいねが来ているか
  - isMatched  : マッチ成立済みか
マッチング済み (isMatched=true) のときのみ discordUsername フィールドを返す。
それ以外では discordUsername は null または未含有。
```

| 要素 | 未マッチ | マッチング済み |
|------|---------|--------------|
| アバター画像（大・`avatar_hash` から組み立て） | ◎ | ◎ |
| ユーザー名 | ◎ | ◎ |
| 一言コメント | ◎ | ◎ |
| 好きなゲーム（タグ） | ◎ | ◎ |
| プレイ時間帯 | ◎ | ◎ |
| ボイチャ / テキスト / BOTH | ◎ | ◎ |
| ソロ / 固定パーティ / BOTH | ◎ | ◎ |
| スキルレベル | ◎ | ◎ |
| 最終ログイン日時 | ◎ | ◎ |
| Discord `@username`（フレンド申請用） | 非表示 | ◎（コピーボタン付き） |
| いいねボタン | ◎（`isLikedByMe=true` は色変更＋「取り消し」表示／マッチング前のみ） | 非表示 |

**API レスポンス例**
```json
GET /api/users/{id}
{
  "username": "tanaka",
  "avatarUrl": "https://cdn.discordapp.com/avatars/.../....png",
  "bio": "平日夜からApex",
  "games": [{"id": 1, "name": "Apex Legends", "coverUrl": "..."}],
  "playTimes": ["NIGHT", "LATE_NIGHT"],
  "playStyle": "VOICE",
  "partyStyle": "BOTH",
  "skillLevel": "INTERMEDIATE",
  "lastLoginAt": "2026-04-19T22:10:00+09:00",
  "isLikedByMe": true,
  "isLikedMe": false,
  "isMatched": false,
  "discordUsername": null
}
```

**非同期処理**
```
- 画面表示時 → API呼び出し → プロフィール描画
- いいねボタン → API呼び出し → ボタン状態更新
```

---

### 画面5：受信いいね一覧画面

**目的：** 自分に送られてきたいいね（`likes.to = 自分`）を確認していいねを返す
**仕様：** ページング必須（`?limit=20&cursor=<like_id>`、受信日時降順）。画面を開いた時点で `users.last_checked_likes_at = now()` を更新（未読バッジリセット）。

| 要素 | 備考 |
|------|------|
| アバター画像（`avatar_hash` から組み立て） | - |
| ユーザー名 | - |
| 好きなゲーム（タグ） | - |
| プレイ時間帯 | - |
| スキルレベル | - |
| いいねを返すボタン | クリックでマッチング成立の可能性 |
| プロフィール詳細ボタン | `@username` 非表示で遷移（未マッチのため） |
| 空の場合の表示 | 「まだいいねが届いていません」 |

**非同期処理**
```
- 画面表示時 → API呼び出し → カード描画
- いいねを返すボタン → API呼び出し → マッチング判定 → カードから消える
```

---

### 画面6：マッチング一覧画面

**仕様：** ページング必須（`?limit=20&cursor=<match_id>`、マッチ成立日時降順）。自分側の `user*_connected=TRUE` のレコードは除外して返す（相手側からは引き続き見える）。画面を開いた時点で `users.last_checked_matches_at = now()` を更新。

| 要素 | 備考 |
|------|------|
| アバター画像（`avatar_hash` から組み立て） | - |
| ユーザー名 | - |
| マッチング成立日時 | - |
| 好きなゲーム（タグ） | - |
| Discord `@username` | コピーボタン付き（snowflake ではなくフレンド申請に使える `@username`） |
| 「Discordで繋がった！」ボタン | 押すと **自分側のみ** 一覧から消える（`user*_connected=TRUE`）。相手の一覧には残る |
| プロフィール詳細ボタン | `@username` 表示ありで遷移 |
| 空の場合の表示 | 「まだマッチングしていません」＋ユーザー一覧へのリンク |

**非同期処理**
```
- 画面表示時 → API呼び出し → カード描画
- コピーボタン → クリップボードAPI
```

---

## 12. バリデーション・制約

### プロフィール設定

| 項目 | 制約 | 必須 |
|------|------|------|
| ユーザー名 | 1〜20文字 | 必須 |
| 好きなゲーム | 1〜10タグ・IGDBサジェストからのみ選択可 | 必須（最低1つ） |
| プレイ時間帯 | 複数選択可（MORNING/AFTERNOON/NIGHT/LATE_NIGHT） | 必須（最低1つ） |
| プレイスタイル | VOICE / TEXT / BOTH から1つ | 必須 |
| パーティスタイル | SOLO / FIXED / BOTH から1つ | 必須 |
| スキルレベル | BEGINNER / INTERMEDIATE / ADVANCED から1つ | 必須 |
| 一言コメント | 0〜100文字（本番では長い自己紹介欄を追加予定） | 任意 |

### いいね・マッチング

| 項目 | 制約 |
|------|------|
| 自分自身へのいいね | 不可（API 側で弾く + `CHECK(from_user_id <> to_user_id)`） |
| マッチング済みへのいいね | 不可（ボタン非表示） |
| いいね取り消し | マッチング前のみ |
| 重複いいね | 不可（`UNIQUE(from_user_id, to_user_id)`） |
| マッチングの二重登録 | 不可（`UNIQUE(user1_id, user2_id)` + `CHECK(user1_id < user2_id)`） |
| マッチング解除 | 試作品では提供しない（UI に明記） |

### ゲームタグ

| 項目 | 制約 |
|------|------|
| ゲーム選択 | IGDBサジェストからのみ（自由入力不可） |
| タグ上限 | 10タグまで |

---

## 13. エラーハンドリング

### バックエンド（Spring Boot）

| ケース | ステータス | 例 |
|--------|-----------|-----|
| バリデーションエラー | 400 | ユーザー名が20文字超え |
| 未ログイン | 401 | 認証なしでAPIアクセス |
| 権限なし | 403 | 他人のプロフィールを勝手に編集 |
| リソースなし | 404 | 存在しないユーザーIDにアクセス |
| 重複 | 409 | 同じ相手に重複いいね |
| IGDB APIエラー | 502 | IGDB側の障害（上流サービス障害 = Bad Gateway） |
| サーバーエラー | 500 | 予期しないエラー |

### フロントエンド（Vue 3）

**エラー表示の使い分け：**

| エラー種別 | 表示方法 |
|-----------|---------|
| バリデーションエラー | フォーム内にインライン表示 |
| 通信・サーバーエラー | トースト通知（Bootstrap 5） |
| IGDBエラー | 「ゲーム検索が一時的に使えません」メッセージ表示 |

**ローディング：**
```
APIリクエスト中はスピナー表示
　→ Vue 3のref()でisLoading管理
　→ Axiosのinterceptorで一元管理
```

**トースト管理：**
```
Piniaのグローバルストアで管理
　→ どの画面・コンポーネントからでも呼び出し可能
　→ 成功・警告・エラーで色を使い分け
```

---

## 14. 将来的な拡張（試作品以降）

- プロフィールに長い自己紹介欄（`bio` の文字数上限拡張）を追加
- プロフィール設定の完了率表示
- ユーザー一覧の並び順変更（共通ゲームが多い順など）
- ブロック機能
- マッチング解除機能
- Steam連携（プレイ時間の信頼性担保）
- 通報機能
- 検索の関連度スコアリング
- ゲーム別スキルレベル（`profile_games.skill_level`）― 現在のスキルレベルはプロフィール全体の自己申告だが、ゲームごとに設定できるよう拡張する
- `time_slot` の時刻レンジ定義を明文化・タイムゾーン対応（海外ユーザー対応・Steam 連携時のプレイ時間マッピングに備える）
- DB を PostgreSQL に移行
- デプロイ（Render / Railway など）
