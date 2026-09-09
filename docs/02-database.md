# תכנון בסיס נתונים

Postgres 17 (Supabase). הכללים כאן נגזרים מ-`supabase-postgres-best-practices`.

## 0. מוסכמות רוחביות

| כלל | נימוק |
|---|---|
| מזהים: `bigint generated always as identity` פנימי + `public_id uuid` לחשיפה | bigint = לוקליות אינדקס טובה; UUID חיצוני כדי לא לדלוף נפחי עסק |
| `text` ולא `varchar(n)` | אין הבדל ביצועים, אין מגבלה שרירותית |
| `timestamptz` ולא `timestamp` | CRM עם משתמשים ואינטגרציות — timezone הוא חובה |
| כסף: `numeric(12,2)` + `currency text` | `float` לכסף הוא באג ממתין |
| מחיקה: `deleted_at timestamptz` (soft delete) | CRM לא מוחק לידים — audit ו-compliance |
| שמות: `snake_case`, טבלאות ברבים | ללא מזהים תלויי-רישיות |
| enums: `text` + `check` ולא `create type ... as enum` | הוספת ערך ל-enum טיפוס דורשת נעילה; check constraint מתעדכן בקלות |

## 1. תרשים ישויות (ליבה)

```
organizations
     │
     ├── users ──┬── user_roles ── roles ── role_permissions ── permissions
     │           │
     ├── courses ── cohorts ─────────────┐
     │                                    │
     ├── leads ──────────► enrollments ◄──┤
     │     │                    │         │
     │     │                    └── students
     │     │                          │
     │     │                    payment_plans ── payments
     │     │
     │     └── lead_scores (היסטוריה)
     │
     ├── tasks
     ├── activities   (טיימליין פולימורפי)
     ├── notes
     ├── documents
     ├── notifications
     ├── ai_interactions
     └── audit_logs
```

## 2. טבלאות ליבה

### 2.1 organizations

```sql
create table organizations (
  id          bigint generated always as identity primary key,
  public_id   uuid not null default gen_random_uuid() unique,
  name        text not null,
  slug        text not null unique,
  timezone    text not null default 'Asia/Jerusalem',
  currency    text not null default 'ILS',
  settings    jsonb not null default '{}'::jsonb,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);
```

### 2.2 users — מראה ל-auth.users

Supabase Auth מחזיק את הסודות. הטבלה הזו מחזיקה את הפרופיל העסקי בלבד.

```sql
create table users (
  id            bigint generated always as identity primary key,
  auth_user_id  uuid not null unique references auth.users(id) on delete cascade,
  org_id        bigint not null references organizations(id),
  email         text not null,
  full_name     text not null,
  phone         text,
  avatar_url    text,
  is_active     boolean not null default true,
  last_seen_at  timestamptz,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now(),
  constraint users_org_email_uniq unique (org_id, email)
);
create index users_org_active_idx on users (org_id) where is_active;
```

### 2.3 הרשאות — RBAC טבלאי, לא עמודת `role text`

```sql
create table permissions (
  id     bigint generated always as identity primary key,
  key    text not null unique,          -- 'lead.read.all', 'payment.write'
  domain text not null,                 -- 'lead'
  action text not null                  -- 'read'
);

create table roles (
  id         bigint generated always as identity primary key,
  org_id     bigint references organizations(id),   -- null = תפקיד מערכת
  key        text not null,
  name       text not null,
  is_system  boolean not null default false,
  constraint roles_org_key_uniq unique (org_id, key)
);

create table role_permissions (
  role_id       bigint not null references roles(id) on delete cascade,
  permission_id bigint not null references permissions(id) on delete cascade,
  primary key (role_id, permission_id)
);

create table user_roles (
  user_id     bigint not null references users(id) on delete cascade,
  role_id     bigint not null references roles(id) on delete cascade,
  granted_by  bigint references users(id),
  granted_at  timestamptz not null default now(),
  primary key (user_id, role_id)
);
create index role_permissions_permission_idx on role_permissions (permission_id);
create index user_roles_role_idx on user_roles (role_id);
```

תפקידי מערכת: `owner`, `admin`, `sales_manager`, `sales_rep`, `academic_coordinator`,
`finance`, `viewer`.

**למה טבלאות ולא `users.role text`:** ברגע שמבקשים "לנציג X מותר גם לראות תשלומים",
מודל של עמודה יחידה נשבר ודורש שכתוב. RBAC טבלאי עונה על זה בשורה אחת.

### 2.4 courses + cohorts

```sql
create table courses (
  id             bigint generated always as identity primary key,
  public_id      uuid not null default gen_random_uuid() unique,
  org_id         bigint not null references organizations(id),
  name           text not null,
  slug           text not null,
  description    text,
  level          text check (level in ('beginner','intermediate','advanced')),
  duration_hours int check (duration_hours > 0),
  list_price     numeric(12,2) not null check (list_price >= 0),
  currency       text not null default 'ILS',
  is_active      boolean not null default true,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),
  deleted_at     timestamptz,
  constraint courses_org_slug_uniq unique (org_id, slug)
);

create table cohorts (
  id             bigint generated always as identity primary key,
  public_id      uuid not null default gen_random_uuid() unique,
  org_id         bigint not null references organizations(id),
  course_id      bigint not null references courses(id),
  name           text not null,
  starts_on      date not null,
  ends_on        date,
  capacity       int not null check (capacity > 0),
  status         text not null default 'planned'
                 check (status in ('planned','open','full','running','completed','cancelled')),
  delivery       text not null default 'online'
                 check (delivery in ('online','onsite','hybrid')),
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),
  constraint cohorts_dates_ck check (ends_on is null or ends_on >= starts_on)
);
create index cohorts_course_idx   on cohorts (course_id);
create index cohorts_org_open_idx on cohorts (org_id, starts_on)
  where status in ('planned','open');
```

`capacity` נאכף בשכבת ה-service עם `select ... for update` על שורת ה-cohort — לא בטריגר.
מרוץ תנאים על המקום האחרון במחזור הוא תרחיש אמיתי ביום פתיחת ההרשמה.

### 2.5 leads — לב המערכת

```sql
create table leads (
  id                 bigint generated always as identity primary key,
  public_id          uuid not null default gen_random_uuid() unique,
  org_id             bigint not null references organizations(id),
  full_name          text not null,
  email              text,
  phone              text,
  phone_e164         text,                      -- מנורמל, בסיס לזיהוי כפילויות
  source             text not null default 'manual',
  source_detail      jsonb not null default '{}'::jsonb,
  status             text not null default 'new'
                     check (status in ('new','contacted','qualified','proposal',
                                       'negotiation','won','lost','nurture')),
  lost_reason        text,
  interest_course_id bigint references courses(id),
  interest_cohort_id bigint references cohorts(id),
  owner_id           bigint references users(id),
  budget             numeric(12,2),
  first_contacted_at timestamptz,
  last_activity_at   timestamptz,
  next_follow_up_at  timestamptz,
  converted_at       timestamptz,
  search_vector      tsvector generated always as (
                       to_tsvector('simple',
                         coalesce(full_name,'') || ' ' ||
                         coalesce(email,'')     || ' ' ||
                         coalesce(phone,''))
                     ) stored,
  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now(),
  deleted_at         timestamptz,
  constraint leads_contact_ck     check (email is not null or phone is not null),
  constraint leads_lost_reason_ck check (status <> 'lost' or lost_reason is not null)
);
```

אינדקסים — נגזרים מהמסכים שהוגדרו, לא "ליתר ביטחון":

```sql
-- מסך "הלידים שלי", ממוין לפי פעילות אחרונה
create index leads_owner_status_idx
  on leads (org_id, owner_id, status, last_activity_at desc)
  where deleted_at is null;

-- מסך follow-ups שפג מועדם
create index leads_followup_idx on leads (org_id, next_follow_up_at)
  where deleted_at is null and status not in ('won','lost');

-- זיהוי כפילויות בקליטת ליד
create unique index leads_org_phone_uniq on leads (org_id, phone_e164)
  where phone_e164 is not null and deleted_at is null;
create unique index leads_org_email_uniq on leads (org_id, lower(email))
  where email is not null and deleted_at is null;

-- חיפוש חופשי
create index leads_search_idx on leads using gin (search_vector);
create index leads_trgm_idx   on leads using gin (full_name gin_trgm_ops);

-- FK indexes (Postgres אינו יוצר אותם אוטומטית)
create index leads_course_idx on leads (interest_course_id);
create index leads_cohort_idx on leads (interest_cohort_id);
create index leads_owner_idx  on leads (owner_id);
```

**החלטה:** `search_vector` הוא עמודה מחושבת (`generated ... stored`) ולא טריגר —
פחות קוד, ואי אפשר שייצא מסנכרון. `pg_trgm` נוסף בשביל חיפוש עמיד לשגיאות כתיב
בעברית, שאותו `tsvector` לא נותן.

### 2.6 students, enrollments, payments

```sql
create table students (
  id              bigint generated always as identity primary key,
  public_id       uuid not null default gen_random_uuid() unique,
  org_id          bigint not null references organizations(id),
  lead_id         bigint unique references leads(id),   -- מקור ההמרה
  full_name       text not null,
  email           text not null,
  phone_e164      text,
  national_id_enc bytea,                                -- מוצפן בשכבת היישום
  status          text not null default 'active'
                  check (status in ('active','graduated','dropped','on_hold')),
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now(),
  deleted_at      timestamptz
);

create table enrollments (
  id            bigint generated always as identity primary key,
  public_id     uuid not null default gen_random_uuid() unique,
  org_id        bigint not null references organizations(id),
  student_id    bigint not null references students(id),
  cohort_id     bigint not null references cohorts(id),
  status        text not null default 'reserved'
                check (status in ('reserved','confirmed','active','completed','cancelled','refunded')),
  price         numeric(12,2) not null check (price >= 0),
  discount      numeric(12,2) not null default 0 check (discount >= 0),
  currency      text not null default 'ILS',
  enrolled_at   timestamptz not null default now(),
  cancelled_at  timestamptz,
  cancel_reason text,
  constraint enrollments_student_cohort_uniq unique (student_id, cohort_id),
  constraint enrollments_discount_ck check (discount <= price)
);
create index enrollments_cohort_idx  on enrollments (cohort_id, status);
create index enrollments_student_idx on enrollments (student_id);

create table payment_plans (
  id            bigint generated always as identity primary key,
  org_id        bigint not null references organizations(id),
  enrollment_id bigint not null references enrollments(id) on delete cascade,
  total_amount  numeric(12,2) not null check (total_amount >= 0),
  currency      text not null default 'ILS',
  installments  int not null default 1 check (installments > 0),
  created_at    timestamptz not null default now()
);
create index payment_plans_enrollment_idx on payment_plans (enrollment_id);

create table payments (
  id              bigint generated always as identity primary key,
  public_id       uuid not null default gen_random_uuid() unique,
  org_id          bigint not null references organizations(id),
  payment_plan_id bigint not null references payment_plans(id) on delete cascade,
  seq             int not null,
  amount          numeric(12,2) not null check (amount > 0),
  currency        text not null default 'ILS',
  due_date        date not null,
  paid_at         timestamptz,
  method          text check (method in ('card','transfer','cash','bit','other')),
  status          text not null default 'pending'
                  check (status in ('pending','paid','failed','refunded','cancelled')),
  external_ref    text,
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now(),
  constraint payments_plan_seq_uniq unique (payment_plan_id, seq),
  constraint payments_paid_ck check (status <> 'paid' or paid_at is not null)
);
-- מסך גבייה: תשלומים שפג מועדם
create index payments_overdue_idx on payments (org_id, due_date)
  where status = 'pending';
create index payments_plan_idx on payments (payment_plan_id);
```

### 2.7 activities — טיימליין פולימורפי

```sql
create table activities (
  id          bigint generated always as identity primary key,
  org_id      bigint not null references organizations(id),
  entity_type text not null check (entity_type in ('lead','student','enrollment','payment','task')),
  entity_id   bigint not null,
  type        text not null check (type in ('call','email','whatsapp','meeting','note',
                                            'status_change','system','ai')),
  title       text not null,
  body        text,
  metadata    jsonb not null default '{}'::jsonb,
  actor_id    bigint references users(id),
  occurred_at timestamptz not null default now(),
  created_at  timestamptz not null default now()
);
create index activities_entity_idx on activities (org_id, entity_type, entity_id, occurred_at desc);
create index activities_actor_idx  on activities (actor_id, occurred_at desc);
```

**החלטה מודעת:** אין FK על `entity_id` (קשר פולימורפי). המחיר: אין אכיפת שלמות ב-DB.
התמורה: טיימליין אחיד לכל ישות במקום שש טבלאות מקבילות. התקינות נאכפת בשכבת ה-service,
וניקוי יתומים ב-job לילי. החלופה — `activity_lead`, `activity_student`, ... — נבחנה ונדחתה:
כל ישות חדשה הייתה דורשת טבלה, אינדקסים, ו-`UNION` בכל שאילתת טיימליין.

### 2.8 audit_logs — מחולק לפי זמן

```sql
create table audit_logs (
  id          bigint generated always as identity,
  org_id      bigint not null,
  actor_id    bigint,
  action      text not null,              -- 'lead.update'
  entity_type text not null,
  entity_id   bigint,
  before      jsonb,
  after       jsonb,
  ip          inet,
  user_agent  text,
  request_id  text,
  created_at  timestamptz not null default now(),
  primary key (id, created_at)
) partition by range (created_at);
```

חלוקה חודשית. ה-audit log של CRM גדל מהר וכמעט אף פעם אינו נקרא מעבר ל-90 יום —
partitioning מאפשר `drop partition` במקום `delete` יקר. יצירת partitions עתידיים ב-`pg_cron`.

### 2.9 טבלאות AI

```sql
create table lead_scores (
  id            bigint generated always as identity primary key,
  org_id        bigint not null references organizations(id),
  lead_id       bigint not null references leads(id) on delete cascade,
  score         numeric(5,4) not null check (score between 0 and 1),
  band          text not null check (band in ('cold','warm','hot')),
  model_version text not null,
  features      jsonb not null,           -- snapshot לצורך הסבר ו-debug
  computed_at   timestamptz not null default now()
);
create index lead_scores_lead_idx on lead_scores (lead_id, computed_at desc);

create table ai_interactions (
  id                bigint generated always as identity primary key,
  org_id            bigint not null references organizations(id),
  user_id           bigint references users(id),
  feature           text not null,        -- 'summarize_call' | 'draft_email' | ...
  entity_type       text,
  entity_id         bigint,
  model             text not null,
  prompt_tokens     int,
  completion_tokens int,
  cost_usd          numeric(10,6),
  latency_ms        int,
  status            text not null check (status in ('ok','error','filtered')),
  error             text,
  created_at        timestamptz not null default now()
);
create index ai_interactions_org_created_idx on ai_interactions (org_id, created_at desc);
```

`ai_interactions` אינו "nice to have": בלעדיו אין דרך לענות על "כמה ה-AI עולה לנו החודש"
ועל "למה הנציג קיבל המלצה שגויה".

## 3. Row Level Security

RLS מופעל על **כל** טבלה עם `org_id`, כולל `force row level security` כדי שגם בעל הטבלה
יהיה כפוף לה.

```sql
-- פונקציות עזר בסכימה פרטית שאינה חשופה דרך ה-API
create schema if not exists private;

create or replace function private.current_org_id()
returns bigint language sql stable security definer set search_path = ''
as $$
  select nullif(current_setting('request.jwt.claims', true)::jsonb ->> 'org_id','')::bigint;
$$;

create or replace function private.has_permission(perm text)
returns boolean language sql stable security definer set search_path = ''
as $$
  select exists (
    select 1
    from public.user_roles ur
    join public.role_permissions rp on rp.role_id = ur.role_id
    join public.permissions p       on p.id = rp.permission_id
    join public.users u             on u.id = ur.user_id
    where u.auth_user_id = (select auth.uid())
      and p.key = perm
  );
$$;

revoke execute on function private.current_org_id(), private.has_permission(text)
  from public, anon, authenticated;
```

מדיניות לדוגמה על `leads`:

```sql
alter table leads enable row level security;
alter table leads force  row level security;

create policy leads_select on leads for select to authenticated
using (
  org_id = (select private.current_org_id())
  and deleted_at is null
  and (
    (select private.has_permission('lead.read.all'))
    or owner_id = (select private.current_user_id())
  )
);

create policy leads_insert on leads for insert to authenticated
with check (
  org_id = (select private.current_org_id())
  and (select private.has_permission('lead.create'))
);
```

שתי נקודות ביצועים קריטיות:

1. כל קריאה לפונקציה עטופה ב-`(select ...)` — Postgres מעריך אותה פעם אחת במקום פעם לכל
   שורה. על 100K לידים ההבדל הוא סדר גודל.
2. `org_id` ו-`owner_id` הן העמודות המובילות באינדקסים. מדיניות RLS ללא אינדקס תואם הופכת
   כל שאילתה ל-sequential scan.

## 4. Connection pooling

Flask מתחבר דרך **Supavisor במצב transaction**. `SET LOCAL` בתוך טרנזקציה מפורשת בטוח
במצב זה — הערך מתאפס ב-`COMMIT` ואינו דולף לבקשה הבאה שתקבל את אותו חיבור. זו בדיוק
הסיבה שהזרקת הזהות היא `SET LOCAL` ולא `SET`.

## 5. מיגרציות

- SQL גולמי, ממוספר: `migrations/0001_init.sql`, `0002_rbac.sql`, ...
- Forward-only. תיקון = מיגרציה חדשה, לא עריכה של קיימת.
- כל DDL על טבלה חיה עם `set lock_timeout = '3s'` — מיגרציה שנתקעת מאחורי שאילתה ארוכה
  נועלת את הטבלה לכל התנועה.
- `create index concurrently` בפרודקשן.
