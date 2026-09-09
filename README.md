# AIAKILOV CRM

מערכת CRM לניהול לידים, תלמידים והרשמות לקורסי AI.

> **סטטוס: אפיון בלבד.** טרם נכתב קוד. ה-repo מכיל את מסמכי התכנון שאושרו,
> ומהווה את הבסיס ל-M0 (יסודות).

## Stack

| שכבה | טכנולוגיה |
|---|---|
| Frontend | HTML5, CSS3, JavaScript ES2025 — ללא framework ([ADR-002](docs/adr/002-no-framework-frontend.md)) |
| Backend | Python, Flask — modular monolith |
| Database | Supabase (Postgres 17) + RLS |
| Data access | `psycopg 3` + repository, ללא ORM ([ADR-003](docs/adr/003-no-orm.md)) |
| Auth | Supabase Auth (JWT, ES256), אימות מקומי מול JWKS |
| AI | scoring פנימי (heuristic → ML) + OpenAI לטיוטות |

## עקרונות מנחים

1. **נתיב כתיבה יחיד** — הדפדפן לעולם לא כותב ל-DB. כל כתיבה עוברת דרך Flask.
2. **אכיפה כפולה** — Flask בודק הרשאה *וגם* Postgres אוכף RLS. Flask מתחבר כ-`authenticated`
   ומזריק את זהות המשתמש ב-`SET LOCAL`, כך שגם שאילתות השרת כפופות ל-RLS.
   ראוט ששכח `WHERE org_id` מחזיר רשימה ריקה, לא נתונים של ארגון אחר. ([ADR-001](docs/adr/001-data-access-path.md))
3. **AI מציע, אדם מאשר** — אין נתיב שבו טקסט שנוצר ב-LLM מגיע ללקוח ללא אישור. ([ADR-005](docs/adr/005-ai-human-in-the-loop.md))
4. **חוק השכבות** — `routes → service → repository → DB`, נאכף ב-CI ולא בביקורת קוד.

## תיעוד

| מסמך | תוכן |
|---|---|
| [00-overview](docs/00-overview.md) | אפיון על, personas, bounded contexts, החלטות מוצר |
| [01-architecture](docs/01-architecture.md) | ארכיטקטורה, זרימת בקשה, מבנה קוד |
| [02-database](docs/02-database.md) | סכימה, אינדקסים, RLS |
| [03-api](docs/03-api.md) | חוזה API, שגיאות, pagination |
| [04-frontend](docs/04-frontend.md) | ארכיטקטורת לקוח, design system, UI/UX |
| [05-ai](docs/05-ai.md) | scoring, LLM, אוטומציות |
| [06-security](docs/06-security.md) | auth, RBAC, הקשחה |
| [07-testing](docs/07-testing.md) | אסטרטגיית בדיקות ו-CI |
| [08-roadmap](docs/08-roadmap.md) | שלבי בנייה ו-Definition of Done |
| [adr/](docs/adr/) | Architecture Decision Records |

## החלטות שנסגרו

| נושא | הוחלט |
|---|---|
| Multi-tenancy | `org_id` בכל טבלה מהיום הראשון |
| שפת ממשק | עברית בלבד, RTL |
| תשלומים | רישום בלבד — **אין נתוני כרטיס אשראי במערכת**, ולכן אין חשיפת PCI-DSS |
| WhatsApp | יצירת טקסט בלבד, ללא Business API |

## פתוח

- SSO ארגוני (Google Workspace)? — נדרש עד M1
- מקורות קליטת לידים? — נדרש עד M4
