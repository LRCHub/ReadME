# 概要

* 歌詞検索サイト / 手動登録ツール / DynamicLyrics 用のバックエンド API 群です。
* 主な役割

  * 曲名から歌詞を検索し、GitHub リポジトリとして保存
  * 既存歌詞の取得・手動登録
  * DynamicLyrics（1文字ごとのタイムスタンプ）の保存
  * 翻訳歌詞の登録・取得
  * 歌詞候補の選択・確定
  * 歌詞フレーズ共有（Shere）

※VideoIDではなくURLを渡してください
Youtubeの仕様変更やBot判定によりエラーを返すことがあります

## ベース URL

* 例: `https://lrchub.coreone.work`
* 以下、`<BASE_URL>` と表記します。

## 共通仕様

* JSON API はすべて UTF-8 / `application/json`。
* 基本レスポンスは以下のような形です。

```json
{
  "ok": true,
  "message": "任意のメッセージ",
  "code": "OPTIONAL_ERROR_CODE"
}
```

* エラー時：

  * `ok: false`
  * HTTP ステータス 4xx / 5xx
  * `code` フィールドに機械可読なエラーコードが入ることがあります。

---

# 1. 歌詞自動検索 API

## 1-1. 歌詞検索

**POST** `/api/lyrics`

### リクエストボディ

```json
{
  "track": "曲名（必須）",
  "artist": "アーティスト名（任意）",
  "youtube_url": "YouTube URL または VideoID（任意）"
}
```

* `track`
  曲名。必須。空だと `"曲名を入力してください。"` エラーになります。
* `artist`
  アーティスト名。省略可。
* `youtube_url`

  * YouTube の URL もしくは 11 文字の VideoID。
  * 渡した場合、その動画リポジトリに直接保存されます。
  * 渡さない場合は、内部で YouTube 検索して video_id を推定します（推定失敗時は GitHub に保存されないこともあります）。

### レスポンス（成功時）

```json
{
  "ok": true,
  "message": "",
  "track": "正規化された曲名",
  "artist": "正規化されたアーティスト名",
  "video_id": "YouTube VideoID もしくは GitHub リポジトリ名",
  "plain_lyrics": "プレーン歌詞（テキスト）",
  "synced_lyrics": "LRC 形式の同期歌詞",
  "dynamic_lyrics": { /* DynamicLyrics.json の内容（GitHub ヒット時のみ） */ },

  "from_github": true,
  "registered": false,
  "queued": false,
  "github_url": "https://github.com/.../VIDEO_ID",

  "has_select_candidates": true,
  "select_candidates_saved": true,

  "candidates_api_url": "https://.../api/lyrics_candidates?video_id=...&include_lyrics=1"
}
```

主要フィールド:

* `plain_lyrics`
  プレーンな歌詞。LRC がある場合も、プレーン歌詞が分かれば入ります。
* `synced_lyrics`
  LRC 形式の同期歌詞。存在しない場合は空文字。
* `dynamic_lyrics`
  リポジトリに `DynamicLyrics.json` がある時だけ返されるオブジェクト。
* `from_github`

  * `true`: 既に GitHub に登録済みの歌詞を再利用
  * `false`: 外部 API から新規取得した歌詞
* `registered`

  * `true`: 今回のリクエストで新たに GitHub リポジトリを作成/更新して登録した
  * `false`: 既存流用、または GitHub に保存できなかった（VideoID 不明など）
* `queued`

  * `true`: GitHub Secondary Rate Limit により、保存処理を内部キューに積んだ状態
  * キューに載っていても `plain_lyrics` / `synced_lyrics` 自体はレスポンスに含まれます。
* `github_url`
  該当曲用の GitHub リポジトリ URL（LRCHub ページ）。
* `has_select_candidates`
  歌詞候補（`select/index.json` / PetitLyrics 候補など）が存在する場合 `true`。
* `select_candidates_saved`
  候補が GitHub の `select/` 配下に保存された場合 `true`。
* `candidates_api_url`
  歌詞候補一覧（`/api/lyrics_candidates`）を取得するための URL。
  ※ `lyrics_candidates` エンドポイントの実装は別途。

### レスポンス（歌詞が見つからない場合）

```json
{
  "ok": true,
  "message": "該当する歌詞は見つかりませんでした。",
  "track": "...",
  "artist": "...",
  "video_id": "videoid もしくは null",
  "plain_lyrics": "",
  "synced_lyrics": "",
  "from_github": false,
  "registered": false,
  "queued": false,
  "github_url": null,
  "has_select_candidates": true,
  "select_candidates_saved": false
}
```

### エラー例

* `400 Bad Request`

  * `"message": "曲名を入力してください。"`
  * `"message": "有効な YouTube の URL / VideoID ではありません。"` など
* `500 Internal Server Error`

  * `"message": "検索に失敗しました。"`

---

## 翻訳歌詞の取得（GET）

**GET** `/api/translation`

`translation/<lang>.txt` の中身を取得します。

### クエリパラメータ

* `youtube_url` または `video_id`（必須）
* `lang`（任意・複数指定可）
  例: `?video_id=VIDEO_ID&lang=en&lang=ja`
  指定しない場合は存在するすべての翻訳を返す。

### レスポンス（成功時）

```json
{
  "ok": true,
  "video_id": "VIDEO_ID",
  "translations": {
    "en": "英訳歌詞のテキスト...",
    "ja": "日本語訳のテキスト..."
  },
  "missing_langs": ["ko"]
}
```

* `missing_langs`
  クエリで `lang` を指定した場合のみ意味があり、
  リクエストされたが存在しなかった言語コードが入ります。

### エラー

* `400`

  * `code: "REQUIRED_VIDEO"`
  * `code: "INVALID_URL"`
* `500`

  * `code: "GITHUB_ERROR"`

---

# 4. 歌詞候補選択・ロック API

## 4-1. 歌詞候補から選択して反映

**POST** `/api/lyrics_select`

`select/index.json` に保存された複数候補の中から 1 件を選び、`README.md` に適用します。
（必要に応じてロックも可能）

### リクエストボディ（通常の候補選択）

```json
{
  "youtube_url": "YouTube URL または VideoID",
  "video_id": "VIDEO_ID（youtube_url の代わりに指定可）",

  "candidate_id": "select/index.json 内の id 文字列",
  "lock": true
}
```

* `candidate_id`
  `select/index.json` の各 candidate の `id` フィールド（例: `"lrclib_1"`, `"petitlyrics_2"`）。
* `lock`

  * `true` の場合、適用後すぐに Sync 歌詞をロック（`SyncLocked=true`）します。
  * 省略可。

### レスポンス（成功時）

```json
{
  "ok": true,
  "message": "選択した歌詞を README に適用しました。",
  "video_id": "VIDEO_ID",
  "candidate_id": "lrclib_1",
  "github_url": "https://github.com/.../VIDEO_ID",
  "locked": true
}
```

### 特殊モード：現在の歌詞をロックする

`candidate_id` の代わりに `request` / `action` に特定文字列を入れると、
候補ではなく「既存歌詞をロック」するモードになります（Sync or Dynamic）。

```json
{
  "video_id": "VIDEO_ID",
  "request": "lock_current_sync"
}
```

対応しているキー（大文字小文字無視）:

* Sync ロック系

  * `"lock_current_sync"`, `"confirm_current_sync"`, `"lock_sync"`
  * `"現在の1行ごとの歌詞を確定"`
* Dynamic ロック系

  * `"lock_current_dynamic"`, `"confirm_current_dynamic"`, `"lock_dynamic"`
  * `"現在の１文字ごとの歌詞を確定"`

#### レスポンス例

```json
{
  "ok": true,
  "code": "OK",
  "message": "現在の1行ごとの歌詞を確定しました。",
  "video_id": "VIDEO_ID",
  "target": "sync",
  "config": {
    "syncVlyrics": true,
    "dynmiclyrics": false,
    "SyncLocked": true,
    "dynmicLock": false
  }
}
```

### 主なエラー

* `400`

  * `code: "REQUIRED_VIDEO"`
  * `code: "REQUIRED_CANDIDATE_ID"`
  * `code: "INVALID_URL"`
  * `code: "LYRICS_LOCKED"`（候補反映前のロックチェック）
  * `code: "CANDIDATE_NOT_FOUND"`
  * `code: "CANDIDATE_MISSING_PATH"`
  * `code: "LYRICS_NOT_FOUND"`（候補ファイルが存在しない / 空）
* `404`

  * `code: "SELECT_NOT_FOUND"`（select/index.json が存在しない）
  * `code: "GITHUB_REPO_NOT_FOUND"`
* `500`

  * `code: "GITHUB_ERROR"`（index 取得失敗）
  * `code: "APPLY_FAILED"`（適用失敗）

---

## 4-2. 歌詞ロック専用 API

**POST** `/api/lyrics_lock`

Sync 歌詞(LRC)または DynamicLyrics をロックするためのシンプルな API。
（`/api/lyrics_select` の特殊モードと似ていますが、こちらは target 指定。）

### リクエストボディ

```json
{
  "youtube_url": "YouTube URL または VideoID",
  "video_id": "VIDEO_ID（youtube_url の代わりに指定可）",
  "target": "sync"  // または "dynamic"
}
```

※ `type` というパラメータ名でも受け付けます（内部で `target` にマージ）。

### レスポンス（成功時）

```json
{
  "ok": true,
  "code": "OK",
  "message": "ロックしました。",
  "video_id": "VIDEO_ID",
  "config": {
    "syncVlyrics": true,
    "dynmiclyrics": true,
    "SyncLocked": true,
    "dynmicLock": false
  }
}
```

* 既にロック済みの場合:

  * `code: "ALREADY_LOCKED"`
  * `message: "既にロック済みです。"`

### 主なエラー

* `400`

  * `code: "REQUIRED_VIDEO"`
  * `code: "INVALID_TARGET"`（`sync`/`dynamic` 以外）
  * `code: "INVALID_URL"`
  * `code: "LYRICS_NOT_FOUND"`（対象の歌詞がまだ登録されていない）
  * `code: "SAVE_FAILED"`（設定保存失敗）
* `404`

  * `code: "GITHUB_REPO_NOT_FOUND"`

---

# 5. フレーズ共有 API (Shere)

## 5-1. 歌詞フレーズ共有 URL 登録

**POST** `/api/share/register`

特定動画の特定タイミングのフレーズを GitHub(`Shere` リポジトリ) に保存し、
共有 URL（`/s/<video_id>/<time_sec>?...`）を返します。

### リクエストボディ

```json
{
  "youtube_url": "YouTube URL または VideoID",
  "video_id": "VIDEO_ID（youtube_url の代わりに指定可）",

  "phrase": "共有したいフレーズ",
  "text": "phrase の別名（phrase が無ければこちら）",

  "lang": "ja",          // 任意。デフォルト ja
  "time_ms": 12345,      // ms または
  "time_sec": 12         // 秒を指定（どちらかは必須）
}
```

* `phrase` / `text`
  どちらかにフレーズが入っていればOK。両方空でも登録自体はされます。
* `time_ms` / `time_sec`
  どちらか必須。片方からもう片方を補完します。

### レスポンス（成功時）

```json
{
  "ok": true,
  "video_id": "VIDEO_ID",
  "time_sec": 12,
  "time_ms": 12000,
  "lang": "ja",
  "share_url": "https://your-domain/s/VIDEO_ID/12?phrase=...&lang=ja",
  "github_saved": true
}
```

* `share_url`

  * `_share_base_url()`（実際のホスト or `SHARE_BASE_URL`）を使った共有用 URL。
  * `phrase` / `lang` はクエリに含まれます（空なら付かない）。

### エラー

* `400`

  * `"youtube_url または video_id を指定してください。"`
  * `"有効な YouTube の URL / VideoID ではありません。"`
  * `"time_sec または time_ms を指定してください。"`
* `500`

  * `"GitHub への保存に失敗しました。"`

---

# 6. HTML ページ（人間用 UI）

API ではありませんが、フロントエンドとして用意されているページです。

## 6-1. 自動検索ページ

**GET** `/`

* 歌詞検索フォーム & 結果表示。
* 内部で `POST /api/lyrics` を呼び出します。
* GitHub への登録状況や LRCHub ページへのリンクなども表示。

## 6-2. 手動登録ページ

**GET** `/manual`

* YouTube プレイヤーと歌詞テキストエリア、タイミング記録 UI。
* `POST /api/manual_init` / `POST /api/manual_register` を利用。
* 1 行ずつ、さらに 1 文字ずつの DynamicLyrics 記録モードをサポート。

## 6-3. タスク一覧ページ

**GET** `/tasks`（実装側でこのパスに HTML がある想定）

* `GET /api/tasks_missing` を利用して、

  * タイムスタンプ無し
  * DynamicLyrics 無し
    の動画リストを表示する管理用 UI。

---

以上が、`web.py` に含まれている主な API の仕様です。
「このエンドポイントのリクエスト例をもう少し詳しく」「OpenAPI（YAML）形式にしてほしい」などあれば、その形式でも書き直します。
