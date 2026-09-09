# ארכיטקטורת מערכת

## 1. עקרון-על: נתיב כתיבה יחיד, אכיפה כפולה

```
                        ┌──────────────────────────────────┐
   Browser (ES2025)     │  Flask API  (נתיב כתיבה יחיד)    │      Supabase
   ─────────────────    │  ─────────────────────────────   │      ─────────
   • Router             │  1. אימות JWT מקומי (JWKS)       │   ┌─ Postgres 17
   • Signal store   ───▶│  2. אכיפת הרשאה (RBAC)           │──▶│  + RLS
   • Web Components     │  3. ולידציית קלט (Pydantic)      │   │  + audit triggers
   • Design tokens      │  4. לוגיקה עסקית                 │   └─ Storage
        │               │  5. שאילתה תחת זהות המשתמש       │      Auth (GoTrue)
        │               └──────────────────────────────────┘         │
        │                              │                             │
        │                              ▼                             │
        │                    ┌──────────────────┐                    │
        │                    │  AI Layer        │                    │
        │                    │  • ML scoring    │                    │
        │                    │  • OpenAI        │                    │
        │                    └──────────────────┘                    │
        │                                                            │
        └──── Realtime (read-only, RLS) ─────────────────────────────┘
        └──── Auth: login/refresh/reset ─────────────────────────────┘
```

**כל כתיבה עוברת דרך Flask.** הדפדפן לעולם לא כותב ישירות ל-Postgres.
הסיבה: audit log, ולידציה, לוגיקה עסקית ו-side effects (התראות, AI) חייבים נקודת אכיפה אחת.

**כל קריאה נאכפת פעמיים.** Flask בודק הרשאה *וגם* Postgres אוכף RLS.
זה נשמע מיותר — הוא לא. באג בראוט של Flask (שכחת `WHERE org_id = ...`) הוא הבאג הנפוץ ביותר
בכל CRM. RLS הופך אותו משליפת נתונים של ארגון אחר לתוצאה ריקה.

### איך RLS חל על שאילתות Flask

Flask לא מתחבר כ-`postgres` או עם `service_role`. הוא מתחבר בתפקיד `crm_app` (לא-superuser,
`NOBYPASSRLS`), ובתחילת כל טרנזקציה מזריק את זהות המשתמש:

```sql
BEGIN;
  SET LOCAL ROLE authenticated;
  SET LOCAL request.jwt.claims = '{"sub":"...","org_id":"...","role":"sales_rep"}';
  -- כל שאילתה מכאן כפופה ל-RLS
COMMIT;
```

`SET LOCAL` מתאפס אוטומטית בסוף הטרנזקציה — אין דליפת זהות בין בקשות ב-connection pool.
ראה [ADR-001](adr/001-data-access-path.md).

## 2. Authentication — Supabase Auth, לא מימוש עצמי

**לא נבנה auth מאפס.** Supabase Auth (GoTrue) נותן: hashing תקני, refresh token rotation,
אימות מייל, שחזור סיסמה, MFA, נעילה אחרי ניסיונות כושלים. מימוש עצמי של כל אלה = כמה שבועות
ו-surface לפגיעויות.

זרימה:

```
1. Browser → Supabase Auth: signInWithPassword
2. Supabase → Browser: access_token (JWT, ES256, 1h) + refresh_token (httpOnly-ish)
3. Browser → Flask: Authorization: Bearer <access_token>
4. Flask: מאמת חתימה מול JWKS מקומי (cached) — ללא קריאת רשת לכל בקשה
5. Flask: שולף org_id + role מ-app_metadata של הטוקן
```

הטוקן נושא `app_metadata.org_id` ו-`app_metadata.role`, שנכתבים על ידי טריגר ב-DB
בעת יצירת משתמש. `app_metadata` — ולא `user_metadata` — כי המשתמש לא יכול לשנות אותו.

## 3. מבנה הקוד

```
aiakilov-crm/
├── backend/
│   ├── app/
│   │   ├── __init__.py            # application factory
│   │   ├── config.py              # הגדרות לפי סביבה
│   │   ├── extensions.py          # מופעים משותפים (pool, limiter, cache)
│   │   ├── core/
│   │   │   ├── auth.py            # אימות JWT, @require_auth
│   │   │   ├── rbac.py            # @require_permission
│   │   │   ├── db.py              # connection pool + RLS context manager
│   │   │   ├── errors.py          # היררכיית שגיאות + handler גלובלי
│   │   │   ├── schemas.py         # בסיסי Pydantic משותפים
│   │   │   ├── pagination.py      # cursor pagination
│   │   │   └── audit.py           # כתיבת audit log
│   │   ├── modules/               # מודול לכל bounded context
│   │   │   └── <context>/
│   │   │       ├── routes.py      # HTTP בלבד — ללא לוגיקה
│   │   │       ├── service.py     # לוגיקה עסקית
│   │   │       ├── repository.py  # SQL בלבד
│   │   │       ├── schemas.py     # Pydantic in/out
│   │   │       └── events.py      # side effects
│   │   └── ai/
│   │       ├── scoring/           # מודל ML פנימי
│   │       ├── llm/               # OpenAI: client, prompts, guardrails
│   │       └── automations/       # חוקים מתוזמנים
│   ├── migrations/                # SQL ממוספר, forward-only
│   └── tests/
├── frontend/
│   ├── index.html
│   ├── src/
│   │   ├── core/                  # router, store, http, events
│   │   ├── components/            # קומפוננטות UI לשימוש חוזר
│   │   ├── features/              # מסך לכל bounded context
│   │   ├── styles/                # tokens, base, utilities
│   │   └── main.js
│   └── tests/
├── docs/
└── .github/workflows/
```

### חוק השכבות (נאכף ב-CI)

```
routes  →  service  →  repository  →  DB
```

- `routes` לא נוגע ב-SQL ולא מכיל `if` עסקי. תפקידו: פענוח בקשה, קריאה ל-service, סריאליזציה.
- `service` לא מכיר `request`, `session` או כל אובייקט Flask. ניתן לבדיקה ללא HTTP.
- `repository` לא מכיל לוגיקה עסקית. רק SQL וממיפוי לשורות.

הפרה של החוק נתפסת בבדיקת import-linter ב-CI, לא בבדיקת עיניים ב-code review.

## 4. Environments

| סביבה | Supabase | ייעוד |
|---|---|---|
| local | Supabase CLI (Docker) | פיתוח יומיומי |
| preview | Supabase branch | לכל PR |
| production | פרויקט `aiakilov-crm` | חי |

מיגרציות הן **forward-only** וממוספרות. אין `down` migrations בפרודקשן — תיקון נעשה
במיגרציה חדשה. זה מונע את הכשל הנפוץ של rollback שמוחק נתונים.

## 5. מה נדחה בכוונה

| נדחה | למה |
|---|---|
| Microservices | אין את הבעיה שזה פותר. Modular monolith עם גבולות אכופים נותן 90% מהיתרון ב-10% מהמורכבות. |
| GraphQL | REST עם חוזה טוב מספיק. GraphQL מוסיף N+1, caching מורכב, ו-authz ברמת שדה. |
| Redis כתלות חובה | Rate limiting ו-cache מתחילים in-memory מאחורי interface. מחליפים ל-Redis כשיש יותר מ-instance אחד. |
| ORM (SQLAlchemy) | Repository עם SQL מפורש. שאילתות CRM הן analytics-heavy; ORM מסתיר את התוכנית ומייצר N+1. |
| Build step ב-frontend לפיתוח | ESM נייטיב. bundling רק ל-production. |
