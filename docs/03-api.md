# תכנון API

REST על בסיס `/api/v1`. גרסה ב-path ולא ב-header — קל יותר ל-debug, ל-caching ול-logs.

## 1. חוזה מעטפת אחיד

**כל** תשובה מוצלחת:

```json
{ "data": { }, "meta": { "request_id": "req_01J...", "took_ms": 34 } }
```

**כל** תשובת רשימה:

```json
{
  "data": [ ],
  "meta": {
    "request_id": "req_01J...",
    "cursor": { "next": "eyJpZCI6MTIz...", "has_more": true },
    "total_estimate": 1420
  }
}
```

`total_estimate` ולא `total` — `count(*)` מדויק על טבלה גדולה עם פילטרים הוא השאילתה
היקרה ביותר בכל מסך רשימה. אנחנו מחזירים הערכה מ-`pg_class.reltuples` כשאין פילטר,
וספירה מדויקת רק כשהמשתמש מבקש אותה במפורש (`?count=exact`).

**כל** שגיאה:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "הנתונים שנשלחו אינם תקינים",
    "details": [ { "field": "email", "code": "invalid_format" } ],
    "request_id": "req_01J..."
  }
}
```

`code` הוא מחרוזת יציבה שהלקוח מסתמך עליה. `message` מיועד לאדם וניתן לשינוי בלי לשבור
לקוחות. שדה `details` מובנה כדי שה-UI יוכל לצבוע את השדה הנכון בטופס.

### טבלת שגיאות

| HTTP | code | מתי |
|---|---|---|
| 400 | `validation_failed` | קלט לא תקין |
| 401 | `unauthenticated` | טוקן חסר / פג / חתימה שגויה |
| 403 | `forbidden` | מאומת אך חסר הרשאה |
| 404 | `not_found` | לא קיים **או** אין הרשאה לראות (מכוון — לא מדליפים קיום) |
| 409 | `conflict` | הפרת ייחודיות, כפילות ליד |
| 409 | `capacity_exceeded` | מחזור מלא |
| 412 | `stale_write` | `If-Match` לא תואם — עריכה מקבילה |
| 422 | `business_rule_violated` | קלט תקין אך אסור עסקית |
| 429 | `rate_limited` | חריגה ממכסה (+ `Retry-After`) |
| 500 | `internal_error` | באג — לעולם ללא פרטים פנימיים ללקוח |

## 2. Pagination — cursor בלבד

אין `?page=`/`?offset=`. עמוד 500 ב-OFFSET סורק 10,000 שורות מיותרות בכל בקשה.

```
GET /api/v1/leads?limit=50&sort=-last_activity_at&cursor=eyJ...
```

ה-cursor הוא base64 של `(sort_value, id)` — ה-tie-breaker על `id` הכרחי, אחרת שורות עם
אותו `last_activity_at` יידלגו או יופיעו פעמיים.

## 3. עדכונים מקביליים

כל משאב מחזיר `ETag` המבוסס על `updated_at`. `PATCH` דורש `If-Match`.
בלי זה, שני נציגים שפותחים אותו ליד — האחרון דורס בשקט את הראשון. זה תרחיש יומיומי
בצוות מכירות, לא קצה נדיר.

## 4. מפת נקודות קצה

### Identity

| Method | Path | הרשאה |
|---|---|---|
| `GET` | `/me` | מאומת |
| `PATCH` | `/me` | מאומת |
| `GET` | `/users` | `user.read` |
| `POST` | `/users/invite` | `user.invite` |
| `PATCH` | `/users/{id}` | `user.write` |
| `GET` | `/roles` | `role.read` |
| `PUT` | `/users/{id}/roles` | `role.assign` |

הרשמה, התחברות, רענון טוקן ואיפוס סיסמה **אינם** כאן — הם מול Supabase Auth ישירות.

### Pipeline

| Method | Path | הערות |
|---|---|---|
| `GET` | `/leads` | פילטרים, חיפוש, מיון |
| `POST` | `/leads` | מזהה כפילויות → 409 עם ה-id הקיים |
| `GET` | `/leads/{id}` | |
| `PATCH` | `/leads/{id}` | דורש `If-Match` |
| `DELETE` | `/leads/{id}` | soft delete |
| `POST` | `/leads/{id}/assign` | שיוך לנציג |
| `POST` | `/leads/{id}/status` | מעבר שלב + `reason` |
| `POST` | `/leads/{id}/convert` | ליד → תלמיד + הרשמה (אטומי) |
| `POST` | `/leads/bulk` | פעולות המוניות ≤ 500 |
| `POST` | `/leads/import` | קליטת CSV אסינכרונית |
| `GET` | `/leads/{id}/timeline` | |

**פילטרים** — תחביר מפורש ומוגבל, לא query builder כללי:

```
?status=new,contacted
&owner_id=42
&score_band=hot
&created_at[gte]=2026-01-01
&q=דנה
```

מנוע פילטרים גנרי בצד הלקוח הוא וקטור SQL injection ו-DoS. הרשימה כאן היא allowlist:
כל שדה מוצהר עם טיפוס ואופרטורים מותרים, וכל מה שלא ברשימה נדחה ב-400.

### Catalog / Enrollment / Billing

| Method | Path |
|---|---|
| `GET POST` | `/courses`, `/courses/{id}` |
| `GET POST` | `/cohorts`, `/cohorts/{id}` |
| `GET` | `/cohorts/{id}/roster` |
| `GET POST` | `/students`, `/students/{id}` |
| `POST` | `/enrollments` |
| `POST` | `/enrollments/{id}/cancel` |
| `GET POST` | `/payments` |
| `POST` | `/payments/{id}/mark-paid` |
| `GET` | `/payments/overdue` |

### Workflow / Engagement

| Method | Path |
|---|---|
| `GET POST PATCH` | `/tasks` |
| `POST` | `/tasks/{id}/complete` |
| `GET POST` | `/{entity}/{id}/notes` |
| `POST` | `/documents/upload-url` (signed URL) |
| `GET` | `/documents?entity_type=&entity_id=` |
| `GET` | `/notifications` |
| `POST` | `/notifications/read` |

העלאת קבצים **אינה** עוברת דרך Flask. הלקוח מבקש signed upload URL, מעלה ישירות
ל-Supabase Storage, ואז מודיע לשרת. כך אין קבצים גדולים בזיכרון של worker.

### Intelligence

| Method | Path | הערות |
|---|---|---|
| `GET` | `/leads/{id}/score` | ציון + הסבר תרומת פיצ'רים |
| `POST` | `/ai/summarize` | סיכום שיחה/טיימליין |
| `POST` | `/ai/draft` | טיוטת מייל / WhatsApp |
| `GET` | `/ai/recommendations` | פעולות מומלצות לנציג |
| `GET` | `/ai/at-risk` | לקוחות בסיכון |

כל נקודות ה-AI **מחזירות טיוטה בלבד**. אין נקודת קצה ששולחת מייל או WhatsApp שנוצר
ב-AI ללא אישור אנושי מפורש. ראה [ADR-005](adr/005-ai-human-in-the-loop.md).

### Platform

| Method | Path |
|---|---|
| `GET` | `/dashboard/summary` |
| `GET` | `/reports/{key}` |
| `GET` | `/audit-logs` |
| `GET POST` | `/saved-views` |
| `GET` | `/health`, `/health/ready` |

## 5. Rate limiting

מדורג לפי עלות, לא מכסה אחידה:

| קבוצה | מכסה |
|---|---|
| קריאה | 300/דקה למשתמש |
| כתיבה | 60/דקה למשתמש |
| bulk / import | 5/דקה למשתמש |
| AI | 20/דקה למשתמש, ותקרת עלות יומית לארגון |
| מאומת-לא (health) | 60/דקה ל-IP |

תקרת העלות ב-AI חשובה מהמכסה: לולאה באגית שקוראת ל-LLM היא הוצאה בלתי מוגבלת.

## 6. Idempotency

`POST` שיוצר משאב או מפעיל side effect מקבל `Idempotency-Key`. המפתח והתגובה נשמרים
24 שעות. בלי זה, לחיצה כפולה על "המר לתלמיד" יוצרת שתי הרשמות ושתי תוכניות תשלום.

## 7. תיעוד

OpenAPI 3.1 **נגזר מסכימות Pydantic**, לא נכתב ביד. מסמך שנכתב ביד מתיישן תוך שבועיים.
נבדק ב-CI: אם ראוט קיים ואינו מופיע ב-spec — ה-build נכשל.
