# אבטחה

## 1. Authentication

Supabase Auth (GoTrue) מנפיק JWT בחתימת **ES256** (מפתחות אסימטריים).
Flask מאמת מקומית מול JWKS שנשמר ב-cache — אין קריאת רשת לכל בקשה, ואין secret משותף
שדליפתו מאפשרת זיוף טוקנים.

| רכיב | ערך |
|---|---|
| Access token | 1 שעה |
| Refresh token | 30 יום, רוטציה בכל שימוש |
| שמירה בלקוח | בזיכרון + refresh ב-cookie `HttpOnly; Secure; SameSite=Lax` |
| MFA | TOTP, חובה ל-`owner` ו-`admin` |

**למה לא `localStorage`:** כל XSS הופך לגניבת סשן מלאה. טוקן בזיכרון נעלם ברענון עמוד,
וה-refresh ב-cookie שאינו נגיש ל-JS מחזיר את הסשן.

### הצהרות בטוקן

`org_id` ו-`role` נכתבים ל-`app_metadata` בלבד — הוא בשליטת השרת.
`user_metadata` נשלט על ידי המשתמש; הסתמכות עליו לצורך הרשאה היא הסלמת הרשאות בשורה אחת.

## 2. Authorization — שלוש שכבות

```
UI       → מסתיר פעולות (נוחות בלבד, לא אבטחה)
Flask    → @require_permission('lead.write')  ← אכיפה
Postgres → RLS policy                          ← אכיפה
```

השכבה השלישית היא זו שמצילה. הבאג הנפוץ ביותר ב-CRM הוא ראוט ששכח `where org_id`.
עם RLS התוצאה היא רשימה ריקה, לא נתוני ארגון אחר.

## 3. ולידציית קלט

Pydantic v2 על **כל** גוף בקשה, query param ו-path param.

- שדות לא מוכרים → נדחים (`extra='forbid'`), לא מתעלמים.
- מגבלות אורך על כל שדה טקסט — בלעדיהן, שדה `notes` הוא DoS.
- טלפון מנורמל ל-E.164, מייל ל-lowercase, לפני כתיבה.
- העלאות: סוג נבדק לפי **magic bytes**, לא לפי סיומת או `Content-Type`.

## 4. SQL Injection

`psycopg` עם פרמטרים בלבד. שרשור מחרוזות ל-SQL נאסר ונתפס ב-CI (`bandit`).
שמות טבלאות/עמודות דינמיים (מיון) עוברים **allowlist מפורש** — לא escaping.

## 5. XSS

- אין `innerHTML` בשום מקום. `textContent` ו-`<template>` בלבד. נאכף ב-ESLint.
- CSP קשיח, ללא `unsafe-inline`:

```
default-src 'self';
script-src  'self';
style-src   'self';
img-src     'self' data: https://*.supabase.co;
connect-src 'self' https://*.supabase.co;
frame-ancestors 'none';
base-uri 'none';
object-src 'none';
```

- טקסט שנוצר ב-AI מטופל כקלט לא מהימן לכל דבר.

## 6. CSRF

ה-API הוא Bearer-token ו-stateless — לא פגיע ל-CSRF קלאסי.
נקודת החשיפה היחידה היא ה-refresh cookie, שמוגן ב-`SameSite=Lax` + בדיקת `Origin`.

## 7. הצפנת מידע רגיש

| נתון | טיפול |
|---|---|
| סיסמאות | לא אצלנו כלל (Supabase Auth) |
| ת"ז | AES-256-GCM ביישום, המפתח ב-secrets manager |
| מסמכים | Storage פרטי, signed URL ל-15 דקות |
| מפתחות API | ENV בלבד, לעולם לא ב-repo |
| בסיס נתונים | הצפנה at-rest (Supabase) |

`git-secrets` כ-pre-commit hook. מפתח שנדחף ל-repo נחשב דלוף — גם אחרי מחיקה.

## 8. Audit

כל כתיבה נרשמת: מי, מה, מתי, מ-IP, לפני/אחרי, `request_id`.
ה-log הוא **append-only** — אין `UPDATE`/`DELETE` (נאכף בהרשאות טבלה).
audit log שניתן לערוך אינו audit log.

## 9. Headers

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

## 10. תלויות

`pip-audit` ו-`npm audit` ב-CI. גרסאות ננעלות ב-lockfile.
CVE בחומרה גבוהה חוסם merge.

## 11. פרטיות

- מחיקת ליד היא soft delete; מחיקה מלאה לפי בקשה היא תהליך מפורש עם אישור admin.
- ייצוא נתוני נושא מידע נתמך (`GET /me/export`).
- שמירה: `audit_logs` 24 חודשים, `ai_interactions` 12 חודשים.
- קליטת לידים מטפסים דורשת רישום הסכמה (`source_detail.consent`) — חובה לפי חוק הספאם
  לפני שליחת דיוור.

## 12. היקף — נסגר 2026-09-09

**המערכת אינה מחזיקה נתוני כרטיס אשראי.** מודול התשלומים רושם תשלומים שבוצעו במקום
אחר ושומר אסמכתה (`payments.external_ref`) בלבד. לכן PCI-DSS אינו חל על המערכת.

זה אינו רק צמצום scope — זו החלטת אבטחה. אחסון PAN היה מטיל דרישות ביקורת, סגמנטציית
רשת והצפנה על **כל** המערכת, לא רק על מודול התשלומים.

אכיפה: בדיקת CI דוחה סכימה שמכילה עמודה בשם המרמז על PAN/CVV, ו-`bandit` מסמן
שדות קלט שנראים כמספרי כרטיס.

### פתוח

- 🟡 SSO ארגוני (Google Workspace)? Supabase תומך, אך משנה את זרימת ניהול המשתמשים.
  נדרשת תשובה עד תחילת **M1**.
