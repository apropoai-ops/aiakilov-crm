# ADR-001 — נתיב גישה לנתונים

**סטטוס:** מוצע · 2026-09-09

## הקשר

עם Supabase יש שלוש דרכים לגשת לנתונים מהדפדפן.

## אפשרויות

### א. הדפדפן ניגש ישירות ל-Supabase (PostgREST), Flask רק ל-AI
- ✅ פחות קוד, Realtime בחינם, RLS אוכף
- ❌ הלוגיקה העסקית נודדת ל-DB (טריגרים, RPC) או לדפדפן — ששם היא לא נאכפת
- ❌ audit log חלקי, קשה לרכז side effects (התראות, scoring)
- ❌ קשה לבצע rate limiting משמעותי

### ב. Flask הוא ה-API היחיד, מתחבר עם `service_role`
- ✅ נקודת אכיפה אחת, לוגיקה מרוכזת
- ❌ `service_role` עוקף RLS — כל באג ב-`WHERE` הוא דליפה בין ארגונים
- ❌ מוותר על רשת הביטחון החזקה ביותר ש-Postgres נותן

### ג. Flask הוא ה-API היחיד, מתחבר כ-`authenticated` עם הזרקת claims ← **נבחר**
- ✅ לוגיקה מרוכזת **וגם** RLS נאכף על שאילתות Flask
- ✅ באג ב-`WHERE` מחזיר תוצאה ריקה במקום נתוני ארגון אחר
- ✅ audit, rate limiting ו-side effects בנקודה אחת
- ❌ מעט יותר מורכבות בשכבת החיבור (~40 שורות, נכתב פעם אחת)
- ❌ מדיניות RLS חייבת להיות אופטימלית, אחרת מחיר ביצועים

## החלטה

אפשרות ג'. כל טרנזקציה נפתחת עם:

```sql
SET LOCAL ROLE authenticated;
SET LOCAL request.jwt.claims = '<claims json>';
```

`SET LOCAL` מתאפס ב-`COMMIT` — בטוח עם Supavisor במצב transaction.

Realtime נשאר חיבור ישיר מהדפדפן ל-Supabase, לקריאה בלבד, תחת RLS.

## השלכות

- ה-connection pool מתחבר בתפקיד `crm_app` עם `NOBYPASSRLS`.
- מפתח ה-`service_role` אינו נטען כלל בשרת האפליקציה — רק במשימות תחזוקה מבודדות.
- כל מדיניות RLS חייבת אינדקס תואם. נבדק ב-CI.
