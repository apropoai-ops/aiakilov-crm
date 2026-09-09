# תוכנית בנייה

## Definition of Done — לכל מודול

מודול אינו "גמור" עד שכל השורות מסומנות:

- [ ] מיגרציה + RLS + אינדקסים
- [ ] Repository + Service + Routes (חוק השכבות נשמר)
- [ ] סכימות Pydantic ל-in/out
- [ ] בדיקות: unit + integration + RLS + API
- [ ] UI: כל 5 מצבי המסך
- [ ] נגישות: מקלדת + ניגודיות
- [ ] RTL נבדק
- [ ] audit log לכל כתיבה
- [ ] תיעוד עודכן
- [ ] CI ירוק, אפס אזהרות
- [ ] Commit + push

## שלבים

מספור: **M** = milestone. אין התחלה של M(n+1) לפני ש-M(n) עומד ב-DoD.

### M0 — יסודות
תשתית repo, CI, Supabase project, connection pool + הזרקת RLS, טיפול שגיאות,
מעטפת תגובה, request id, לוגים מובנים, health checks.
**מסך:** אין. **תוצר:** שלד שאפשר לפרוס.

### M1 — Identity
טבלאות RBAC, אימות JWT, `@require_permission`, הזמנת משתמשים, ניהול תפקידים.
**מסכים:** התחברות, הגדרות/משתמשים, הגדרות/תפקידים.
זה חייב להיות ראשון — כל שאר המערכת תלויה בו.

### M2 — Design System
Tokens, מצב כהה, RTL, קומפוננטות בסיס, router, signals, שכבת HTTP,
Toast, Skeleton, Empty/Error states, Command Palette.
**תוצר:** דף Storybook פנימי לכל הקומפוננטות.

### M3 — Catalog
קורסים, מחזורים, תפוסה.
מודול קטן שמאמת את כל המחסנית מקצה לקצה לפני שניגשים למודול המורכב.

### M4 — Pipeline (הליבה)
לידים: CRUD, פילטרים, חיפוש, טבלה וירטואלית, kanban, שיוך, מעברי סטטוס,
טיימליין, הערות, זיהוי כפילויות, ייבוא CSV.
**זהו המודול הגדול ביותר.** יפוצל לשלושה PR-ים.

### M5 — Enrollment
המרת ליד לתלמיד (אטומי), תלמידים, הרשמות, roster, אכיפת קיבולת.

### M6 — Billing
תוכניות תשלום, תשלומים, פיגורים, גבייה, תזרים.

### M7 — Workflow & Engagement
משימות, מסמכים, התראות, Realtime.

### M8 — Intelligence
Scoring heuristic + UI הסבר, LLM (סיכומים, טיוטות, המלצות), מנוע אוטומציות,
זיהוי סיכון. אימון ML נכנס כשהנתונים מצדיקים.

### M9 — Analytics
דשבורד, דוחות, משפך, ביצועי נציגים.

### M10 — הקשחה
בדיקות עומס, כוונון אינדקסים לפי `pg_stat_statements`, סקר אבטחה, גיבויים
ותרגיל שחזור, runbook, ניטור.

## עבודת Git

- ענף לכל feature: `feat/m4-leads-table`
- Conventional Commits
- Commit אחרי כל יחידה שעומדת ב-DoD
- אין push ישיר ל-`main`

## תיעוד

Obsidian הוא היעד. **שרת ה-MCP של Obsidian אינו מחובר כרגע** (`CONNECTION_CLOSED`),
לכן התיעוד נכתב ל-`docs/` כ-Markdown תקני עם קישורי wiki. ברגע שהחיבור יתוקן, ניתן
להצביע על התיקייה כ-Vault או להעתיק אותה פנימה — לא יאבד תוכן.
