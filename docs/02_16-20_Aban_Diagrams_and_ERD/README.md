<div dir="rtl">

# فاز دوم (۱۶ تا ۲۰ آبان): چهار نمای اصلی طراحی و معماری (Four Design Views)

**نام پروژه:** سیم‌فست (SimFast) — سامانه خرید شماره مجازی و دریافت بلادرنگ کد فعال‌سازی  
**درس:** تحلیل و طراحی سیستم‌های نرم‌افزاری (SSAD) — پاییز ۱۴۰۵  

---

## ۱. دیاگرام مورد کاربرد (UML Use Case Diagram)

این دیاگرام تعامل کاربران و سیستم‌های بیرونی را با هسته سامانه سیم‌فست در مرز MVP نشان می‌دهد.

```mermaid
flowchart LR
    User(["👤 کاربر واردشده (Signed-in User)"])
    Guest(["👤 کاربر مهمان (Guest)"])
    Gateway(["🏦 درگاه پرداخت (شتاب / کریپتو)"])
    Provider(["☁️ وب‌سرویس تامین‌کننده (5sim / Mock)"])

    subgraph SimFast_MVP ["مرز سامانه سیم‌فست (SimFast MVP Boundary)"]
        UC1("ثبت‌نام و ورود به حساب")
        UC2("مشاهده کاتالوگ سرویس‌ها و قیمت‌ها")
        UC3("شارژ کیف پول (ریالی و کریپتو)")
        UC4("خرید شماره و بلوکه شدن وجه (HOLD)")
        UC5("دریافت کد OTP و کسر قطعی وجه (COMMIT)")
        UC6("لغو سفارش / انقضا و عودت وجه (REFUND)")
        UC7("مشاهده تاریخچه سفارش‌ها و تراکنش‌ها")
    end

    Guest --> UC1
    Guest --> UC2
    User --> UC2
    User --> UC3
    User --> UC4
    User --> UC5
    User --> UC6
    User --> UC7

    UC3 <--> Gateway
    UC4 <--> Provider
    UC5 <--> Provider
    UC6 <--> Provider



    flowchart TD
    Start(["شروع: انتخاب سرویس و کشور توسط کاربر و فشردن دکمه خرید"]) --> CheckAuth{"آیا کاربر وارد شده است؟"}
    
    CheckAuth -- "خیر" --> ErrAuth["خطای 401: عدم احراز هویت و نمایش صفحه ورود"]
    ErrAuth --> EndFail(["پایان ناموفق"])

    CheckAuth -- "بله" --> CheckBal{"آیا موجودی آزاد کاربر کافی است؟"}
    CheckBal -- "خیر (مسیر خطا)" --> ErrBal["خطای 400: عدم کفایت موجودی کیف پول"]
    ErrBal --> EndFail

    CheckBal -- "بله" --> HoldFunds["انتقال اتمیک وجه از موجودی آزاد به بلوکه‌شده (ثبت تراکنش HOLD)"]
    HoldFunds --> ReqNum["ارسال درخواست دریافت شماره به آداپتور تامین‌کننده"]
    
    ReqNum --> CheckProv{"آیا تامین‌کننده شماره ارائه داد؟"}
    CheckProv -- "خیر (خطای سرور تامین‌کننده)" --> RollbackHold["بازگشت فوری مبلغ به موجودی آزاد (ثبت تراکنش REFUND)"]
    RollbackHold --> ErrProv["نمایش خطای موقت عدم موجودی خط در سرور"]
    ErrProv --> EndFail

    CheckProv -- "بله" --> SaveOrder["ثبت سفارش با وضعیت WAITING_FOR_SMS و آغاز مهلت ۱۵ دقیقه‌ای"]
    SaveOrder --> PollLoop{"بررسی وضعیت: دریافت کد پیامک یا اقدام به لغو/انقضا"}

    PollLoop -- "کد پیامک دریافت شد (مسیر موفقیت)" --> CommitFunds["کسر قطعی وجه از موجودی بلوکه‌شده (ثبت تراکنش COMMIT)"]
    CommitFunds --> ShowOTP["تغییر وضعیت به RECEIVED و نمایش کد OTP به کاربر"]
    ShowOTP --> EndSuccess(["پایان موفقیت‌آمیز"])

    PollLoop -- "لغو توسط کاربر یا اتمام ۱۵ دقیقه (مسیر جایگزین)" --> RefundFunds["بازگرداندن کامل مبلغ به موجودی آزاد (ثبت تراکنش REFUND)"]
    RefundFunds --> CancelOrder["تغییر وضعیت سفارش به CANCELED یا TIMEOUT"]
    CancelOrder --> EndCancel(["پایان با عودت وجه"])



    flowchart LR
    subgraph Client ["لایه رابط کاربری (Frontend)"]
        UI["قالب‌های Jinja2 + TailwindCSS + کامپوننت‌های تعاملی"]
        APIClient["ماژول تبادل داده جاوااسکریپت (Fetch Client)"]
        UI --> APIClient
    end

    subgraph Server ["لایه سرویس‌دهنده و منطق تجاری (FastAPI Modular Monolith)"]
        Router["مسیریابی اندپوینت‌ها و اعتبارسنجی ورودی (Pydantic Schemas)"]
        Auth["لایه کنترل دسترسی، بررسی مالکیت و مدیریت نشست (Auth Guards)"]
        Services["منطق پردازش کیف پول، ماشین وضعیت سفارش و زمان‌بندی"]
        Repo["لایه دسترسی به داده‌ها (SQLAlchemy ORM Repositories)"]
        Adapter["الگوی Adapter تامین‌کنندگان (FiveSimAdapter / MockAdapter)"]

        Router --> Auth
        Auth --> Services
        Services --> Repo
        Services --> Adapter
    end

    subgraph Storage ["پایگاه داده پایدار (Relational Database)"]
        DB[("PostgreSQL / SQLite\nجداول: Users, Wallets, Orders, Transactions, Catalog")]
    end

    subgraph External ["سیستم‌های بیرونی (External Providers)"]
        ExtSMS["وب‌سرویس تامین شماره (5sim.net API)"]
        ExtPay["درگاه‌های پرداخت (شتاب شاپرک / درگاه کریپتو)"]
    end

    APIClient -- "درخواست‌های HTTP (GET/POST)" --> Router
    Router -- "پاسخ JSON و کدهای وضعیت" --> APIClient
    Repo -- "تراکنش‌های پایگاه داده (ACID Transactions)" --> DB
    Adapter -- "فراخوانی REST API تحت پروتکل امن" --> ExtSMS
    Services -- "تاییدیه و کال‌بک پرداخت" --> ExtPay



    erDiagram
    USER ||--|| WALLET : "مالکیت دارد (1 به 1)"
    USER ||--o{ ORDER : "ثبت می‌کند (1 به 0..*)"
    SERVICE_CATALOG ||--o{ ORDER : "تعیین تعرفه (1 به 0..*)"
    WALLET ||--o{ TRANSACTION : "شامل تراکنش (1 به 0..*)"
    ORDER |o--o{ TRANSACTION : "ایجاد می‌کند (0..1 به 0..*)"

    USER {
        int id PK
        string phone_number UK "شماره موبایل یکتا"
        string password_hash "رمز عبور هش‌شده"
        string role "user یا admin"
        datetime created_at "زمان ثبت‌نام"
    }

    WALLET {
        int id PK
        int user_id FK "کلید خارجی یکتا به کاربر"
        bigint available_balance "موجودی آزاد قابل مصرف (شرط >= 0)"
        bigint locked_balance "موجودی قفل‌شده در انتظار کد (شرط >= 0)"
        datetime updated_at "زمان آخرین تراکنش"
    }

    SERVICE_CATALOG {
        int id PK
        string service_code "شناسه سرویس مانند telegram یا openai"
        string country_code "کد کشور مانند US یا GB"
        string operator_name "نام اپراتور مخابراتی"
        bigint price_toman "تعرفه به تومان"
        int stock_count "تعداد خطوط آماده تحویل"
        boolean is_active "وضعیت فعال بودن سرویس"
    }

    ORDER {
        int id PK
        int user_id FK "شناسه کاربر مالک سفارش"
        int catalog_id FK "ارجاع به تعرفه سرویس"
        string external_provider_id "شناسه سفارش در سرور تامین‌کننده"
        string virtual_number "شماره تلفن مجازی تحویل‌شده"
        string otp_code "کد پیامک دریافتی (قابل خالی بودن)"
        bigint price_toman "مبلغ قفل‌شده برای سفارش"
        string status "وضعیت: WAITING_FOR_SMS, RECEIVED, CANCELED, TIMEOUT"
        datetime expires_at "مهلت انقضای ۱۵ دقیقه‌ای"
        datetime created_at "زمان ثبت سفارش"
    }

    TRANSACTION {
        int id PK
        int wallet_id FK "کیف پول مقصد/مبدا"
        int order_id FK "شناسه سفارش مرتبط (اختیاری در شارژ)"
        bigint amount "مبلغ تراکنش به تومان"
        string tx_type "نوع: DEPOSIT, HOLD, COMMIT, REFUND"
        datetime created_at "زمان ثبت تراکنش"
    }
