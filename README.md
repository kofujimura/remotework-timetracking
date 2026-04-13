# おひとりオフィス

リモートワーク向けのシンプルな勤怠管理ツールです。Google アカウントでログインし、出勤・退勤・休憩などの打刻と、チームへの報告・メモを記録できます。

## 機能

### 打刻ページ（`index.html`）
- Google アカウントでログイン・ログアウト
- 出勤 / 退勤 / 休憩 / 復帰 の打刻
- 報告・メモの投稿
- 本日のチーム活動フィード（リアルタイム更新）
- 自分の打刻履歴（直近20件）

### レポートページ（`report.html`）
- 月別カレンダーで全員の記録を確認
- 日付クリックで詳細表示（ユーザーごとに色分け）
- 自分の記録を編集・削除

## 技術スタック

| 項目 | 内容 |
|------|------|
| フロントエンド | HTML + Vanilla JS（フレームワークなし） |
| CSS | [Pico.css](https://picocss.com/) |
| バックエンド | [Supabase](https://supabase.com/)（PostgreSQL + Auth + Realtime） |
| 認証 | Google OAuth（implicit flow） |
| デプロイ | 静的ファイルホスティング（任意） |

## セットアップ

### 1. Supabase プロジェクトの準備

#### テーブル作成

```sql
CREATE TABLE time_entries (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID DEFAULT auth.uid() NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
  mode TEXT NOT NULL,
  message TEXT
);

ALTER TABLE time_entries ENABLE ROW LEVEL SECURITY;
```

#### RLS ポリシー

```sql
CREATE POLICY "Allow users to read their own entries" ON time_entries
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Allow users to insert their own entries" ON time_entries
  FOR INSERT WITH CHECK (
    auth.uid() = user_id
    AND (SELECT email FROM auth.users WHERE id = auth.uid()) LIKE '%@yourdomain.ac.jp'
  );

CREATE POLICY "Allow users to update their own entries" ON time_entries
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Allow users to delete their own entries" ON time_entries
  FOR DELETE USING (auth.uid() = user_id);
```

#### RPC 関数

```sql
GRANT USAGE ON SCHEMA auth TO authenticated;
GRANT SELECT ON TABLE auth.users TO authenticated;

-- 本日の活動フィード
CREATE OR REPLACE FUNCTION get_todays_activities()
RETURNS TABLE (created_at TIMESTAMPTZ, email TEXT, mode TEXT)
AS $$
BEGIN
  RETURN QUERY
  SELECT t.created_at, u.email::text, t.mode
  FROM time_entries t JOIN auth.users u ON t.user_id = u.id
  WHERE t.created_at AT TIME ZONE 'Asia/Tokyo' >= date_trunc('day', now() AT TIME ZONE 'Asia/Tokyo')
    AND t.mode != '報告'
  ORDER BY t.created_at ASC;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 月別データ取得
CREATE OR REPLACE FUNCTION get_activities_by_date(start_date DATE, end_date DATE)
RETURNS TABLE (id UUID, created_at TIMESTAMPTZ, email TEXT, mode TEXT, message TEXT, user_id UUID)
AS $$
BEGIN
  RETURN QUERY
  SELECT t.id, t.created_at, u.email::text, t.mode, t.message, t.user_id
  FROM time_entries t JOIN auth.users u ON t.user_id = u.id
  WHERE (t.created_at AT TIME ZONE 'Asia/Tokyo')::date BETWEEN start_date AND end_date
  ORDER BY t.created_at ASC;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Realtime 有効化
ALTER TABLE time_entries REPLICA IDENTITY FULL;
ALTER PUBLICATION supabase_realtime ADD TABLE time_entries;
```

### 2. Google OAuth の設定

1. Supabase Dashboard → **Authentication → Providers → Google** を有効化
2. Google Cloud Console で OAuth クライアントを作成し、Client ID / Secret を設定
3. 承認済みリダイレクト URI に `https://<your-project>.supabase.co/auth/v1/callback` を追加

### 3. Redirect URLs の設定

Supabase Dashboard → **Authentication → URL Configuration → Redirect URLs** にアプリのURLを追加

```
https://your-app-url.example.com
https://your-app-url.example.com/report.html
```

### 4. コードの設定

`index.html` と `report.html` の以下の値を書き換えます：

```javascript
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-key';
```

また、ドメイン制限の `otsuma.ac.jp` を自組織のドメインに変更します：

- `index.html` および `report.html` の `updateUI` 関数内
- Supabase の INSERT ポリシーの `LIKE '%@yourdomain.ac.jp'`

### 5. デプロイ

`index.html` と `report.html` を静的ファイルホスティングに配置するだけで動作します。

- Cloudflare Pages
- Netlify
- Vercel
- Firebase Hosting など

## ライセンス

MIT
