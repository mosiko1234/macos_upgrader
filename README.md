# מערכת עדכוני macOS לרשתות מבודדות (Air-Gapped)

## סקירה כללית

מערכת זו מאפשרת למחשבי Mac עם שבבי Apple Silicon ברשתות מבודדות (ללא חיבור אינטרנט) לקבל עדכוני macOS באמצעות פרוקסי חתימה קריפטוגרפית ושרת תוכן מקומי. המערכת מיירטת בקשות חתימה ל-TSS (Tatsu Signing Server) של Apple, מבצעת חתימה אופליין באמצעות העברת מדיה פיזית (USB), ומספקת כרטיסי חתימה חתומים לאפשר התקנת עדכונים.

## ארכיטקטורה

### רכיבים עיקריים

```
רשת מבודדת                        רשת חיצונית (אינטרנט)
┌─────────────────────┐              ┌──────────────────┐
│  Client Mac         │              │  Signer Script   │
│  (Apple Silicon)    │              │  + BuildManifest │
└──────────┬──────────┘              └────────┬─────────┘
           │                                  │
           │ בקשת TSS                         │ בקשת TSS
           ▼                                  ▼
┌─────────────────────────────────┐  ┌──────────────────┐
│  שרת פנימי (אותו מארח)          │  │  Apple TSS       │
│  ┌───────────────────────────┐  │  │  (gs.apple.com)  │
│  │ Proxy Server :443         │◄─┼──┤  העברה ב-USB     │
│  │ (gs.apple.com TSS)        │  │  └──────────────────┘
│  │ ┌───────────────────────┐ │  │
│  │ │ Ticket Cache          │ │  │
│  │ └───────────────────────┘ │  │
│  │ ┌───────────────────────┐ │  │
│  │ │ Pending Queue         │ │  │
│  │ └───────────────────────┘ │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ Reposado Server :8088     │  │
│  │ (אספקת תוכן)              │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 1. Proxy Server (שרת פרוקסי)

**תפקיד**: מיירט בקשות TSS מ-Client Macs ומספק כרטיסי חתימה

**טכנולוגיה**: Python + Flask

**פורט**: 443 (HTTPS)

**פונקציונליות עיקרית**:
- מיירט בקשות HTTPS ל-gs.apple.com/TSS/controller
- מנתח בקשות TSS ומחלץ ECID, ApNonce, BuildManifest
- בודק אם קיים כרטיס חתום במטמון (Cache)
- אם קיים - מחזיר מיד את הכרטיס (HTTP 200)
- אם לא קיים - שומר את הבקשה לעיבוד חיצוני (HTTP 503)
- מנהל מטמון כרטיסים עם תפוגה של 7 ימים
- **שולח התראות בזמן אמת** על בקשות חדשות (קריטי בגלל חלון 15 דקות!)

### 2. Reposado Content Server (שרת תוכן)

**תפקיד**: מארח קבצי עדכון macOS וקטלוגים

**טכנולוגיה**: Reposado (כלי Python לשכפול עדכוני Apple)

**פורט**: 8088 (HTTP)

**תוכן**:
- קבצי .pkg (חבילות עדכון)
- קבצי .ipsw (קושחה)
- קטלוגי עדכונים

**הערה חשובה**: התוכן מסונכרן מהאינטרנט ומועבר לרשת המבודדת באמצעות USB

### 3. Signer Script (סקריפט חתימה חיצוני)

**תפקיד**: מקבל כרטיסי חתימה מ-Apple TSS בעבור בקשות ממתינות

**טכנולוגיה**: Python + tsschecker

**מיקום**: מערכת מחוברת לאינטרנט

**תהליך**:
1. קורא קבצי JSON של בקשות ממתינות מ-USB
2. מפעיל tsschecker עם BuildManifest לכל בקשה
3. מתקשר עם שרתי TSS האמיתיים של Apple
4. שומר כרטיסים חתומים עם שמות קבצים לפי ECID/ApNonce
5. מעתיק כרטיסים חזרה ל-USB

**דרישה קריטית**: נדרש קובץ BuildManifest.plist שמחולץ מקובץ IPSW של גרסת macOS היעד

### 4. Certificate Management (ניהול אישורים)

**תפקיד**: יצירת אישורי TLS להקמת אמון עם Client Macs

**רכיבים**:
- Root CA (רשות אישורים שורש) - עצמי-חתום
- Leaf Certificate (אישור עלה) - עבור gs.apple.com
- סקריפטים להתקנה ב-System Keychain

**חשיבות**: Client Macs חייבים לסמוך על האישור כדי לקבל חיבורי HTTPS למיירט

## תהליך עבודה מלא

### שלב 0: הגדרת Client Mac

1. התקנת Root CA ב-System Keychain (אמון ברמת מערכת)
2. הגדרת DNS כך ש-gs.apple.com יפנה לשרת הפרוקסי הפנימי
3. הגדרת CatalogURL להצביע על שרת Reposado הפנימי:
   ```bash
   sudo defaults write /Library/Preferences/com.apple.SoftwareUpdate CatalogURL \
     "http://internal-server:8088/content/catalogs/..."
   ```

### שלב 1: גילוי תוכן (ברשת המבודדת)

1. Client Mac שואל את שרת Reposado לגבי עדכונים זמינים
2. Client מוריד מטא-דאטה ו-BuildManifest
3. Client מזהה שעדכון זמין

### שלב 2: מיירוט בקשת TSS (ברשת המבודדת)

1. Client Mac מתחיל התקנת עדכון ושולח בקשת TSS ל-gs.apple.com
2. DNS מפנה לשרת הפרוקסי
3. שרת הפרוקסי מקבל בקשה דרך TLS
4. הבקשה מנותחת ו-ECID/ApNonce/BuildManifest מחולצים
5. בדיקת מטמון

### שלב 3a: Cache Hit (כרטיס קיים במטמון)

1. כרטיס נמצא במטמון
2. תגובת TSS נבנית עם הכרטיס
3. תגובה מוחזרת ל-Client Mac (HTTP 200)
4. Client מוריד תוכן משרת Reposado
5. התקנת העדכון ממשיכה

### שלב 3b: Cache Miss (כרטיס לא קיים)

1. מטא-דאטה של הבקשה נשמרת לקובץ JSON בתיקיית pending
2. HTTP 503 Service Unavailable מוחזר ל-Client Mac (ינסה שוב מאוחר יותר)
3. מנהל מערכת מייצא בקשות ממתינות ל-USB **מיד**

### שלב 4: חתימה חיצונית (רשת מחוברת לאינטרנט) - **דחוף! תוך 15 דקות**

⚠️ **קריטי**: כרטיסי TSS תקפים רק ל-**15 דקות** מרגע יצירת ה-ApNonce!

1. כונן USB מחובר **מיד** למערכת חיצונית
2. מנהל מערכת מספק BuildManifest.plist לגרסת macOS היעד
3. Signer Script קורא קבצי בקשות ממתינות
4. לכל בקשה, tsschecker מופעל עם ECID, ApNonce, ונתיב BuildManifest
5. כרטיסים חתומים נשמרים עם שמות ECID/ApNonce
6. כרטיסים מועתקים ל-USB

**חלון זמן**: כל התהליך (שלבים 3b-5) חייב להסתיים תוך **15 דקות** מרגע שה-Client Mac יצר את הבקשה!

### שלב 5: ייבוא כרטיסים (ברשת המבודדת) - **מיידי**

1. כונן USB מוחזר **מיד** לרשת המבודדת
2. מנהל מערכת מייבא כרטיסים לשרת הפרוקסי
3. כרטיסים נשמרים במטמון
4. בקשות ממתינות מוסרות מהתור
5. Client Mac מנסה שוב ומקבל כרטיס מהמטמון
6. Client מוריד תוכן מ-Reposado ומשלים התקנה

**הערה**: לאחר ייבוא, הכרטיס נשאר תקף במטמון ל-7 ימים לשימוש חוזר

## דרישות מערכת

### שרת פנימי (רשת מבודדת)

**חומרה**:
- מעבד: 2+ ליבות
- זיכרון: 4GB RAM מינימום
- דיסק: 100GB מינימום (למטמון ותור)
- רשת: IP סטטי ברשת הפנימית

**תוכנה**:
- מערכת הפעלה: Ubuntu 22.04 LTS או macOS Server
- Python: 3.10 ומעלה
- Flask, plistlib, SQLite
- Reposado

### מערכת חתימה חיצונית

**חומרה**:
- מעבד: 2+ ליבות
- זיכרון: 2GB RAM
- דיסק: 20GB
- רשת: גישה לאינטרנט (gs.apple.com)

**תוכנה**:
- מערכת הפעלה: Ubuntu 22.04 LTS או macOS
- Python: 3.10 ומעלה
- tsschecker או ipsw CLI

### Client Mac

**דרישות**:
- macOS 12.0 (Monterey) ומעלה
- Apple Silicon (M1/M2/M3/M4)
- גישת רשת לשרת הפרוקסי ושרת Reposado

## אבטחה

### אבטחת TLS/אישורים

- שימוש במפתחות RSA 2048-bit או ECDSA 256-bit
- אלגוריתם חתימה: SHA-256 ומעלה
- תוקף: 10 שנים ל-Root CA, שנתיים ל-Leaf Certificate
- הרשאות קובץ: 0600 למפתחות פרטיים

### אבטחת רשת

- הגבלת גישה לשרת הפרוקסי לרשת הפנימית בלבד
- חסימת גישה ישירה לאינטרנט מ-Client Macs
- שימוש ב-DNS מקומי או /etc/hosts להפניית gs.apple.com
- תיעוד ניסיונות חיבור לביקורת

### אבטחת נתונים

- אחסון נתונים רגישים (ECID, nonces) עם הרשאות מתאימות
- שקול הצפנה במנוחה לדרישות תאימות
- מחיקה מאובטחת של כרטיסים שפג תוקפם

### העברת USB

- סריקת כונני USB לתוכנות זדוניות לפני שימוש
- שימוש בכונני USB מוצפנים לסביבות רגישות
- תיעוד שרשרת משמורת

## תפעול יומיומי

### ⚠️ אזהרה קריטית: חלון זמן של 15 דקות

**כרטיסי TSS תקפים רק ל-15 דקות מרגע יצירת ה-ApNonce על ידי ה-Client Mac!**

זה אומר שהתהליך המלא (ייצוא → חתימה → ייבוא) חייב להתבצע תוך 15 דקות.

### תהליך עבודה - תגובה מיידית נדרשת

**כאשר מתקבלת בקשה חדשה** (ניטור בזמן אמת):

1. **מיידי** - ייצוא בקשות ממתינות:
   ```bash
   ./bin/proxy-server export --output /mnt/usb/export
   ```

2. **מיידי** - העברת USB פיזית לרשת חיצונית (ריצה!)

3. **מיידי** - ברשת החיצונית, הרצת סקריפט חתימה:
   ```bash
   ./bin/signer --input /mnt/usb/export \
                --output /mnt/usb/tickets \
                --manifest /path/to/BuildManifest.plist
   ```

4. **מיידי** - העברת USB חזרה לרשת המבודדת (ריצה!)

5. **מיידי** - ייבוא כרטיסים:
   ```bash
   ./bin/proxy-server import --input /mnt/usb/tickets
   ```

**זמן יעד**: כל התהליך תוך **10 דקות** (מרווח בטיחות של 5 דקות)

### אסטרטגיות לעמידה בחלון הזמן

**אופציה 1: ניטור אקטיבי**
- מנהל מערכת במוכנות עם USB
- התראות בזמן אמת על בקשות חדשות
- תהליך ידני מהיר

**אופציה 2: תיאום מראש**
- תיאום חלון זמן עם משתמשים
- מנהל מערכת מוכן מראש
- משתמש מתחיל עדכון בזמן מוסכם

**אופציה 3: אוטומציה חלקית** (עתידי)
- VM מבודד עם גישה מוגבלת לאינטרנט
- העברת קבצים אוטומטית
- עדיין דורש אישור ידני לאבטחה

### תחזוקה שבועית

- הרצת ניקוי מטמון להסרת כרטיסים שפג תוקפם (7 ימים)
- סקירת לוגים לאיתור שגיאות או חריגות
- בדיקת זמינות שטח דיסק
- בדיקת תוקף אישורים
- **סקירת זמני תגובה** - האם עמדנו בחלון 15 הדקות?
- **אימון תרגילי חירום** - תרגול התהליך המהיר

## מדדי ביצועים וניטור

**מדדים קריטיים**:
- **זמן תגובה לבקשה חדשה** (יעד: <10 דקות, מקסימום: 15 דקות)
- שיעור cache hits (יעד: >80% לאחר תקופת התחלה)
- עומק תור בקשות ממתינות
- גיל ממוצע של כרטיסים במטמון
- ניצול שטח דיסק
- שיעור שגיאות לפי סוג

**התראות קריטיות**:
- 🔴 **בקשה חדשה התקבלה** - התראה מיידית (SMS/Email/Slack)
- 🔴 **בקשה בת 10 דקות** - אזהרת זמן אזל
- 🟡 שטח דיסק מתחת ל-10GB
- 🟡 תור ממתינות עולה על 100 בקשות
- 🟡 שיעור שגיאות עולה על סף
- 🟡 תוקף אישור יפוג תוך 30 יום

## מדדי ביצועים

### קיבולת

- בקשות במקביל: 10-50 Client Macs
- זמן תגובה: <100ms ל-cache hits, <500ms ל-cache misses
- גודל מטמון: 1000-5000 כרטיסים (50MB-500MB)
- עומק תור: 10-100 בקשות ממתינות בזמן מחזורי עדכון

### אופטימיזציה

- שימוש ב-SQLite למטא-דאטה של מטמון (חיפושים מהירים)
- אחסון קבצי כרטיסים בנפרד (מניעת נפיחות מסד נתונים)
- אינדקס לפי cache key, ECID, וזמן תפוגה

## פתרון בעיות נפוצות

### Client Mac מציג שגיאת אישור

**פתרון**:
1. ודא ש-Root CA מותקן ב-System Keychain
2. בדוק הגדרות אמון ל-SSL
3. ודא ש-DNS מפנה לשרת הפרוקסי

### בקשה תקועה בתור ממתינות

**פתרון**:
1. בדוק לוגים של Signer Script לשגיאות
2. ודא ש-tsschecker יכול להגיע לשרתי Apple
3. בדוק תקינות פורמט קובץ הבקשה

### Cache miss לכרטיס שיובא

**פתרון**:
1. ודא שחישוב cache key תואם
2. בדוק מוסכמת שמות קבצי כרטיסים
3. סקור לוגי ייבוא לשגיאות

### שרת הפרוקסי לא מגיב

**פתרון**:
1. בדוק סטטוס שירות
2. ודא תוקף אישור TLS
3. בדוק כללי חומת אש
4. סקור לוגי שגיאות

## שיפורים עתידיים אפשריים

1. **התראות בזמן אמת** (קריטי!): SMS/Email/Slack כאשר מתקבלת בקשה חדשה
2. **טיימר חזותי**: ספירה לאחור של 15 דקות בממשק
3. **לוח בקרה Web**: ניטור בזמן אמת של תור ומטמון עם אזהרות זמן
4. **חתימה אוטומטית**: VM מבודד עם גישה מוגבלת לאינטרנט (מפחית זמן תגובה)
5. **זמינות גבוהה**: מספר מופעי Proxy Server עם מטמון משותף
6. **אבטחה משופרת**: אימות TLS הדדי, חתימת בקשות
7. **אנליטיקה**: מעקב אחר זמני תגובה ושיעור הצלחה בחלון 15 דקות

## הערות חשובות

### BuildManifest.plist

קובץ זה **קריטי** לתהליך החתימה. הוא מכיל את ה-hashes הקריפטוגרפיים הנדרשים לחתימה תקפה.

**איך להשיג**:
1. הורד קובץ IPSW של גרסת macOS היעד מאתר Apple
2. חלץ את BuildManifest.plist מתוך ה-IPSW (זהו ארכיון ZIP)
3. שמור את הקובץ במיקום נגיש לסקריפט החתימה

**שימוש**:
```bash
# חילוץ BuildManifest מ-IPSW
unzip -p macOS_13.6.ipsw BuildManifest.plist > BuildManifest.plist

# שימוש עם סקריפט החתימה
./bin/signer --manifest BuildManifest.plist ...
```

### הפרדת אחריות

המערכת מפרידה בין שני תהליכים עיקריים:

1. **חתימה קריפטוגרפית** (Proxy Server, פורט 443): מטפל רק בבקשות TSS
2. **אספקת תוכן** (Reposado, פורט 8088): מטפל בהורדת חבילות עדכון

הפרדה זו מאפשרת:
- שרת פרוקסי קל משקל וממוקד
- ניהול תוכן עצמאי
- אין קונפליקטים בפורטים
- פשטות בתחזוקה

## סיכום

מערכת זו מספקת פתרון מלא לעדכון מחשבי Mac ברשתות מבודדות תוך שמירה על אבטחה ותאימות לפרוטוקולי Apple. התהליך דורש תיאום ידני (העברת USB) אך מאפשר עדכונים מאובטחים ללא חיבור אינטרנט ישיר.

**יתרונות**:
- ✅ תמיכה מלאה ב-Apple Silicon
- ✅ תאימות לפרוטוקולי Apple הרשמיים
- ✅ אבטחה מוגברת (אין חיבור אינטרנט ישיר)
- ✅ שקיפות מלאה לתהליך החתימה
- ✅ ניהול מטמון אוטומטי
- ✅ תיעוד מקיף לביקורת

**אתגרים**:
- 🔴 **קריטי**: חלון זמן של 15 דקות בלבד לכל בקשה חדשה!
- ⚠️ דורש תגובה מיידית ומנהל מערכת במוכנות
- ⚠️ דורש BuildManifest לכל גרסת macOS
- ⚠️ דורש הגדרת אמון TLS על כל Client Mac
- ⚠️ דורש תחזוקת שרת Reposado נפרד
- ⚠️ לא מתאים לעדכונים ספונטניים - דורש תיאום מראש

---


graph TD
    subgraph "רשת מבודדת (Air-Gapped Network)"
        style internalNet fill:#e1f5fe,stroke:#01579b,stroke-width:2px
        internalNet["רשת פנימית - ללא גישה לאינטרנט"]

        clientMac["Client Mac (Apple Silicon)
        IP: 192.168.1.x
        OS: macOS 12+"]

        subgraph "שרת פנימי (Internal Server) - 192.168.1.10"
            style internalServer fill:#b3e5fc,stroke:#0288d1,stroke-width:2px
            proxyServer["Proxy Server
            (Python/Flask)
            Port: 443"]
            reposadoServer["Reposado Server
            (תוכן העדכון)
            Port: 8088"]
            
            subgraph "אחסון פרוקסי"
                style proxyStorage fill:#81d4fa,stroke:#039be5,stroke-width:1px
                ticketCache["Ticket Cache
                (כרטיסים חתומים)"]
                pendingQueue["Pending Queue
                (בקשות ממתינות)"]
            end
        end

        clientMac -- "בקשת TSS (HTTPS)" --> proxyServer
        proxyServer -- "תגובת TSS (כרטיס)" --> clientMac
        clientMac -- "הורדת תוכן (HTTP)" --> reposadoServer
        proxyServer -- "בדיקה/שמירה" --> ticketCache
        proxyServer -- "שמירה" --> pendingQueue
    end

    subgraph "רשת חיצונית (Internet Network)"
        style externalNet fill:#ffe0b2,stroke:#e65100,stroke-width:2px
        externalNet["רשת מחוברת לאינטרנט"]

        subgraph "מערכת חתימה חיצונית (External Signer System)"
            style externalSigner fill:#ffcc80,stroke:#fb8c00,stroke-width:2px
            signerScript["Signer Script
            (Python/tsschecker)"]
            buildManifest["BuildManifest.plist
            (מתוך IPSW)"]
        end

        appleTSS["Apple TSS
        (gs.apple.com)"]

        signerScript -- "בקשת חתימה" --> appleTSS
        appleTSS -- "כרטיס חתום" --> signerScript
        buildManifest -- "שימוש ב-Hashes" --> signerScript
    end

    subgraph "העברה פיזית (Sneakernet)"
        style usbTransfer fill:#e0e0e0,stroke:#616161,stroke-width:2px,stroke-dasharray: 5 5
        usbDrive["כונן USB
        (העברה ידנית)"]
    end

    pendingQueue -.-> usbDrive
    usbDrive -.-> signerScript
    signerScript -.-> usbDrive
    usbDrive -.-> ticketCache

    classDef default fill:#f9f9f9,stroke:#333,stroke-width:2px;
    classDef network fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef server fill:#b3e5fc,stroke:#0288d1,stroke-width:2px;
    classDef storage fill:#81d4fa,stroke:#039be5,stroke-width:1px;
    classDef external fill:#ffe0b2,stroke:#e65100,stroke-width:2px;
    classDef signer fill:#ffcc80,stroke:#fb8c00,stroke-width:2px;
    classDef usb fill:#e0e0e0,stroke:#616161,stroke-width:2px,stroke-dasharray: 5 5;

    class internalNet,clientMac network;
    class internalServer,proxyServer,reposadoServer server;
    class proxyStorage,ticketCache,pendingQueue storage;
    class externalNet,appleTSS external;
    class externalSigner,signerScript,buildManifest signer;
    class usbTransfer,usbDrive usb;
