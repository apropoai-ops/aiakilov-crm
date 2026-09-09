# ארכיטקטורת Frontend ו-UI/UX

Vanilla ES2025, ללא framework. זו אילוץ שהוגדר — והוא סביר, אך דורש שנבנה ארבעה
דברים שה-framework היה נותן בחינם. נבנה אותם פעם אחת, קטן ונבדק.

## 1. ארבעת אבני היסוד

### 1.1 Signals — ריאקטיביות ב-~120 שורות

```js
// core/signal.js
export function signal(initial) {
  let value = initial;
  const subs = new Set();
  const read = () => { if (activeEffect) subs.add(activeEffect); return value; };
  read.set = (next) => {
    if (Object.is(value, next)) return;      // אין רינדור מיותר
    value = next;
    for (const fn of [...subs]) schedule(fn); // batched על microtask
  };
  read.subscribe = (fn) => { subs.add(fn); return () => subs.delete(fn); };
  return read;
}
```

עדכונים מקובצים ל-microtask אחד. בלי זה, עדכון של 5 שדות בטופס מפעיל 5 רינדורים.

### 1.2 Router — History API

Path-based (`/leads/123`), לא hash. תומך ב-nested routes, ב-guards לפי הרשאה,
וב-lazy loading של מודול המסך דרך `import()` דינמי.

### 1.3 בסיס קומפוננטה — Custom Elements ללא Shadow DOM

```js
export class Component extends HTMLElement {
  connectedCallback() { this._cleanup = effect(() => this.render()); }
  disconnectedCallback() { this._cleanup?.(); }   // ללא זה — דליפת זיכרון
}
```

**החלטה: אין Shadow DOM.** הוא מבודד CSS, אבל חוסם עיצוב גלובלי, מסבך טפסים
(`form` לא חוצה shadow boundary), פוגע ב-RTL ומקשה על מצב כהה. במקום זאת: מוסכמת
מרחב-שמות ב-CSS (`.crm-<component>__<part>`) ו-`@layer` לשליטה בסדר.

### 1.4 שכבת HTTP

מודול יחיד שאחראי על: הזרקת `Authorization`, רענון טוקן אוטומטי ב-401 (עם תור בקשות
כדי שלא יופעלו 12 רענונים במקביל), `AbortController` לכל בקשה, מיפוי שגיאות למודל אחיד,
ו-retry עם exponential backoff על 429/503 בלבד.

## 2. רינדור רשימות גדולות

טבלת לידים היא המסך המרכזי ותכיל עשרות אלפי שורות. הפתרון: **virtual scrolling** —
רינדור של חלון גלוי + buffer בלבד. שורה = `<template>` משוכפל, לא `innerHTML` (שהוא
גם איטי וגם וקטור XSS).

## 3. מבנה

```
frontend/src/
├── core/          signal, router, http, events, i18n, permissions
├── components/    button, input, select, modal, drawer, table, toast,
│                  skeleton, empty-state, error-state, badge, avatar,
│                  date-picker, combobox, tabs, dropdown, pagination
├── features/      dashboard/ leads/ students/ courses/ payments/
│                  tasks/ reports/ settings/
├── styles/        tokens.css, base.css, layers.css, utilities.css
└── main.js
```

`components/` לא יודע דבר על CRM. `features/` מרכיב אותם. זהו הגבול שמונע כפילות.

## 4. Design System

### 4.1 Tokens — שתי שכבות

**פרימיטיבים** (לא בשימוש ישיר) → **סמנטיים** (מה שהקוד משתמש בו):

```css
:root {
  /* primitives */
  --blue-600:#2563eb; --slate-50:#f8fafc; --slate-900:#0f172a;
  /* semantic */
  --color-bg: var(--slate-50);
  --color-surface: #fff;
  --color-text: var(--slate-900);
  --color-accent: var(--blue-600);
  --radius-md: 10px;
  --space-4: 16px;
  --shadow-md: 0 4px 12px -2px rgb(15 23 42 / .08);
}
:root[data-theme="dark"] {
  --color-bg: #0b1120; --color-surface: #111827; --color-text: #e5e7eb;
}
```

מצב כהה משנה **רק** את השכבה הסמנטית. קומפוננטה שכותבת `#fff` ישירות — נכשלת ב-lint.

### 4.2 RTL

`margin-inline-start` ולא `margin-left`. `inset-inline-end` ולא `right`.
Logical properties נותנות תמיכה דו-כיוונית ללא גיליון CSS שני. מספרים, מטבע ותאריכים
דרך `Intl.NumberFormat` / `Intl.DateTimeFormat` עם `he-IL`.

### 4.3 Glassmorphism — במידה

`backdrop-filter: blur()` יפה, אבל הוא compositing יקר. נשתמש בו על משטחים
**קטנים ולא-גוללים בלבד**: header, command palette, modal overlay, toast.
לעולם לא על רקע טבלה גוללת — שם הוא הורג את קצב הפריימים.

### 4.4 אנימציות

- מעברים 150–250ms, `cubic-bezier(.4,0,.2,1)`.
- `transform` ו-`opacity` בלבד. אנימציה של `width`/`top` גורמת ל-layout בכל פריים.
- `@media (prefers-reduced-motion: reduce)` מבטל הכל — לא אופציונלי.

## 5. מצבי מסך — כולם, לכל מסך

לכל מסך שמביא נתונים יש **חמישה** מצבים, וכולם מיושמים לפני שהמסך נחשב גמור:

| מצב | טיפול |
|---|---|
| Loading | Skeleton התואם למבנה התוכן, לא ספינר |
| Empty (אין נתונים) | הסבר + פעולה ראשית ("הוסף ליד ראשון") |
| Empty (פילטר לא החזיר) | שונה מהקודם! + "נקה פילטרים" |
| Error | הודעה + "נסה שוב" + `request_id` להעתקה |
| Partial | נתונים ישנים + באנר "מציג מידע מלפני 2 דקות" |

ההבחנה בין שני מצבי ה-Empty היא ההבדל בין מוצר שנראה גמור לכזה שלא.

## 6. תוכנית מסכים

| מסך | תוכן עיקרי |
|---|---|
| Dashboard | KPI (לידים חדשים, יחס המרה, תפוסת מחזור, גבייה), משפך, משימות היום, לידים חמים |
| Leads — Table | טבלה וירטואלית, פילטרים, בחירה מרובה, פעולות המוניות |
| Leads — Kanban | גרירה בין שלבים, WIP לכל עמודה |
| Lead detail | פאנל ימני: פרטים · ציון AI + הסבר · פעולות. מרכז: טיימליין, הערות, מסמכים, משימות |
| Students | רשימה + כרטיס תלמיד + היסטוריית הרשמות |
| Courses / Cohorts | קורסים, מחזורים, תפוסה חזותית, roster |
| Payments | לוח גבייה, פיגורים, תחזית תזרים |
| Tasks | היום / השבוע / באיחור |
| Reports | משפך, ביצועי נציגים, ROI לפי מקור ליד |
| Settings | משתמשים, תפקידים והרשאות, שדות, אינטגרציות |

### Command Palette (`Ctrl/Cmd+K`)

ניווט וחיפוש גלובלי. במערכת שנציג עובד בה 6 שעות ביום, זה חוסך יותר זמן מכל אנימציה.

## 7. נגישות

- כל אינטראקציה זמינה במקלדת. `:focus-visible` נראה תמיד.
- ניגודיות ≥ 4.5:1 בשני המצבים — נבדק אוטומטית ב-CI.
- Modal: מלכודת פוקוס, `Esc` לסגירה, החזרת פוקוס לאלמנט הפותח.
- `aria-live` להתראות Toast.

## 8. תקציב ביצועים (נאכף ב-CI)

| מדד | תקרה |
|---|---|
| JS ראשוני (gzip) | 60KB |
| LCP | < 1.5s |
| INP | < 200ms |
| CLS | < 0.05 |
| רינדור טבלה, 10K שורות | < 100ms |

חריגה = כשל build, לא הערה ב-review.
