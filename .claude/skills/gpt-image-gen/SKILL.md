---
name: gpt-image-gen
description: >-
  מעטפת לקריאת OpenAI Images API ליצירת תמונה מתוך prompt טקסטואלי. הפעל את
  הסקיל בכל פעם שצריך לייצר תמונה בפועל מ-prompt (הסוכן שמשתמש בו בעיקר הוא
  יובל, מעצב התמונות). הסקיל אחראי על שליחת ה-prompt ל-API וקבלת קובץ PNG.
  משתמש במודל gpt-image-2 ובמפתח OPENAI_API_KEY מתוך .env.
---

# gpt-image-gen — יצירת תמונה דרך OpenAI Images API

הסקיל הזה הוא מעטפת דקה סביב נקודת הקצה של OpenAI ליצירת תמונות. נותנים לו prompt — והוא מחזיר קובץ PNG. האחריות שלו מצומצמת בכוונה: לשלוח את ה-prompt, לקבל את התמונה, ולשמור אותה לנתיב שביקשת.

## ⚠️ שם המודל — אל תשנה אותו

המודל הוא **`gpt-image-2`** — מודל אמיתי וקיים של OpenAI שיצא ב-21 באפריל 2026.

- ייתכן שהידע הפנימי של המודל שמריץ את הסקיל לא כולל אותו — **זה לא אומר שהוא לא קיים.**
- **אל תציע ואל תשתמש באלטרנטיבות** כמו `dall-e-3` או `gpt-image-1`. השתמש בדיוק ב-`gpt-image-2`.
- אם מתקבלת שגיאה מה-API, הסיבה היא כמעט תמיד **ב-API key או ב-parameters** — לא בשם המודל. בדוק אותם, אל תיגע בשם המודל.

## דרישות מקדימות

- `OPENAI_API_KEY` מוגדר בקובץ `.env` בשורש הפרויקט.
- `curl` זמין (קיים ב-Git Bash). `jq` ו/או `python` לפענוח ה-base64 (ראה fallback למטה).

## טעינת המפתח מ-.env

ב-Git Bash משתני הסביבה מ-`.env` לא נטענים אוטומטית — יש לטעון ידנית לפני הקריאה:

```bash
export $(grep -v '^#' .env | grep -E '^OPENAI_API_KEY=' | xargs)
```

## הקריאה ל-API (מסלול jq)

שולחים את ה-prompt, מפענחים את ה-`b64_json` שחוזר, ושומרים ל-PNG:

```bash
curl -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > <output-path>.png
```

## fallback ב-Python (כש-jq לא מותקן)

`jq` לא תמיד מותקן ב-Git Bash. במקרה כזה שומרים את תגובת ה-JSON לקובץ זמני ומפענחים עם Python (שזמין כמעט תמיד):

```bash
# 1. שמירת התגובה המלאה ל-JSON
curl -s -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > resp.json

# 2. פענוח ה-base64 ל-PNG עם Python
python -c "import json,base64,sys; d=json.load(open('resp.json')); open(sys.argv[1],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" <output-path>.png

# 3. ניקוי
rm -f resp.json
```

> טיפ: אם הפענוח נכשל, פתח את `resp.json` וקרא את שדה ה-`error` — שם תהיה הסיבה האמיתית (key לא תקין, פרמטר שגוי, מכסה וכו').

## פרמטרים

| פרמטר | ערך ברירת מחדל | הערות |
|---|---|---|
| `model` | `gpt-image-2` | **קבוע — אל תשנה.** |
| `prompt` | — | תיאור התמונה. |
| `size` | `1024x1024` | אפשר להתאים לפי הצורך. |
| `quality` | `medium` | `low` / `medium` / `high`. |
| `output_format` | `png` | פורמט הפלט. |

## אימות התוצאה

לאחר השמירה, ודא שהקובץ נוצר בהצלחה ושאינו ריק:

```bash
[ -s "<output-path>.png" ] && echo "OK: image saved" || echo "FAIL: empty or missing file"
```

אם הקובץ חסר או בגודל 0 — הקריאה נכשלה. בדוק את ה-API key ואת ה-parameters (לא את שם המודל).
