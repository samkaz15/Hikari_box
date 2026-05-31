# キャストデータ設計書

> 女性スタッフ（キャスト）データは本システムの最重要・最大テーブル群  
> 設計の3原則: ① 計算ロジックをDBに持たせない ② 時系列は削除しない ③ 給与データは多重暗号化

---

## 1. テーブル全体像（キャスト関連ERD）

```
[casts] 基本プロフィール（マスター）
  │
  ├─[cast_contracts] 契約条件（日当・ポイント単価）
  │     ※ 変更のたびに新レコード追加（上書き禁止）
  │
  ├─[cast_attendances] 勤怠日次記録
  │     └─[cast_attendance_points] 勤怠起因ポイント明細
  │
  ├─[cast_performances] 営業日次成績
  │     ├─ 指名数 / リクエスト数
  │     ├─ 同伴数（自客/他客）
  │     ├─ ボトル出数 → F-04と連動
  │     └─ 個人純売上
  │
  ├─[cast_point_ledger] ポイント台帳（全加減算の履歴）
  │     ※ イベントソーシング方式：残高は計算で出す・直接書き換えない
  │
  ├─[cast_salary_drafts] 給与ドラフト（計算途中）
  │     └─[cast_salary_confirmed] 給与確定レコード（編集不可・暗号化）
  │
  ├─[cast_assessments] 査定レコード（3ヶ月ごと）
  │
  └─[cast_recruitment] 採用・体入記録
```

---

## 2. 各テーブル定義

### 2-1. casts（基本プロフィール）

```sql
CREATE TABLE casts (
  cast_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  store_id       UUID NOT NULL REFERENCES stores(store_id),
  stage_name     VARCHAR(50) NOT NULL,           -- 源氏名
  real_name      VARCHAR(100),                   -- 本名（暗号化列）
  birth_date     DATE,                           -- 誕生日（暗号化列）
  phone          VARCHAR(20),                    -- 連絡先（暗号化列）
  status         ENUM('active','leave','retired') DEFAULT 'active',
  join_date      DATE NOT NULL,
  retire_date    DATE,
  photo_url      VARCHAR(255),                   -- S3署名付きURL
  created_at     TIMESTAMPTZ DEFAULT now(),
  updated_at     TIMESTAMPTZ DEFAULT now()
);
-- real_name / birth_date / phone は PostgreSQL pgcrypto で列単位暗号化
```

### 2-2. cast_contracts（契約条件マスター）

```sql
CREATE TABLE cast_contracts (
  contract_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cast_id           UUID NOT NULL REFERENCES casts(cast_id),
  effective_from    DATE NOT NULL,               -- この契約の適用開始日
  effective_to      DATE,                        -- NULL = 現在有効
  base_daily_wage   INTEGER NOT NULL,            -- 基本日当（円）
  dohan_quota       INTEGER DEFAULT 4,           -- 同伴ノルマ（回/月）
  shimei_rate       INTEGER DEFAULT 0,           -- 本指名バック率（%）
  sales_threshold   INTEGER DEFAULT 100000,      -- 純売上加算の閾値（円）
  sales_bonus       INTEGER DEFAULT 2000,        -- 閾値超過時の日当加算（円）
  point_unit_yen    INTEGER DEFAULT 1000,        -- 1ポイントあたりの単価（円）
  created_by        UUID NOT NULL,               -- 登録者（権限管理）
  note              TEXT
);
-- 給与改定のたびに INSERT。過去契約は絶対に UPDATE しない。
```

### 2-3. cast_attendances（勤怠記録）

```sql
CREATE TABLE cast_attendances (
  attendance_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cast_id         UUID NOT NULL REFERENCES casts(cast_id),
  store_id        UUID NOT NULL REFERENCES stores(store_id),
  business_date   DATE NOT NULL,
  attend_type     ENUM(
    'normal',        -- 通常出勤
    'late',          -- 遅刻
    'dohan_late',    -- 同伴遅刻（ペナルティなし）
    'early_leave',   -- 早退
    'absent_day',    -- 当日欠勤
    'overtime'       -- 残業
  ) NOT NULL,
  checkin_at      TIMESTAMPTZ,
  checkout_at     TIMESTAMPTZ,
  working_hours   NUMERIC(4,2),                 -- 実稼働時間（自動計算）
  note            TEXT,
  recorded_by     UUID NOT NULL,
  UNIQUE (cast_id, business_date)               -- 1日1レコード制約
);
```

### 2-4. cast_point_ledger（ポイント台帳）★ 設計の核心

```sql
CREATE TABLE cast_point_ledger (
  ledger_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cast_id         UUID NOT NULL REFERENCES casts(cast_id),
  business_date   DATE NOT NULL,
  event_type      ENUM(
    'late',           -- 遅刻     → -2pt
    'dohan_late',     -- 同伴遅刻 →  0pt
    'early_leave',    -- 早退     → -Xpt（契約依存）
    'absent_day',     -- 当日欠勤 → -Xpt（契約依存）
    'overtime',       -- 残業     → +1pt
    'shimei',         -- 場内指名 → +Xpt
    'honshimei',      -- 本指名   → +Xpt
    'dohan',          -- 同伴     → +Xpt
    'bottle',         -- ボトル   → +Xpt
    'manual_adjust',  -- 手動調整 → 権限者のみ
    'salary_settle'   -- 給与精算 → リセット
  ) NOT NULL,
  point_delta     INTEGER NOT NULL,             -- 加算:正 / 減算:負
  reference_id    UUID,                         -- 起因レコードのID（attendance_id等）
  note            TEXT,
  created_by      UUID NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT now()
);

-- ポイント残高はこのビューで計算（直接残高列を持たない）
CREATE VIEW cast_point_balance AS
SELECT
  cast_id,
  SUM(point_delta) AS current_balance,
  MAX(business_date) AS last_event_date
FROM cast_point_ledger
WHERE event_type != 'salary_settle'
  AND business_date > (
    SELECT COALESCE(MAX(business_date), '1900-01-01')
    FROM cast_point_ledger l2
    WHERE l2.cast_id = cast_point_ledger.cast_id
      AND l2.event_type = 'salary_settle'
  )
GROUP BY cast_id;
```

> **なぜ残高列を持たないか**  
> 残高を直接書き換えると、給与計算ミスが起きたとき「いつ・何が原因で狂ったか」が追えなくなる。  
> 全イベントの積み上げで残高を計算することで、任意の時点に巻き戻せる。

### 2-5. cast_performances（日次成績）

```sql
CREATE TABLE cast_performances (
  perf_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cast_id            UUID NOT NULL REFERENCES casts(cast_id),
  store_id           UUID NOT NULL REFERENCES stores(store_id),
  business_date      DATE NOT NULL,
  shimei_count       INTEGER DEFAULT 0,         -- 場内指名数
  honshimei_count    INTEGER DEFAULT 0,         -- 本指名数
  request_count      INTEGER DEFAULT 0,         -- リクエスト数
  dohan_own          INTEGER DEFAULT 0,         -- 同伴（自客）
  dohan_other        INTEGER DEFAULT 0,         -- 同伴（他客）
  bottle_count       INTEGER DEFAULT 0,         -- ボトル出数
  gross_sales        INTEGER DEFAULT 0,         -- 粗売上（担当テーブル合計・税込）
  net_sales          INTEGER DEFAULT 0,         -- 純売上（税抜・GMV）
  back_amount        INTEGER DEFAULT 0,         -- バック金額（自動計算）
  UNIQUE (cast_id, business_date)
);
```

### 2-6. cast_salary_drafts / cast_salary_confirmed（給与計算）

```sql
-- 給与ドラフト（計算途中・編集可能）
CREATE TABLE cast_salary_drafts (
  draft_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cast_id             UUID NOT NULL REFERENCES casts(cast_id),
  period_start        DATE NOT NULL,
  period_end          DATE NOT NULL,
  base_wage_total     INTEGER,                  -- 基本日当合計
  sales_bonus_total   INTEGER,                  -- 純売上加算合計
  point_bonus_total   INTEGER,                  -- ポイント換算額
  shimei_back_total   INTEGER,                  -- 指名バック合計
  deduction_total     INTEGER,                  -- 控除合計（売掛天引き等）
  draft_total         INTEGER,                  -- 仮計算合計
  status              ENUM('calculating','reviewing','approved') DEFAULT 'calculating',
  updated_by          UUID,
  updated_at          TIMESTAMPTZ DEFAULT now()
);

-- 給与確定（承認後・編集不可・暗号化）
CREATE TABLE cast_salary_confirmed (
  salary_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  draft_id            UUID NOT NULL REFERENCES cast_salary_drafts(draft_id),
  cast_id             UUID NOT NULL REFERENCES casts(cast_id),
  period_start        DATE NOT NULL,
  period_end          DATE NOT NULL,
  confirmed_amount    BYTEA NOT NULL,           -- AES-256暗号化
  confirmed_at        TIMESTAMPTZ DEFAULT now(),
  confirmed_by        UUID NOT NULL,
  checksum            VARCHAR(64) NOT NULL      -- SHA-256で改ざん検知
);
-- confirmed後はUPDATE/DELETE権限をDBレベルで剥奪（RLSポリシー）
```

---

## 3. データ量の試算

### 1店舗・キャスト20名・3年運用の場合

| テーブル | レコード数/年 | 3年累計 | 容量目安 |
|----------|-------------|---------|---------|
| cast_attendances | 20名×300日 = 6,000 | 18,000件 | 約5MB |
| cast_performances | 同上 = 6,000 | 18,000件 | 約8MB |
| cast_point_ledger | 6,000×平均5イベント = 30,000 | 90,000件 | 約20MB |
| cast_salary_confirmed | 20名×12ヶ月 = 240 | 720件 | 約1MB（暗号化） |

**→ キャストデータ全体で3年間・1店舗あたり約50〜80MB**  
**→ 10店舗展開でも800MB未満。DBコストへの影響は軽微。**

> 「データが重い」のは容量ではなく **計算ロジックの複雑さ** と **計算誤りが給与ミスに直結するリスク**。

---

## 4. 給与計算フロー（バッチ処理）

```
毎月末 深夜バッチ（自動）
  │
  ├─ 1. 期間内の cast_attendances を集計
  │      → 出勤日数・稼働時間・遅刻回数 etc.
  │
  ├─ 2. cast_point_ledger を集計
  │      → 期間ポイント合計 × point_unit_yen = ポイントボーナス
  │
  ├─ 3. cast_performances を集計
  │      → 純売上合計・sales_threshold超過日数 × sales_bonus
  │      → shimei_back_total（本指名バック）
  │
  ├─ 4. cast_contracts を参照（effective_to IS NULL の最新契約）
  │      → 基本日当 × 出勤日数
  │
  ├─ 5. 控除計算
  │      → 売掛未回収分（accounts_receivable と突合）
  │
  ├─ 6. cast_salary_drafts に書き込み（status: 'calculating'）
  │
  └─ 7. ママ/店長に「給与ドラフト確認依頼」Push通知

承認フロー（手動）
  ├─ ママ/店長がドラフト確認・修正（status: 'reviewing'）
  ├─ オーナー/ママが最終承認（status: 'approved'）
  └─ cast_salary_confirmed に暗号化して書き込み → 編集不可ロック
```

---

## 5. セキュリティ設計（キャスト特有）

```
閲覧権限マトリクス:

                    オーナー  ママ  店長  チーフ  キャスト本人
─────────────────────────────────────────────────────────
源氏名              ○        ○     ○     ○       ○（自分のみ）
本名・連絡先        ○        ○     ○     ✕       ○（自分のみ）
勤怠記録            ○        ○     ○     参照     ○（自分のみ）
ポイント残高        ○        ○     ○     ✕       ○（自分のみ）
給与ドラフト        ○        ○     ○     ✕       ✕
給与確定            ○        ○     ✕     ✕       ○（自分のみ・金額のみ）
他キャストの給与    ○        ○     ✕     ✕       ✕
```

- **Row Level Security (RLS)**: PostgreSQLのRLS機能でアプリ層と独立してDB側でも制御
- **監査ログ**: 給与データへのアクセスは全件 `audit_log` に記録（誰が・いつ・何を見たか）
- **暗号化**: 本名・連絡先・給与確定額は AES-256 列暗号化（鍵はAWS KMS管理）

---

## 6. 30時間未満アラートの実装

```sql
-- 月次稼働時間チェックビュー
CREATE VIEW cast_monthly_hours AS
SELECT
  c.cast_id,
  c.stage_name,
  DATE_TRUNC('month', a.business_date) AS month,
  SUM(a.working_hours) AS total_hours,
  CASE WHEN SUM(a.working_hours) < 30 THEN true ELSE false END AS under_threshold
FROM casts c
JOIN cast_attendances a ON c.cast_id = a.cast_id
WHERE c.status = 'active'
GROUP BY c.cast_id, c.stage_name, DATE_TRUNC('month', a.business_date);

-- 毎月25日に閾値チェックバッチを実行
-- → under_threshold = true のキャストをママ/店長にPush通知
```
