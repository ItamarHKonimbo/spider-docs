# Spider — פלואו השוואת מחירים

## מה זה ולמה צריך אותו

לקוחות Konimbo שמוכרים מוצרים צריכים לדעת בכל רגע נתון מה המחיר של המוצרים שלהם ביחס למתחרים — KSP, Bug, Ivory, ועוד. ניטור ידני של עשרות ומאות מוצרים בחנויות שונות הוא בלתי אפשרי.

פלואו ה-Spider פותר את זה אוטומטית: הוא מוציא מחירים עדכניים מחנויות מתחרות דרך שירות Spider-UI, מצלב אותם עם המוצרים שבחנות של הלקוח ב-Medusa, מחשב מחיר מומלץ למכירה, וכותב הכל ישירות ל-metadata של כל מוצר — כך שהמידע זמין בממשק הניהול ללא עבודה ידנית.

---

## ארכיטקטורה כללית

```
Baserow (קונפיגורציה)
        │
        ▼
  get_shops  ──► רשימת חנויות
        │
        ▼
  ╔═══════════════════════════════════════╗
  ║  לולאה על כל חנות                    ║
  ║  (parallel: false, skip_failures)     ║
  ║                                       ║
  ║  get_tags                             ║
  ║  (בודק status, מוצא tag לפי tag_name) ║
  ║       │                               ║
  ║       ▼                               ║
  ║  get_data_from_shop                   ║
  ║  (מוצרים לפי tag_id,                  ║
  ║   vendors נטענים מ-Spider API)        ║
  ║       │                               ║
  ║       ▼                               ║
  ║  create_set                           ║
  ║       │                               ║
  ║       ▼                               ║
  ║  search_urls ◄── Spider-UI API        ║
  ║       │                               ║
  ║       ├──► found                      ║
  ║       │      │                        ║
  ║       │      ▼                        ║
  ║       │  calculate_price              ║
  ║       │      │                        ║
  ║       └──► not_found                  ║
  ║  (כולל unsupported vendors + שגיאות) ║
  ║              │                        ║
  ║         post_medusa ──► Medusa        ║
  ║              │                        ║
  ║              ▼                        ║
  ║         sent_mail ──► Gmail           ║
  ╚═══════════════════════════════════════╝
```

**נתיב הפלואו:** `f/integrations/spider/spider_youleap`

---

## מוקדמות — מה צריך להיות מוגדר לפני הרצה

### 1. טבלת Baserow
בטבלה `2649` בכתובת `table.konimbo.co.il` צריכה להיות שורה לכל חנות Medusa.

| שדה | תיאור | דוגמה |
|-----|--------|--------|
| `שם החנות` | מזהה החנות | |
| `token` | Bearer JWT לאימות מול Medusa Admin API | ראה [Known Issue 3](#known-issue-3--בעיית-האימות-jok-token) |
| `base_url` | כתובת בסיס של Medusa Admin | `https://retail-os6-rms.youleap.com/` |
| `tag_name` | שם התג ב-Medusa שמסמן מוצרים לעיבוד | `spider` |
| `status` | האם החנות פעילה | `true` / `false` — חנות שאינה פעילה נדלגת בכוונה |
| `scope` | רמת העדכון ב-Medusa | `variant` (ברירת מחדל) או `product` |
| `calculated_expression` | נוסחת LiquidJS גלובלית לחנות | אופציונלי; ניתן לדרוס per-variant במטאדאטה |

### 2. Tag ב-Medusa
צריך להתקיים tag בשם התואם ל-`tag_name` שבשורת Baserow של אותה חנות. הפלואו מחפש את ה-tag ועובד **רק** על המוצרים שמתויגים בו.

> **מדוע Tag ולא Collection:** ב-Medusa, מוצר יכול להשתייך לקולקציה אחת בלבד. שימוש ב-tag מאפשר ללקוח לתייג מוצר ב-`spider` ובמקביל לשייכו לקולקציה משלו ללא הגבלה.

### 3. Metadata על variants ב-Medusa
כל variant שרוצים לכלול בהשוואה צריך metadata **תחת מפתח `spider`** במבנה הבא:

```json
{
  "spider": {
    "store_1": "ksp",
    "url_product_1": "https://ksp.co.il/web/cat/product/12345",
    "product_category_1": "tvs",
    "store_2": "bug",
    "url_product_2": "https://bug.co.il/p/67890",
    "product_category_2": "tvs",
    "calculated_expression": "{{ final_price | times: 0.9 }}"
  }
}
```

כלומר: לכל חנות מתחרה — triplet אחד של `store_N / url_product_N / product_category_N`.

השדה `calculated_expression` הוא **per-variant** — כל מוצר יכול לקבל נוסחה שונה. אם השדה לא קיים, ה-`calculated_expression` מ-Baserow (אם הוגדר) ישמש כ-fallback. אם גם הוא לא קיים — מחיר ה-Spider עובר ללא שינוי.

**רשימת חנויות נתמכות:** נטענת **דינמית** מ-Spider API לפי הקטגוריות הקיימות במוצרים — אין רשימה hardcoded בקוד. variant עם חנות שלא קיימת ב-Spider מסומן אוטומטית כ-`unsupported` ומופיע בדוח המייל.

> **הכנסת הנתונים הראשונית:** עדיין לא הוחלט מה המנגנון שמכניס את ה-metadata לחנות (Excel, פורם חיצוני, הכנסה ידנית). עד שהמנגנון יוחלט — יש להכניס ידנית דרך Medusa Admin.

---

## תיאור הצמתים

### צומת 1 — `get_shops`
**מה עושה:** שולח GET ל-Baserow API ומחזיר את רשימת כל החנויות המוגדרות.

**קלט:** אין

**פלט:** Baserow response מלא; הפלואו משתמש ב-`.results[]` — מערך של אובייקטי חנות

**כתובת API:** `https://table.konimbo.co.il/api/database/rows/table/2649/?user_field_names=true`

**הערה:** ה-Baserow API token מוגדר hardcoded בקוד הסקריפט.

---

### לולאה על חנויות
הפלואו רץ על כל חנות מהרשימה **ברצף** (`parallel: false`).
אם צומת כלשהו נכשל עבור חנות מסוימת, הלולאה **ממשיכה לחנות הבאה** (`skip_failures: true`). ראה [Known Issue 2](#known-issue-2--אין-התראה-על-כשלים-בזמן-ריצה).

---

### צומת 2 — `get_tags`
**מה עושה:** בודק אם החנות פעילה (`status`) ומחפש tag ב-Medusa Admin לפי `tag_name`.

**קלט:** קונפיגורציית החנות הנוכחית מהאיטרטור (`flow_input.iter.value`)

**פלט:**
- אם `status` לא פעיל: `{ skip: true, reason: "status_disabled" }`
- אם ה-tag לא נמצא: `{ error: "...", skip: true }`
- אם הצליח: `{ tag_id, tag_value, skip: false }`

**כתובת API:** `{base_url}admin/product-tags?q={tag_name}&limit=100`

**התאמה:** exact match לפי `value` של ה-tag (case-insensitive, trim).

**התנהגות בשגיאה:** מחזיר `{ error: "...", skip: true }` — הצומת הבא (`get_data_from_shop`) בודק את הדגל ומחזיר תוצאה ריקה, כך שהלולאה ממשיכה ללא קריסה.

---

### צומת 3 — `get_data_from_shop`
**מה עושה:** אם ה-tag תקין — מושך עד 1,000 מוצרים לפי `tag_id`, מזהה אילו חנויות Spider תומך בהן (קריאת API דינמית), ומסנן variants.

**קלט:** `tag_info` מהצומת הקודם; קונפיגורציית חנות

**לוגיקת vendors:** לכל קטגוריה שנמצאת במטאדאטה של המוצרים, שולח קריאה ל-`getdestributors` ב-Spider API ומקבל רשימת חנויות תמיכה. variant עם חנות שאינה ברשימה → `unsupported`.

**כתובת API (מוצרים):** `{base_url}admin/products?tag_id[]={tag_id}&limit=1000`

**כתובת API (vendors):** `https://spider-ui.rudenetworks.com/admin/v1/?method=getdestributors&category={category}&token=...`

**הגבלה:** עד 1,000 מוצרים בקריאה אחת. אם ה-collection תגדל יש צורך ב-pagination.

**פלט:**
```json
{
  "total": 45,
  "by_vendor": { "ksp": 20, "bug": 15 },
  "variants": [...],
  "unsupported": [
    {
      "product_id": "...", "variant_id": "...", "store": "חנות-לא-נתמכת",
      "url": "...", "price": -1, "description": "Store not supported by Spider: ..."
    }
  ]
}
```

> **⚠️ Wiring Bug (ריבוי חנויות):** לוקח `store_data[0]` מכל המערך. עובד עם חנות אחת. עם מספר חנויות — תמיד ישתמש בcredentials של חנות 1. ראה [Known Issue 1](#known-issue-1--wiring-שגוי-עם-ריבוי-חנויות).

---

### צומת 4 — `create_set`
**מה עושה:** מקבץ את ה-variants לפי `store|category` כדי למזער קריאות ל-Spider. מעביר את ה-`unsupported` הלאה ללא שינוי. מבצע deduplication לפי `product_id + variant_id + url`.

**פלט:** `{ groups: [{ store, category, data: [...] }], unsupported: [...] }`

---

### צומת 5 — `search_urls`
**מה עושה:** לכל קבוצה (store + category), שולח קריאה ל-Spider API ומצלב URLs. מוסיף את `unsupported` ישירות ל-`not_found`.

**נורמליזציית URL:** הסרת פרוטוקול (`https://`), trailing slash, lowercase.

**כתובת Spider API:**
```
https://spider-ui.rudenetworks.com/admin/v1/
  ?method=resultdestributor
  &distributor={store}
  &category={category}
  &token={hardcoded}
```

**טיפול בשגיאת רשת:** אם קריאה ל-Spider נכשלת, כל ה-variants של אותה קבוצה מתווספים ל-`not_found` עם `price = -1` וסיבת השגיאה — לא נעלמים בשקט.

**פלט:**
```json
{
  "found": [
    { "product_id": "...", "variant_id": "...", "store": "ksp",
      "url": "...", "price": 1499, "description": "..." }
  ],
  "not_found": [
    { "product_id": "...", "variant_id": "...", "store": "bug",
      "url": "...", "price": -1, "description": "Invalid or not found link in Spider" },
    { "...", "description": "Store not supported by Spider: ..." },
    { "...", "description": "שגיאת רשת..." }
  ]
}
```

---

### צומת 6 — `calculate_price`
**מה עושה:** לכל variant ב-`found` — מוצא את המחיר הנמוך ביותר בין כל החנויות שנמצאו, ואז מחשב `final_price` על ידי הפעלת `calculated_expression`.

**לוגיקת החישוב:**
1. **קיבוץ לפי `variant_id`** — כל השורות של אותו variant מקובצות יחד
2. **מחיר נמוך** — בוחר את המחיר הנמוך ביותר (`> 0`) מבין כל החנויות
3. **Expression** — מריץ `calculated_expression` מ-`metadata.spider` עם `final_price` כמשתנה
4. מוסיף `final_price` המחושב לכל השורות של ה-variant

**דוגמה:** ביטוי `{{ final_price | times: 0.9 }}` + KSP=1000₪, Bug=900₪ → `final_price` = 810₪

**פלט:** `{ total, processed, results: [...] }` — המערך בתוך מפתח `results`

---

### צומת 7 — `post_medusa`
**מה עושה:** כותב את כל תוצאות ההשוואה חזרה ל-Medusa. כל הבקשות נשלחות **במקביל** (`Promise.allSettled`).

**מה נכתב ל-metadata של כל variant (תחת מפתח `spider`):**

| שדה | תוכן |
|-----|------|
| `store_N` | שם החנות המתחרה |
| `url_product_N` | הקישור למוצר בחנות המתחרה |
| `product_category_N` | הקטגוריה ב-Spider |
| `price_N` | המחיר שנמצא ב-Spider (`-1` אם לא נמצא) |
| `final_price` | מחיר סופי לאחר חישוב expression (מגיע מ-`calculate_price`) |

**מבנה ה-payload:**
```json
{ "metadata": { "spider": { "store_1": "ksp", "url_product_1": "...", "price_1": "1499", "final_price": "1349" } } }
```

**Scope:** בהתאם לשדה `scope` ב-Baserow:
- `variant` (ברירת מחדל): `POST {base_url}admin/products/{product_id}/variants/{variant_id}`
- `product`: `POST {base_url}admin/products/{product_id}`

**פלט:** `{ total, successful, failed, results[] }`

> **⚠️ Wiring Bug (ריבוי חנויות):** גם `post_medusa` לוקח `store_data[0]` — אותה בעיה. ראה [Known Issue 1](#known-issue-1--wiring-שגוי-עם-ריבוי-חנויות).

---

### צומת 8 — `sent_mail`
**מה עושה:** שולח דוח מייל HTML עם רשימת המוצרים שלא נמצאו ב-Spider — כולל URLים שגויים, חנויות לא נתמכות, ושגיאות רשת.

**מצב נוכחי:** נשלח לאיתמר (`itamarh@konimbo.co.il`) לצורך בדיקה פנימית בלבד — זה אינו חלק מהמוצר הסופי.

**הגדרות SMTP:** Gmail + app password, מוגדרים hardcoded ב-flow.yaml.

> **עתידי (לא הוחלט):** כאשר הפלואו יעבוד per-store, יהיה צורך להחליט מי מקבל את הדוח — הלקוח, צוות Konimbo, או שניהם.

---

## Known Issues

### Known Issue 1 — Wiring שגוי עם ריבוי חנויות

**תיאור:**
הלולאה הראשית רצה על כל חנות מ-Baserow. `get_tags` מקבל נכון את החנות הנוכחית דרך `flow_input.iter.value`.

אך `get_data_from_shop` ו-`post_medusa` מקבלים את **כל** מערך החנויות (`results.j.results`) ולוקחים תמיד `[0]`.

**כשיש חנות אחת:** `[0]` = החנות היחידה = תקין.

**כשיהיו מספר חנויות:** בכל איטרציה, גם כשמעבדים חנות 2 או 3, שני הצמתים ישתמשו בcredentials של **חנות 1 בלבד** — מוצרים ייגרשו ועדכונים ייכתבו לחנות 1.

**תיקון:** לשנות ב-flow.yaml את ה-expression של `store_data` ב-`get_data_from_shop` ו-`post_medusa` מ-`results.j.results` ל-`flow_input.iter.value`.

**דחיפות:** לתקן לפני הוספת חנות שנייה ל-Baserow.

---

### Known Issue 2 — אין התראה על כשלים בזמן ריצה

**תיאור:**
כאשר חנות נכשלת מסיבות runtime (token פג, Medusa לא זמינה, בעיית רשת), `skip_failures: true` מדלג עליה בשקט. אין מייל, אין לוג מרכזי — רק שגיאה אדומה בממשק Windmill.

**שים לב:** חנות עם `status = false` ב-Baserow נדלגת **בכוונה** עם לוג ברור — זה לא בעיה. הבעיה היא כשלים **בלתי צפויים** בזמן ריצה.

**תיקון מוצע:** אסוף שמות חנויות שנכשלו לאורך הלולאה והוסף לדוח המייל שנשלח בסוף.

---

### Known Issue 3 — בעיית האימות (JOK Token)

**תיאור:** הפלואו מאמת מול Medusa Admin API עם **JOK Token** — Bearer JWT הנוצר מ-username/password. הטוקן נשמר ב-Baserow.

**למה זה בעייתי:**
- JOK tokens עלולים לפוג תוקף ולגרום לכשלים בזמן ריצה ללא אזהרה מוקדמת
- שמירת credentials ב-Baserow אינה פתרון מאובטח לסביבת production
- לא סקיילאבלי — לכל לקוח חדש צריך ליצור משתמש ב-Medusa, לייצר token ידנית ולהכניס ל-Baserow

**הסיבה לפתרון הנוכחי:** כרגע לא קיים מנגנון API Key ב-Medusa שמאפשר גישה ל-Admin API ללא משתמש אנושי.

**מה צריך לחקור:** האם Medusa תומכת ב-API Keys בגרסה נוכחית או עתידית; אחסון ב-Windmill Secrets.

---

## הפעלת הפלואו

### מצב נוכחי
הפלואו מופעל **ידנית** דרך ממשק Windmill.

**נתיב:** `f/integrations/spider/spider_youleap`

### אופן הפעלה עתידי
הפלואו יופעל דרך **Schedule** ב-Windmill — פעם או פעמיים ביום, על כל החנויות ברצף.

---

## Backlog — מה נותר לעשות

### Bugs לתיקון

1. **תקן wiring לריבוי חנויות** — שנה ב-flow.yaml את `store_data` של `get_data_from_shop` ו-`post_medusa` מ-`results.j.results` ל-`flow_input.iter.value`. דחוף לפני הוספת חנות שנייה. [פירוט](#known-issue-1--wiring-שגוי-עם-ריבוי-חנויות)

2. **התראה על כשלים בזמן ריצה** — אסוף שמות חנויות שנכשלו (runtime failures, לא status disabled) והוסף לדוח המייל. [פירוט](#known-issue-2--אין-התראה-על-כשלים-בזמן-ריצה)

### Design Decisions שממתינים להחלטה

3. **מנגנון הכנסת URLs** — Excel / פורם חיצוני לזין את ה-metadata הראשוני (`store_N / url_product_N / product_category_N / calculated_expression`) ל-Medusa.

4. **נמעני המייל הסופיים** — כרגע נשלח לאיתמר לבדיקה. לקבוע מי מקבל בproduction.

### שיפורים עתידיים

5. **Pagination** — תמיכה ב-tag עם יותר מ-1,000 מוצרים.

6. **אימות מאובטח** — החלפת JOK Token ב-API Key כאשר Medusa תתמוך בכך, ואחסון ב-Windmill Secrets.

7. **Webhook + per-store invocation** — הפרדת הרצות לפי חנות דרך webhook, הפעלה אוטומטית.

8. **דוח מפורט יותר** — הרחבת המייל לכלול מוצרים שהמחיר שלהם השתנה משמעותית.
