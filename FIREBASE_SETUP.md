# دليل إعداد Firebase (بدون كارت ائتمان) — خطوة بخطوة

Firebase خدمة من Google بتوفر قاعدة بيانات سحابية مجانية (باقة Spark) بدون أي
بيانات دفع في الباقة الأساسية. هنستخدمها لتخزين تسجيلات الحضور والانصراف.

---

## الخطوة 1: إنشاء مشروع Firebase

1. روح على: https://console.firebase.google.com
2. سجّل دخول بحساب Gmail بتاعك
3. اضغط **"Add project"** أو **"إنشاء مشروع"**
4. اكتب اسم زي `attendance-terra` (أي اسم تحبه)
5. لو سأل عن Google Analytics — اضغط **"إلغاء التفعيل" (Disable)** مش محتاجينه، هيسهل الإعداد
6. اضغط **Create project** واستنى شوية لحد ما يخلص

✅ لن يُطلب منك أي بيانات دفع في هذه الخطوات.

---

## الخطوة 2: تفعيل قاعدة البيانات (Firestore)

1. من القائمة الجانبية، تحت **Build**، اضغط **Firestore Database**
2. اضغط **Create database**
3. اختار **"Start in test mode"** (وضع الاختبار — هيسهّل علينا الإعداد الأول، هنظبط الحماية بعدين)
4. اختار أقرب منطقة (زي `eur3` أو `europe-west` لو موجودة) واضغط **Enable**

---

## الخطوة 3: تسجيل تطبيق الويب

1. من الصفحة الرئيسية للمشروع (اضغط أيقونة الترس ⚙️ ← **Project settings**)
2. انزل لقسم **"Your apps"**
3. اضغط أيقونة **</>** (Web)
4. اكتب اسم زي `attendance-web` واضغط **Register app**
5. هتظهرلك شاشة فيها كود زي ده:

```javascript
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "attendance-terra.firebaseapp.com",
  projectId: "attendance-terra",
  storageBucket: "attendance-terra.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef123456"
};
```

**انسخ القيم دي كاملة** — هتحتاجها في الخطوة الجاية.

---

## الخطوة 4: وضع الإعدادات في ملفات التطبيق

1. افتح ملف `index.html` (من المشروع اللي هبعتلك)
2. دور على السطر ده تقريبًا في بداية الملف:

```javascript
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  ...
};
```

3. استبدل القيم دي بالقيم الحقيقية اللي نسختها من الخطوة 3
4. **كرر نفس الخطوة في ملف `report.html`** (فيه نفس السطر بالظبط)

---

## الخطوة 5: تأمين قاعدة البيانات (مهم بعد التأكد إن كل حاجة شغالة)

وضع "test mode" اللي اخترناه بيسمح لأي حد يقرا/يكتب في القاعدة لمدة 30 يوم بس،
وبعدين بيتقفل تلقائيًا. عشان تأمّنها بشكل دائم ومناسب لاستخدامنا:

1. في Firestore Database، روح لتبويب **Rules**
2. امسح اللي موجود واستبدله بده:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /attendance_events/{eventId} {
      allow read: if true;
      allow create: if request.resource.data.employeeName is string
                    && request.resource.data.type in ["check_in", "check_out", "leave", "permission"];
      allow update, delete: if false;
    }
    match /app_config/employees {
      allow read: if true;
      allow write: if true;
    }
    match /employee_auth/{employeeName} {
      allow read: if true;
      allow create: if request.resource.data.passwordHash is string
                    && !exists(/databases/$(database)/documents/employee_auth/$(employeeName));
      allow delete: if true;
      allow update: if false;
    }
  }
}
```

القواعد دي بتسمح لأي حد يضيف تسجيل حضور/انصراف/إجازة/استئذان جديد (زي ما إحنا
عايزين، بدون تسجيل دخول معقد)، لكن بتمنع أي حد يعدّل أو يمسح تسجيلات قديمة، وبتتأكد
إن البيانات المُرسلة شكلها صحيح. الجزء الخاص بـ `app_config/employees` بيسمح لصفحة
`admin.html` (المحمية بكلمة سر منفصلة داخل الصفحة نفسها) بتحديث قائمة أسماء الموظفين.

الجزء الخاص بـ `employee_auth` بيخزن باسورد كل موظف (مشفّر، مش نص واضح). القاعدة
`!exists(...)` بتمنع أي حد يغيّر باسورد موظف عن طريق إعادة "إنشائه" وهو موجود بالفعل —
الباسورد بيتحدد مرة واحدة بس أول ما الموظف يسجل دخول، وبعد كده محتاج مسح (Reset) من
صفحة `admin.html` قبل ما يتغيّر.

> ⚠️ **ملاحظة أمان:** الباسورد مش مشفّر بطريقة قوية جدًا (SHA-256 بسيط بدون "ملح" -
> salt)، وده مناسب لمنع التلاعب العرضي بين الزملاء، لكنه مش بديل عن نظام مصادقة حقيقي.
> لو الأمان مهم جدًا بالنسبالك، فكّر لاحقًا في استخدام Firebase Authentication بدل
> الطريقة دي.

> ⚠️ **لو كنت نشرت نسخة قديمة من القواعد قبل كده، ارجع لتبويب Rules واستبدلها
> بالنص الكامل أعلاه، وبعدين اضغط Publish تاني.** النسخة القديمة بتسمح بس بـ
> `check_in`/`check_out` وهتمنع تسجيل الإجازة والاستئذان.

3. اضغط **Publish**

---

## حدود الباقة المجانية (Spark)

| المورد | الحد المجاني | استخدامنا المتوقع |
|---|---|---|
| القراءة (Reads) | 50,000 يوميًا | أقل من 1% من الحد حتى مع عشرات الموظفين |
| الكتابة (Writes) | 20,000 يوميًا | تسجيلين بس (حضور/انصراف) لكل موظف يوميًا |
| التخزين | 1 GB | بيانات نصية بسيطة، هتاخد سنين لتقترب من الحد |

خلاصة: مستحيل تقترب من أي حد من الحدود دي بعدد موظفين عادي، فمش هتتحاسب خالص.
