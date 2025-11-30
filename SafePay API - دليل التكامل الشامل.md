# 📘 SafePay API - دليل التكامل الشامل

<div dir="rtl">

## 📋 جدول المحتويات

1. [نظرة عامة](#نظرة-عامة)
2. [الهيكل العام](#الهيكل-العام)
3. [المصادقة والأمان](#المصادقة-والأمان)
4. [Direct Payment API](#direct-payment-api)
5. [Escrow Management API](#escrow-management-api)
6. [Payment Management API](#payment-management-api)
7. [Settlement API](#settlement-api)
8. [Escrow Configuration API](#escrow-configuration-api)
9. [Reports & Analytics API](#reports--analytics-api)
10. [Webhooks API](#webhooks-api)
11. [Disputes API](#disputes-api)
12. [Support Tickets API](#support-tickets-api)
13. [User Management API](#user-management-api)
14. [معالجة الأخطاء](#معالجة-الأخطاء)
15. [نماذج الكود](#نماذج-الكود)
16. [أفضل الممارسات](#أفضل-الممارسات)

---

## 🎯 نظرة عامة

### ما هو SafePay API؟

SafePay API هو نظام دفع وضمان إلكتروني متكامل يوفر واجهة برمجة تطبيقات (RESTful API) شاملة للشركاء والمنصات التجارية. يتيح SafePay للمطورين دمج خدمات الدفع والضمان الإلكتروني في تطبيقاتهم بسهولة.

### مخطط معماري النظام

```mermaid
graph TB
    subgraph "Client Layer"
        A[تطبيق الشريك]
        B[منصة التجارة الإلكترونية]
    end
    
    subgraph "API Gateway"
        C[SafePay API]
        D[Authentication]
        E[Authorization]
    end
    
    subgraph "Business Logic"
        F[Payment Service]
        G[Escrow Service]
        H[Settlement Service]
        I[Report Service]
    end
    
    subgraph "Data Layer"
        J[(قاعدة البيانات)]
        K[Accounting System]
        L[Notification System]
    end
    
    A --> C
    B --> C
    C --> D
    D --> E
    E --> F
    E --> G
    E --> H
    E --> I
    F --> J
    G --> J
    H --> J
    I --> J
    H --> K
    F --> L
    G --> L
    H --> L
    
    style C fill:#2196F3,color:#fff
    style K fill:#4CAF50,color:#fff
    style L fill:#FF9800,color:#fff
```

### مخطط تدفق المصادقة

```mermaid
sequenceDiagram
    participant Client as 👤 العميل
    participant API as 🏦 SafePay API
    participant Auth as 🔐 نظام المصادقة
    participant DB as 🗄️ قاعدة البيانات

    Client->>API: Request with Headers<br/>X-API-Key + X-API-Secret
    API->>Auth: التحقق من API Key و Secret
    Auth->>DB: البحث عن API Key
    
    alt ❌ API Key غير موجود أو غير صالح
        Auth-->>API: 401 Unauthorized
        API-->>Client: رسالة خطأ
    else ✅ API Key صالح
        alt ✅ تم إرسال X-API-Secret
            Auth->>Auth: فك تشفير api_secret
            Auth->>Auth: مقارنة مع X-API-Secret
            
            alt ❌ API Secret غير متطابق
                Auth-->>API: 401 Unauthorized
                API-->>Client: رسالة خطأ
            else ✅ API Secret متطابق
                Auth->>DB: جلب صلاحيات API Key
                Auth->>Auth: التحقق من الصلاحيات المطلوبة
                
                alt ❌ لا توجد صلاحيات كافية
                    Auth-->>API: 403 Forbidden
                    API-->>Client: رسالة خطأ
                else ✅ صلاحيات كافية
                    Auth->>DB: التحقق من IP Whitelist
                    Auth->>DB: التحقق من Rate Limit
                    Auth->>DB: تحديث last_used_at
                    Auth->>API: Partner ID + Permissions
                    API->>API: معالجة الطلب
                    API-->>Client: Response
                end
            end
        else ✅ لم يتم إرسال X-API-Secret
            Auth->>DB: جلب صلاحيات API Key
            Auth->>Auth: التحقق من الصلاحيات المطلوبة
            
            alt ❌ لا توجد صلاحيات كافية
                Auth-->>API: 403 Forbidden
                API-->>Client: رسالة خطأ
            else ✅ صلاحيات كافية
                Auth->>DB: تحديث last_used_at
                Auth->>API: Partner ID + Permissions
                API->>API: معالجة الطلب
                API-->>Client: Response
            end
        end
    end
```

### المميزات الرئيسية

- ✅ **الدفع المباشر** (Direct Payment) - معالجة المدفوعات بدون ضمان
- ✅ **خدمة الضمان** (Escrow Service) - حماية المشتري والبائع في المعاملات
- ✅ **طرق دفع متعددة** - دعم Mada، Credit Card، Bank Transfer، Wallet
- ✅ **Webhooks** - إشعارات فورية للأحداث المهمة
- ✅ **Double-Entry Accounting** - نظام محاسبي مزدوج القيد
- ✅ **تقارير وتحليلات** - إحصائيات شاملة ومفصلة
- ✅ **إدارة النزاعات** - نظام متكامل لحل النزاعات
- ✅ **تسويات تلقائية** - إدارة طلبات التسوية

### حالات الاستخدام

- **منصات التجارة الإلكترونية** - دمج SafePay في متاجر إلكترونية
- **أسواق B2B/B2C** - معاملات بين الشركات والأفراد
- **خدمات الاستعانة** - دفع مقابل الخدمات
- **بيع المنتجات المستعملة** - ضمان المعاملات
- **خدمات التوصيل** - ضمان تسليم الطلبات

---

## 🏗️ الهيكل العام

### Base URL

```
Production:  https://api.safepay.com
Sandbox:     https://sandbox-api.safepay.com
Local:       http://localhost:8000
```

### Headers المطلوبة

جميع طلبات API تتطلب Headers التالية:

```http
Authorization: Bearer {api_key}
Content-Type: application/json
Accept: application/json
```

**أو باستخدام X-API-Key:**

```http
X-API-Key: {api_key}
X-API-Secret: {api_secret}  (اختياري)
Content-Type: application/json
```

### Naming Conventions

#### URLs
- استخدام **kebab-case** للأسماء المركبة: `/api/partner/escrows/create-with-payment`
- استخدام **lowercase** لجميع المسارات
- استخدام **plural** للأسماء: `/api/partner/payments` وليس `/api/partner/payment`

#### Request/Response
- استخدام **snake_case** للأسماء في JSON: `payment_id`, `escrow_number`
- استخدام **camelCase** في JavaScript/TypeScript عند التكامل

#### Status Codes
- استخدام **أرقام HTTP** القياسية: `200`, `201`, `400`, `401`, `404`, `422`, `500`

---

## 🔐 المصادقة والأمان

### Authentication Methods

جميع طلبات API (بما في ذلك تسجيل الدخول) تتطلب إرسال API Key و API Secret في **Headers** وليس في Body.

#### 1. Bearer Token (مُوصى به)

```http
Authorization: Bearer {api_key}
X-API-Secret: {api_secret}  (اختياري - يُوصى به للأمان)
```

**مثال:**
```http
Authorization: Bearer spk_live_1234567890abcdefghijklmnopqrstuvwxyz
X-API-Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

#### 2. X-API-Key Header

```http
X-API-Key: {api_key}
X-API-Secret: {api_secret}  (اختياري - يُوصى به للأمان)
```

**مثال:**
```http
X-API-Key: spk_live_1234567890abcdefghijklmnopqrstuvwxyz
X-API-Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### ⚠️ ملاحظات مهمة:

1. **API Key و API Secret يُرسلان في Headers فقط** - لا يتم إرسالهما في Request Body
2. **API Secret اختياري** - لكن يُوصى بشدة بإرساله لزيادة الأمان
3. **التحقق من API Secret** - إذا تم إرسال `X-API-Secret`، سيتم التحقق منه مقابل القيمة المحفوظة في قاعدة البيانات
4. **تشفير البيانات** - API Key و API Secret يتم تشفيرهما في قاعدة البيانات باستخدام Laravel Crypt

### الحصول على API Key

1. تسجيل الدخول إلى لوحة تحكم الشريك
2. الانتقال إلى **API Keys** → **Create New Key**
3. نسخ المفتاح وحفظه بشكل آمن
4. **⚠️ تحذير:** لا تشارك API Key مع أي شخص

### API Key Types

| النوع | الوصف | الاستخدام |
|------|------|----------|
| `spk_live_*` | Production Key | للاستخدام في الإنتاج |
| `spk_test_*` | Test Key | للاختبار والتطوير |
| `spk_sandbox_*` | Sandbox Key | للاختبار في بيئة Sandbox |

### Security Best Practices

1. **استخدام HTTPS فقط** - جميع الطلبات يجب أن تتم عبر HTTPS
2. **عدم تخزين API Key في الكود** - استخدام Environment Variables
3. **تحديد IP Whitelist** - تقييد الوصول من IPs محددة
4. **Rate Limiting** - احترام حدود الطلبات (100 طلب/دقيقة)
5. **تحديث المفاتيح بانتظام** - تغيير API Keys بشكل دوري

### 🔐 آلية التحقق من API Key و API Secret

عند إرسال طلب API، يتم التحقق من API Key و API Secret على النحو التالي:

#### 1. قراءة Headers
```php
$apiKey = $request->header('X-API-Key');
$apiSecret = $request->header('X-API-Secret');
$bearerToken = $request->bearerToken(); // للـ Authorization: Bearer
```

#### 2. التحقق من API Key
- البحث عن API Key في قاعدة البيانات (جميع المفاتيح النشطة)
- فك تشفير كل API Key ومقارنته مع القيمة المرسلة
- التحقق من حالة المفتاح (active/inactive)
- التحقق من تاريخ الانتهاء (expires_at)

#### 3. التحقق من API Secret (إذا تم إرساله)
- فك تشفير `api_secret` المحفوظ في قاعدة البيانات
- مقارنة القيمة المرسلة مع القيمة المحفوظة
- إذا لم تتطابق القيم، يتم رفض الطلب

#### 4. التحقق من الصلاحيات
- جلب صلاحيات API Key من قاعدة البيانات
- التحقق من الصلاحية المطلوبة للـ endpoint (read/write/delete)
- إذا لم تكن الصلاحيات كافية، يتم رفض الطلب

#### 5. التحقق من IP Whitelist (إن وجد)
- إذا كان API Key يحتوي على `whitelisted_ips`
- التحقق من أن IP المرسل موجود في القائمة
- إذا لم يكن موجوداً، يتم رفض الطلب

### 📋 مثال على التحقق (من الكود)

```php
// في SecureApiGateway Middleware
protected function performAuthentication(Request $request): array
{
    $apiKey = $request->header('X-API-Key');
    $apiSecret = $request->header('X-API-Secret');
    $bearerToken = $request->bearerToken();

    // التحقق من API Key
    if ($apiKey) {
        $apiKeyModel = $this->apiKeyService->validateApiKey($apiKey, $apiSecret);
        if ($apiKeyModel) {
            return [
                'success' => true,
                'user' => $apiKeyModel,
                'type' => 'api_key'
            ];
        }
    }

    // التحقق من Bearer Token
    if ($bearerToken) {
        $user = $this->validateBearerToken($bearerToken);
        if ($user) {
            return [
                'success' => true,
                'user' => $user,
                'type' => 'bearer_token'
            ];
        }
    }

    return ['success' => false, 'error' => 'Authentication required', 'code' => 401];
}
```

### 🔄 مخطط تدفق التحقق من API Key و Secret

```mermaid
sequenceDiagram
    participant Client as 👤 العميل
    participant Middleware as 🔐 SecureApiGateway
    participant Service as 🔑 ApiKeyService
    participant DB as 🗄️ قاعدة البيانات

    Client->>Middleware: Request with Headers<br/>X-API-Key + X-API-Secret
    Middleware->>Service: validateApiKey(apiKey, apiSecret)
    
    Service->>DB: جلب جميع API Keys النشطة
    DB-->>Service: قائمة API Keys
    
    loop لكل API Key
        Service->>Service: فك تشفير api_key
        Service->>Service: مقارنة مع القيمة المرسلة
        
        alt ✅ API Key متطابق
            alt ✅ تم إرسال API Secret
                Service->>Service: فك تشفير api_secret
                Service->>Service: مقارنة مع القيمة المرسلة
                
                alt ✅ API Secret متطابق
                    Service->>DB: التحقق من الصلاحيات
                    Service->>DB: التحقق من IP Whitelist
                    Service->>DB: التحقق من Rate Limit
                    Service-->>Middleware: ✅ API Key صالح
                    Middleware-->>Client: ✅ Request معالج
                else ❌ API Secret غير متطابق
                    Service-->>Middleware: ❌ 401 Unauthorized
                    Middleware-->>Client: رسالة خطأ
                end
            else ✅ لم يتم إرسال API Secret
                Service->>DB: التحقق من الصلاحيات
                Service-->>Middleware: ✅ API Key صالح (بدون Secret)
                Middleware-->>Client: ✅ Request معالج
            end
        else ❌ API Key غير متطابق
            Service->>Service: تجربة API Key التالي
        end
    end
    
    alt ❌ لم يتم العثور على API Key
        Service-->>Middleware: ❌ 401 Unauthorized
        Middleware-->>Client: رسالة خطأ
    end
```

### 🔑 تسجيل الدخول للـ Partner API

#### ⚠️ **مهم جداً:** هناك نوعان من تسجيل الدخول

**النوع الأول: تسجيل الدخول البرمجي (API-to-API)**
- يستخدم **API Key و API Secret في Headers**
- مناسب للمصادقة البرمجية بين الأنظمة
- **لا يتطلب email/password في Body**

**النوع الثاني: تسجيل الدخول للمستخدمين (User-to-API)**
- يستخدم **email/username و password في Body**
- مناسب للمستخدمين الذين يريدون الدخول عبر API
- **يتطلب email/password في Body**

#### 📋 تفاصيل تسجيل الدخول البرمجي (API Key/Secret)

#### 📋 مثال على طلب تسجيل الدخول

**Endpoint:** `POST /api/partner/auth/login`

**Headers المطلوبة:**
```http
X-API-Key: spk_live_1234567890abcdefghijklmnopqrstuvwxyz
X-API-Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Content-Type: application/json
Accept: application/json
```

**أو باستخدام Bearer Token:**
```http
Authorization: Bearer spk_live_1234567890abcdefghijklmnopqrstuvwxyz
X-API-Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Content-Type: application/json
Accept: application/json
```

**Request Body (للتسجيل البرمجي - API Key/Secret):**

Body **اختياري تماماً** ويمكن أن يكون:
- **فارغاً** `{}`
- **غير موجود** (لا ترسله)
- **يحتوي على metadata فقط** (بيانات إضافية)

#### مثال 1: Body فارغ (مُوصى به)
```json
{}
```

#### مثال 2: بدون Body (مُوصى به)
لا ترسل body على الإطلاق

#### مثال 3: Body مع metadata (اختياري)
```json
{
  "metadata": {
    "device_id": "device-123",
    "app_version": "1.0.0",
    "platform": "web",
    "user_agent": "Mozilla/5.0...",
    "ip_address": "192.168.1.1"
  }
}
```

**⚠️ تحذيرات مهمة:**

1. **❌ لا ترسل API Key أو API Secret في Body**
   ```json
   // ❌ خطأ - لا تفعل هذا
   {
     "api_key": "spk_live_...",
     "api_secret": "..."
   }
   ```

2. **✅ API Key و Secret في Headers فقط**
   ```http
   X-API-Key: spk_live_...
   X-API-Secret: ...
   ```

3. **Body اختياري تماماً** - المصادقة تتم عبر Headers فقط

4. **إذا أرسلت body**، تأكد من:
   - `Content-Type: application/json` في Headers
   - Body بصيغة JSON صحيحة

---

#### 📋 تفاصيل تسجيل الدخول للمستخدمين (Email/Password)

**Endpoint:** `POST /api/partner/auth/login`

**Headers المطلوبة:**
```http
Content-Type: application/json
Accept: application/json
```

**⚠️ ملاحظة:** لا حاجة لـ API Key/Secret في Headers عند استخدام email/password

**Request Body (مطلوب):**
```json
{
  "email": "partner@example.com",
  "password": "your_password"
}
```

**أو باستخدام username:**
```json
{
  "username": "partner_username",
  "password": "your_password"
}
```

**Response (Success - 200 OK):**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {
      "id": 1,
      "email": "partner@example.com",
      "username": "partner_username",
      "is_partner": 1,
      "status": "active"
    }
  }
}
```

**Response (Error - 401 Unauthorized):**
```json
{
  "success": false,
  "message": "Invalid credentials",
  "errors": {
    "email": ["The provided credentials are incorrect."]
  }
}
```

**⚠️ ملاحظات مهمة:**

1. **✅ Email/Password في Body** - هذا منطقي ومطلوب لتسجيل دخول المستخدمين
2. **✅ API Key/Secret في Headers** - هذا منطقي ومطلوب للمصادقة البرمجية
3. **يمكن استخدام أي من الطريقتين** حسب نوع التطبيق:
   - **تطبيق برمجي (API-to-API)**: استخدم API Key/Secret
   - **تطبيق للمستخدمين (User-to-API)**: استخدم Email/Password

**Response (Success - 200 OK):**
```json
{
  "success": true,
  "message": "Authentication successful",
  "data": {
    "partner": {
      "id": 1,
      "company_name": "Noon",
      "status": "active"
    },
    "api_key": {
      "id": 123,
      "name": "Production API Key",
      "permissions": ["read", "write"],
      "last_used_at": "2024-01-15T10:30:00Z"
    }
  }
}
```

#### 🔄 مخطط تدفق تسجيل الدخول

```mermaid
sequenceDiagram
    participant Client as 👤 العميل
    participant Middleware as 🔐 SecureApiGateway
    participant Service as 🔑 ApiKeyService
    participant DB as 🗄️ قاعدة البيانات
    participant Controller as 🎮 AuthController

    Client->>Middleware: POST /api/partner/auth/login<br/>Headers: X-API-Key + X-API-Secret
    
    Note over Middleware: 1. التحقق من الأمان الأساسي
    Middleware->>Middleware: performSecurityChecks()
    
    Note over Middleware: 2. التحقق من المصادقة
    Middleware->>Service: validateApiKey(apiKey, apiSecret)
    
    Service->>DB: جلب جميع API Keys النشطة
    DB-->>Service: قائمة API Keys
    
    loop لكل API Key
        Service->>Service: فك تشفير api_key
        Service->>Service: مقارنة مع X-API-Key
        
        alt ✅ API Key متطابق
            alt ✅ تم إرسال X-API-Secret
                Service->>Service: فك تشفير api_secret
                Service->>Service: مقارنة مع X-API-Secret
                
                alt ✅ API Secret متطابق
                    Service->>DB: التحقق من الصلاحيات
                    Service->>DB: التحقق من IP Whitelist
                    Service->>DB: التحقق من Rate Limit
                    Service->>DB: تحديث last_used_at
                    Service-->>Middleware: ✅ API Key صالح
                    
                    Middleware->>Controller: Request مع API Key في attributes
                    Controller->>DB: جلب بيانات Partner
                    DB-->>Controller: بيانات Partner
                    Controller-->>Client: ✅ 200 OK + بيانات Partner
                else ❌ API Secret غير متطابق
                    Service-->>Middleware: ❌ 401 Unauthorized
                    Middleware-->>Client: رسالة خطأ
                end
            else ✅ لم يتم إرسال X-API-Secret
                Service->>DB: التحقق من الصلاحيات
                Service->>DB: تحديث last_used_at
                Service-->>Middleware: ✅ API Key صالح (بدون Secret)
                
                Middleware->>Controller: Request مع API Key
                Controller-->>Client: ✅ 200 OK
            end
        else ❌ API Key غير متطابق
            Service->>Service: تجربة API Key التالي
        end
    end
    
    alt ❌ لم يتم العثور على API Key
        Service-->>Middleware: ❌ 401 Unauthorized
        Middleware-->>Client: رسالة خطأ
    end
```

#### 🔍 كيف يعمل التحقق (مثل getAuthHeaders في Tests)

في ملفات الاختبار مثل `EscrowApiTest.php`، يتم استخدام نفس الآلية:

```php
protected function getAuthHeaders()
{
    // استخدام المفتاح الأصلي (قبل التشفير)
    $apiKeyValue = $this->apiKeyValue ?? $this->apiKey->api_key;
    
    // إذا كان المفتاح مشفراً، فك التشفير
    if ($apiKeyValue === '********' || strlen($apiKeyValue) < 20) {
        $encryptedValue = $this->apiKey->getRawApiKey();
        $apiKeyValue = Crypt::decryptString($encryptedValue);
    }
    
    return [
        'Authorization' => 'Bearer ' . $apiKeyValue,
        'X-API-Key' => $apiKeyValue,  // بديل
        'X-API-Secret' => $this->apiSecretValue,  // إذا كان متوفراً
        'Accept' => 'application/json',
        'Content-Type' => 'application/json'
    ];
}
```

#### ✅ ملخص آلية تسجيل الدخول

1. **إرسال Headers:**
   - `X-API-Key: {api_key}` أو `Authorization: Bearer {api_key}`
   - `X-API-Secret: {api_secret}` (اختياري لكن مُوصى به)

2. **التحقق في Middleware (`SecureApiGateway`):**
   - يقرأ `X-API-Key` و `X-API-Secret` من Headers
   - يستدعي `ApiKeyService::validateApiKey($apiKey, $apiSecret)`

3. **التحقق في Service (`ApiKeyService`):**
   - البحث عن API Key في قاعدة البيانات (جميع المفاتيح النشطة)
   - فك تشفير كل API Key ومقارنته مع القيمة المرسلة
   - إذا تم إرسال Secret، فك تشفيره ومقارنته مع القيمة المحفوظة
   - التحقق من الصلاحيات و IP Whitelist و Rate Limit
   - تحديث `last_used_at` و `usage_count`

4. **الوصول للـ Controller:**
   - إذا نجح التحقق، يتم تمرير Request للـ Controller
   - API Key يتم حفظه في `$request->attributes->set('api_key', $apiKeyModel)`

5. **Response:**
   - Controller يعيد بيانات Partner و API Key

#### ❌ **خطأ شائع:** إرسال API Key في Body

**❌ خطأ:**
```json
{
  "api_key": "spk_live_...",
  "api_secret": "..."
}
```

**✅ صحيح:**
```http
X-API-Key: spk_live_...
X-API-Secret: ...
```

---

## 💳 Direct Payment API

### نظرة عامة

Direct Payment API يتيح معالجة المدفوعات مباشرة بدون استخدام خدمة الضمان. مناسب للمدفوعات الفورية والخدمات الرقمية.

### مخطط تدفق عملية الدفع المباشر

```mermaid
sequenceDiagram
    participant Client as 👤 العميل
    participant Partner as 🔌 الشريك (API)
    participant SafePay as 🏦 SafePay API
    participant Gateway as 💳 بوابة الدفع
    participant DB as 🗄️ قاعدة البيانات

    Client->>Partner: إرسال طلب دفع
    Partner->>SafePay: POST /api/partner/payments/initiate
    SafePay->>SafePay: التحقق من صحة البيانات
    SafePay->>SafePay: التحقق من الرصيد (إذا كانت wallet)
    
    alt ❌ رصيد غير كافي
        SafePay-->>Partner: 500 Insufficient Balance
        Partner-->>Client: رسالة خطأ
    else ✅ رصيد كافي
        SafePay->>DB: إنشاء سجل الدفع (pending)
        SafePay->>DB: إنشاء القيد المزدوج
        SafePay-->>Partner: 201 Payment Created
        Partner-->>Client: payment_url
        
        Client->>Gateway: تنفيذ الدفع
        Gateway-->>Client: تأكيد الدفع
        
        Client->>Partner: POST /api/partner/payments/confirm
        Partner->>SafePay: تأكيد الدفع
        SafePay->>DB: تحديث حالة الدفع (completed)
        SafePay->>DB: تحديث القيود المحاسبية
        SafePay-->>Partner: 200 Payment Confirmed
        Partner-->>Client: تأكيد النجاح
    end
```

### مخطط حالة الدفع

```mermaid
stateDiagram-v2
    [*] --> pending: إنشاء الدفع
    pending --> processing: بدء المعالجة
    processing --> completed: نجاح الدفع
    processing --> failed: فشل الدفع
    pending --> cancelled: إلغاء الدفع
    completed --> refunded: استرجاع المبلغ
    failed --> [*]
    cancelled --> [*]
    refunded --> [*]
    
    note right of pending
        حالة أولية
        في انتظار الدفع
    end note
    
    note right of completed
        تم الدفع بنجاح
        تم إنشاء القيود المحاسبية
    end note
```

### 1. بدء عملية دفع مباشر

**Endpoint:** `POST /api/partner/payments/initiate`

**الوصف:** إنشاء عملية دفع مباشر جديدة. يتم إنشاء القيد المزدوج تلقائياً.

**HTTP Method:** `POST`

**URL:** `/api/partner/payments/initiate`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | decimal | ✅ | المبلغ (أقل قيمة: 0.01) |
| `currency` | string | ✅ | العملة (3 أحرف: SAR, USD, etc.) |
| `payment_method` | string | ✅ | طريقة الدفع: `wallet`, `mada`, `credit_card`, `bank_transfer` |
| `customer_email` | email | ✅ | بريد العميل الإلكتروني |
| `description` | string | ❌ | وصف الدفع (حد أقصى 500 حرف) |
| `return_url` | url | ❌ | رابط العودة بعد النجاح |
| `cancel_url` | url | ❌ | رابط الإلغاء |
| `metadata` | object | ❌ | بيانات إضافية (JSON) |
| `customer_data` | object | ❌ | بيانات العميل: `firstname`, `lastname`, `phone` |

#### Request Example

```json
{
  "amount": 1000.00,
  "currency": "SAR",
  "payment_method": "wallet",
  "customer_email": "customer@example.com",
  "description": "دفع منتج رقمي",
  "return_url": "https://merchant.com/success",
  "cancel_url": "https://merchant.com/cancel",
  "metadata": {
    "order_id": "ORD-12345",
    "merchant_id": "MERCH-001"
  },
  "customer_data": {
    "firstname": "أحمد",
    "lastname": "محمد",
    "phone": "+966500000000"
  }
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "message": "Payment initiated successfully",
  "data": {
    "payment_id": "PAY-ZDKZODCQM",
    "transaction_id": "TRX1001",
    "payment_url": "https://safepay.com/payment/PAY-ZDKZODCQM",
    "status": "pending",
    "amount": 1000.00,
    "currency": "SAR",
    "payment_method": "wallet",
    "expires_at": "2025-01-25T16:00:00.000Z",
    "created_at": "2025-01-24T16:00:00.000Z"
  }
}
```

#### Response Example (Error - 422 Validation Error)

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": {
    "amount": ["The amount must be at least 0.01."],
    "customer_email": ["The customer email field is required."],
    "payment_method": ["The selected payment method is invalid."]
  }
}
```

#### Response Example (Error - 500 Insufficient Balance)

```json
{
  "success": false,
  "message": "Failed to initiate payment",
  "error": "Insufficient balance. Available: 500.00, Required: 1000.00"
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `201` | Created - تم إنشاء الدفع بنجاح |
| `400` | Bad Request - بيانات غير صحيحة |
| `401` | Unauthorized - API Key غير صالح |
| `422` | Validation Error - أخطاء في التحقق |
| `500` | Internal Server Error - خطأ في الخادم |

#### Notes / Constraints

- الحد الأدنى للمبلغ: **0.01** من العملة المحددة
- صلاحية رابط الدفع: **24 ساعة** من وقت الإنشاء
- طرق الدفع المدعومة: `wallet`, `mada`, `credit_card`, `bank_transfer`
- العملات المدعومة: `SAR`, `USD`, `EUR`, `GBP`

---

### 2. تأكيد الدفع المباشر

**Endpoint:** `POST /api/partner/payments/confirm`

**الوصف:** تأكيد عملية الدفع المباشر بعد اكتمال الدفع من العميل. يحدث الحالة من `pending` إلى `completed`.

**HTTP Method:** `POST`

**URL:** `/api/partner/payments/confirm`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payment_id` | string | ✅ | معرف الدفع المراد تأكيده |
| `payment_token` | string | ❌ | Token من بوابة الدفع |
| `payment_data` | object | ❌ | بيانات إضافية من بوابة الدفع |

#### Request Example

```json
{
  "payment_id": "PAY-ZDKZODCQM",
  "payment_token": "token_from_gateway_12345",
  "payment_data": {
    "gateway_response": "success",
    "transaction_reference": "TXN-123456"
  }
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "Payment confirmed successfully",
  "data": {
    "payment_id": "PAY-ZDKZODCQM",
    "transaction_id": "TRX1001",
    "status": "completed",
    "amount": 1000.00,
    "currency": "SAR",
    "paid_at": "2025-01-24T16:00:00.000Z",
    "payment_method": "wallet"
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | OK - تم التأكيد بنجاح |
| `400` | Bad Request - بيانات غير صحيحة |
| `404` | Not Found - الدفع غير موجود |
| `422` | Validation Error - الدفع في حالة غير قابلة للتأكيد |

---

### 3. استرجاع المبلغ (Refund)

**Endpoint:** `POST /api/partner/payments/refund`

**الوصف:** استرجاع مبلغ الدفع (كامل أو جزئي) باستخدام القيد المزدوج.

**HTTP Method:** `POST`

**URL:** `/api/partner/payments/refund`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payment_id` | string | ✅ | معرف الدفع المراد استرجاعه |
| `amount` | decimal | ❌ | المبلغ المراد استرجاعه (إذا لم يُحدد، يتم استرجاع المبلغ كاملاً) |
| `reason` | string | ❌ | سبب الاسترجاع |

#### Request Example (Full Refund)

```json
{
  "payment_id": "PAY-ZDKZODCQM",
  "reason": "طلب العميل"
}
```

#### Request Example (Partial Refund)

```json
{
  "payment_id": "PAY-ZDKZODCQM",
  "amount": 500.00,
  "reason": "استرجاع جزئي - منتج معيب"
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "Refund processed successfully",
  "data": {
    "refund_id": "REF-ABC123XYZ",
    "payment_id": "PAY-ZDKZODCQM",
    "amount": 500.00,
    "original_amount": 1000.00,
    "refund_type": "partial",
    "status": "completed",
    "reason": "استرجاع جزئي - منتج معيب",
    "refunded_at": "2025-01-24T16:30:00.000Z"
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | OK - تم الاسترجاع بنجاح |
| `400` | Bad Request - المبلغ أكبر من المبلغ الأصلي |
| `404` | Not Found - الدفع غير موجود |
| `422` | Validation Error - الدفع في حالة غير قابلة للاسترجاع |

#### Notes / Constraints

- يمكن استرجاع الدفع فقط إذا كان في حالة `completed`
- المبلغ المسترجع يجب أن يكون أقل من أو يساوي المبلغ الأصلي
- الاسترجاع الكامل: لا حاجة لتحديد `amount`
- الاسترجاع الجزئي: يجب تحديد `amount` أقل من المبلغ الأصلي

---

### 4. استعلام حالة الدفع

**Endpoint:** `GET /api/partner/payments/{payment_id}`

**الوصف:** الحصول على تفاصيل حالة الدفع.

**HTTP Method:** `GET`

**URL:** `/api/partner/payments/{payment_id}`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payment_id` | string | ✅ | معرف الدفع |

#### Request Example

```http
GET /api/partner/payments/PAY-ZDKZODCQM
Authorization: Bearer {api_key}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "payment_id": "PAY-ZDKZODCQM",
    "transaction_id": "TRX1001",
    "status": "completed",
    "amount": 1000.00,
    "currency": "SAR",
    "payment_method": "wallet",
    "customer_email": "customer@example.com",
    "description": "دفع منتج رقمي",
    "created_at": "2025-01-24T15:00:00.000Z",
    "paid_at": "2025-01-24T16:00:00.000Z",
    "expires_at": "2025-01-25T16:00:00.000Z",
    "metadata": {
      "order_id": "ORD-12345",
      "merchant_id": "MERCH-001"
    }
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | OK - تم الحصول على البيانات |
| `404` | Not Found - الدفع غير موجود |

---

## 🛡️ Escrow Management API

### نظرة عامة

Escrow Management API يتيح إدارة معاملات الضمان الإلكتروني. يوفر حماية للمشتري والبائع في المعاملات التجارية.

### مخطط تدفق عملية الضمان

```mermaid
sequenceDiagram
    participant Partner as 🔌 الشريك
    participant API as 🏦 SafePay API
    participant Buyer as 👤 المشتري
    participant Seller as 🏪 البائع
    participant DB as 🗄️ قاعدة البيانات

    Partner->>API: POST /api/partner/escrows/create-with-payment
    API->>API: التحقق من البيانات
    API->>DB: إنشاء سجل الضمان
    API->>DB: حساب العمولات والرسوم
    API->>DB: التحقق من رصيد المشتري
    
    alt ❌ رصيد غير كافي
        API-->>Partner: 500 Insufficient Balance
    else ✅ رصيد كافي
        API->>DB: خصم المبلغ من حساب المشتري
        API->>DB: إيداع المبلغ في حساب الضمان
        API->>DB: إنشاء القيود المحاسبية المزدوجة
        API-->>Partner: 201 Escrow Created
        
        Note over Buyer,Seller: فترة الفحص
        Buyer->>API: تأكيد التسليم
        API->>DB: تحديث حالة الضمان
        API->>DB: إفراج الأموال للبائع
        API->>DB: إنشاء قيود محاسبية للإفراج
        API-->>Seller: إشعار استلام المبلغ
    end
```

### مخطط حالة الضمان

```mermaid
stateDiagram-v2
    [*] --> awaiting_acceptance: إنشاء الضمان
    awaiting_acceptance --> awaiting_payment: قبول الضمان
    awaiting_payment --> awaiting_delivery: دفع المبلغ
    awaiting_delivery --> inspection_period: تأكيد التسليم
    inspection_period --> completed: انتهاء فترة الفحص
    inspection_period --> disputed: رفع نزاع
    disputed --> resolved: حل النزاع
    resolved --> completed: إفراج الأموال
    awaiting_acceptance --> cancelled: إلغاء
    awaiting_payment --> cancelled: إلغاء
    awaiting_delivery --> cancelled: إلغاء
    completed --> [*]
    cancelled --> [*]
    
    note right of awaiting_payment
        في انتظار دفع المشتري
        المبلغ محتفظ به في حساب الضمان
    end note
    
    note right of inspection_period
        فترة الفحص
        يمكن للمشتري رفع نزاع
    end note
```

### 1. إنشاء ضمان بدون دفع

**Endpoint:** `POST /api/partner/escrows`

**الوصف:** إنشاء معاملة ضمان جديدة بدون دفع (يتم الدفع لاحقاً).

**HTTP Method:** `POST`

**URL:** `/api/partner/escrows`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | ✅ | عنوان المعاملة (حد أقصى 255 حرف) |
| `description` | string | ✅ | وصف تفصيلي للمعاملة |
| `amount` | decimal | ✅ | المبلغ الإجمالي (أقل قيمة: 0.01) |
| `currency` | string | ✅ | العملة (3 أحرف: SAR) |
| `buyer_email` | email | ✅ | بريد المشتري |
| `seller_email` | email | ✅ | بريد البائع |
| `category_id` | integer | ✅ | معرف الفئة |
| `escrow_term_id` | integer | ❌ | معرف شروط الضمان |
| `type` | integer | ❌ | نوع الضمان: `1`=seller, `2`=buyer, `3`=broker |
| `inspection_period_days` | integer | ❌ | فترة الفحص بالأيام (1-30) |
| `auto_release_days` | integer | ❌ | فترة الإفراج التلقائي بالأيام (1-90) |
| `milestones` | array | ❌ | مراحل الدفع (للمعاملات المتعددة المراحل) |
| `buyer_data` | object | ❌ | بيانات المشتري: `firstname`, `lastname`, `phone` |
| `seller_data` | object | ❌ | بيانات البائع: `firstname`, `lastname`, `phone` |
| `metadata` | object | ❌ | بيانات إضافية (JSON) |

**Milestone Object:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | ✅ | عنوان المرحلة |
| `amount` | decimal | ✅ | مبلغ المرحلة |
| `percentage` | decimal | ❌ | نسبة المرحلة من المبلغ الإجمالي |
| `due_date` | date | ✅ | تاريخ استحقاق المرحلة |

#### Request Example

```json
{
  "title": "شراء لابتوب gaming",
  "description": "شراء لابتوب gaming مواصفات عالية - Intel i7, 16GB RAM, RTX 3060",
  "amount": 7500.00,
  "currency": "SAR",
  "buyer_email": "buyer@example.com",
  "seller_email": "seller@example.com",
  "category_id": 1,
  "escrow_term_id": 1,
  "type": 2,
  "inspection_period_days": 7,
  "buyer_data": {
    "firstname": "أحمد",
    "lastname": "محمد",
    "phone": "+966500000000"
  },
  "seller_data": {
    "firstname": "محمد",
    "lastname": "علي",
    "phone": "+966500000001"
  },
  "milestones": [
    {
      "title": "دفعة أولى",
      "amount": 3750.00,
      "percentage": 50,
      "due_date": "2025-02-01"
    },
    {
      "title": "دفعة نهائية",
      "amount": 3750.00,
      "percentage": 50,
      "due_date": "2025-02-15"
    }
  ]
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "message": "Escrow created successfully",
  "data": {
    "escrow_id": 123,
    "escrow_number": "ESC-ABC123XYZ",
    "title": "شراء لابتوب gaming",
    "amount": 7500.00,
    "currency": "SAR",
    "status": 0,
    "onboarding_status": "awaiting_acceptance",
    "buyer": {
      "id": 10,
      "email": "buyer@example.com",
      "name": "أحمد محمد"
    },
    "seller": {
      "id": 11,
      "email": "seller@example.com",
      "name": "محمد علي"
    },
    "milestones": [
      {
        "id": 5,
        "title": "دفعة أولى",
        "amount": 3750.00,
        "status": "pending"
      }
    ],
    "created_at": "2025-01-24T16:00:00.000Z"
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `201` | Created - تم إنشاء الضمان بنجاح |
| `400` | Bad Request - بيانات غير صحيحة |
| `422` | Validation Error - أخطاء في التحقق |

---

### 2. إنشاء ضمان مع دفع (عملية واحدة)

**Endpoint:** `POST /api/partner/escrows/create-with-payment`

**الوصف:** إنشاء معاملة ضمان ودفعها في عملية واحدة باستخدام القيد المزدوج.

**HTTP Method:** `POST`

**URL:** `/api/partner/escrows/create-with-payment`

#### Parameters

**Body Parameters:**

جميع معاملات إنشاء الضمان + معاملات الدفع:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payment_method` | string | ✅ | طريقة الدفع: `wallet`, `mada`, `credit_card`, `bank_transfer` |
| `payment_data` | object | ❌ | بيانات الدفع الإضافية |

#### Request Example

```json
{
  "title": "شراء لابتوب gaming",
  "description": "شراء لابتوب gaming مواصفات عالية",
  "amount": 7500.00,
  "currency": "SAR",
  "buyer_email": "buyer@example.com",
  "seller_email": "seller@example.com",
  "category_id": 1,
  "escrow_term_id": 1,
  "type": 2,
  "payment_method": "wallet",
  "payment_data": {},
  "inspection_days": 7,
  "buyer_data": {
    "firstname": "أحمد",
    "lastname": "محمد",
    "phone": "+966500000000"
  }
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "message": "تم إنشاء معاملة الضمان ودفعها بنجاح",
  "data": {
    "escrow": {
      "id": 123,
      "escrow_number": "ESC-ABC123XYZ",
      "title": "شراء لابتوب gaming",
      "amount": 7500.00,
      "total_paid": 7650.00,
      "status": 2,
      "onboarding_status": "awaiting_delivery"
    },
    "payment": {
      "transaction_id": "TRX1001",
      "method": "wallet",
      "amount": 7650.00,
      "status": "completed"
    },
    "commissions": {
      "total_charge": 150.00,
      "safepay_commission": 100.00,
      "partner_commission": 50.00,
      "buyer_charge": 150.00,
      "seller_charge": 0.00
    }
  }
}
```

---

### 3. دفع ضمان موجود

**Endpoint:** `POST /api/partner/escrows/{escrow_id}/pay`

**الوصف:** دفع ضمان موجود (كامل أو مرحلة معينة).

**HTTP Method:** `POST`

**URL:** `/api/partner/escrows/{escrow_id}/pay`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `escrow_id` | integer | ✅ | معرف الضمان |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payment_method` | string | ✅ | طريقة الدفع |
| `payment_data` | object | ❌ | بيانات الدفع |
| `milestone_id` | integer | ❌ | معرف المرحلة (إذا كان الدفع لمرحلة معينة) |

#### Request Example (Full Payment)

```json
{
  "payment_method": "wallet",
  "payment_data": {}
}
```

#### Request Example (Milestone Payment)

```json
{
  "payment_method": "wallet",
  "payment_data": {},
  "milestone_id": 5
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "Payment processed successfully",
  "data": {
    "escrow_id": 123,
    "transaction_id": "TRX1001",
    "amount": 3750.00,
    "method": "wallet",
    "funded_amount": 3750.00,
    "status": "awaiting_paid"
  }
}
```

---

### 4. إفراج الأموال من الضمان

**Endpoint:** `POST /api/partner/escrows/{escrow_id}/release`

**الوصف:** إفراج الأموال من حساب الضمان إلى حساب البائع باستخدام القيد المزدوج.

**HTTP Method:** `POST`

**URL:** `/api/partner/escrows/{escrow_id}/release`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `escrow_id` | integer | ✅ | معرف الضمان |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | decimal | ✅ | المبلغ المراد إفراجه |
| `reason` | string | ❌ | سبب الإفراج |

#### Request Example

```json
{
  "amount": 7500.00,
  "reason": "تأكيد التسليم - العميل راضٍ عن المنتج"
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "Funds released successfully",
  "data": {
    "escrow_id": 123,
    "escrow_number": "ESC-ABC123XYZ",
    "amount": 7500.00,
    "status": 1,
    "onboarding_status": "completed",
    "released_at": "2025-01-24T17:00:00.000Z"
  }
}
```

---

### 5. الحصول على تفاصيل الضمان

**Endpoint:** `GET /api/partner/escrows/{escrow_id}`

**الوصف:** الحصول على تفاصيل معاملة الضمان.

**HTTP Method:** `GET`

**URL:** `/api/partner/escrows/{escrow_id}`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `escrow_id` | integer | ✅ | معرف الضمان |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "id": 123,
    "escrow_number": "ESC-ABC123XYZ",
    "title": "شراء لابتوب gaming",
    "amount": 7500.00,
    "funded_amount": 7500.00,
    "status": 1,
    "onboarding_status": "awaiting_delivery",
    "buyer": {
      "id": 10,
      "email": "buyer@example.com",
      "name": "أحمد محمد"
    },
    "seller": {
      "id": 11,
      "email": "seller@example.com",
      "name": "محمد علي"
    },
    "milestones": [
      {
        "id": 5,
        "title": "دفعة أولى",
        "amount": 3750.00,
        "status": "completed",
        "payment_status": "paid"
      }
    ],
    "created_at": "2025-01-24T16:00:00.000Z"
  }
}
```

---

### 6. قائمة معاملات الضمان

**Endpoint:** `GET /api/partner/escrows`

**الوصف:** الحصول على قائمة معاملات الضمان مع إمكانية التصفية والترقيم.

**HTTP Method:** `GET`

**URL:** `/api/partner/escrows`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `status` | string | ❌ | حالة الضمان: `pending`, `active`, `completed`, `cancelled` |
| `per_page` | integer | ❌ | عدد النتائج في الصفحة (افتراضي: 15) |
| `page` | integer | ❌ | رقم الصفحة (افتراضي: 1) |
| `date_from` | date | ❌ | تاريخ البداية |
| `date_to` | date | ❌ | تاريخ النهاية |

#### Request Example

```http
GET /api/partner/escrows?status=pending&per_page=15&page=1
Authorization: Bearer {api_key}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "escrows": [
      {
        "id": 123,
        "escrow_number": "ESC-ABC123XYZ",
        "title": "شراء لابتوب gaming",
        "amount": 7500.00,
        "status": 1,
        "onboarding_status": "awaiting_delivery",
        "created_at": "2025-01-24T16:00:00.000Z"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 15,
      "total": 50,
      "last_page": 4,
      "from": 1,
      "to": 15
    }
  }
}
```

---

### 7. إحصائيات معاملات الضمان

**Endpoint:** `GET /api/partner/escrows/stats`

**الوصف:** الحصول على إحصائيات شاملة لمعاملات الضمان.

**HTTP Method:** `GET`

**URL:** `/api/partner/escrows/stats`

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "total_escrows": 150,
    "total_amount": 2000000.00,
    "pending_escrows": 10,
    "active_escrows": 120,
    "completed_escrows": 20,
    "cancelled_escrows": 0,
    "by_status": {
      "pending": 10,
      "active": 120,
      "completed": 20,
      "cancelled": 0
    },
    "by_category": {
      "1": 50,
      "2": 100
    }
  }
}
```

---

### 8. إلغاء معاملة الضمان

**Endpoint:** `POST /api/partner/escrows/{escrow_id}/cancel`

**الوصف:** إلغاء معاملة ضمان موجودة.

**HTTP Method:** `POST`

**URL:** `/api/partner/escrows/{escrow_id}/cancel`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `reason` | string | ❌ | سبب الإلغاء |

#### Request Example

```json
{
  "reason": "طلب الإلغاء من المشتري"
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "Escrow cancelled successfully",
  "data": {
    "escrow_id": 123,
    "status": "cancelled",
    "cancelled_at": "2025-01-24T18:00:00.000Z"
  }
}
```

---

## 💰 Payment Management API

### نظرة عامة

Payment Management API يتيح إدارة المدفوعات المرتبطة بمعاملات الضمان أو المدفوعات المباشرة.

### 1. إنشاء طلب دفع

**Endpoint:** `POST /api/partner/payments`

**الوصف:** إنشاء طلب دفع (يمكن ربطه بـ Escrow أو دفع مباشر).

**HTTP Method:** `POST`

**URL:** `/api/partner/payments`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `escrow_id` | integer | ❌ | معرف الضمان (إذا كان الدفع مرتبط بضمان) |
| `amount` | decimal | ✅ | المبلغ |
| `currency` | string | ✅ | العملة |
| `payment_method` | string | ✅ | طريقة الدفع |
| `description` | string | ❌ | وصف الدفع |
| `customer_email` | email | ✅ | بريد العميل (مطلوب إذا لم يكن هناك escrow_id) |

#### Request Example (مع Escrow)

```json
{
  "escrow_id": 123,
  "amount": 7500.00,
  "currency": "SAR",
  "payment_method": "wallet",
  "description": "دفع ضمان"
}
```

#### Request Example (دفع مباشر)

```json
{
  "amount": 1000.00,
  "currency": "SAR",
  "payment_method": "wallet",
  "customer_email": "customer@example.com",
  "description": "دفع مباشر"
}
```

---

### 2. إلغاء طلب الدفع

**Endpoint:** `POST /api/partner/payments/{payment_id}/cancel`

**الوصف:** إلغاء طلب دفع موجود.

**HTTP Method:** `POST`

**URL:** `/api/partner/payments/{payment_id}/cancel`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `reason` | string | ❌ | سبب الإلغاء |

---

### 3. إحصائيات المدفوعات

**Endpoint:** `GET /api/partner/payments/stats`

**الوصف:** الحصول على إحصائيات المدفوعات.

**HTTP Method:** `GET`

**URL:** `/api/partner/payments/stats`

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "total_payments": 150,
    "total_amount": 500000.00,
    "completed_payments": 140,
    "pending_payments": 10,
    "failed_payments": 0,
    "by_method": {
      "wallet": 80,
      "mada": 50,
      "credit_card": 15,
      "bank_transfer": 5
    }
  }
}
```

---

## 💸 Settlement API

### نظرة عامة

Settlement API يتيح إدارة طلبات التسوية (سحب الأموال من الحساب). عند إنشاء التسوية من قبل الشريك، يتم إنشاء طلب تسوية بحالة `pending` في انتظار موافقة الإدارة.

**ملاحظات مهمة:**
- ✅ يتم إنشاء **تفاصيل التسوية (SettlementDetails)** تلقائياً عند إنشاء التسوية
- ✅ يتم حساب **الرسوم والضرائب** تلقائياً (VAT، Processing Fees، Commission)
- ✅ **القيود المحاسبية المزدوجة (Double Entry)** تُنشأ فقط بعد موافقة الإدارة
- ✅ يمكن ربط التسوية بمعاملات أو ضمانات محددة
- ✅ يتم تسجيل جميع العمليات في Activity Log
- ⚠️ **التسويات التي ينشئها الشريك تكون بحالة `pending` وتحتاج موافقة من الإدارة**

### مخطط تدفق عملية التسوية

```mermaid
sequenceDiagram
    participant Partner as 🔌 الشريك
    participant API as 🏦 SafePay API
    participant Admin as 👨‍💼 الإدارة
    participant Accounting as 📊 النظام المحاسبي
    participant DB as 🗄️ قاعدة البيانات

    Partner->>API: POST /api/partner/settlements
    API->>API: التحقق من البيانات
    API->>DB: إنشاء سجل التسوية (pending)
    API->>DB: إنشاء تفاصيل التسوية (SettlementDetails)
    API->>API: حساب الرسوم والضرائب
    API-->>Partner: 201 Settlement Created (pending)
    
    Note over Partner,Admin: في انتظار الموافقة
    
    Admin->>API: مراجعة طلب التسوية
    Admin->>API: الموافقة على التسوية
    
    API->>Accounting: إنشاء القيود المحاسبية المزدوجة
    Accounting->>DB: تسجيل القيود (Debit/Credit)
    API->>DB: تحديث حالة التسوية (processing)
    API->>DB: ربط المعاملات/الضمانات
    API->>DB: تحديث حالة التسوية (completed)
    API-->>Admin: تأكيد الموافقة
    API-->>Partner: إشعار موافقة التسوية
```

### مخطط حالة التسوية

```mermaid
stateDiagram-v2
    [*] --> pending: الشريك يطلب التسوية
    pending --> processing: موافقة الإدارة
    pending --> rejected: رفض الإدارة
    processing --> completed: اكتمال المعالجة
    processing --> failed: فشل المعالجة
    completed --> [*]
    rejected --> [*]
    failed --> [*]
    
    note right of pending
        حالة أولية
        بدون قيود محاسبية
        في انتظار الموافقة
    end note
    
    note right of processing
        بعد الموافقة
        تم إنشاء القيود المحاسبية
        جاري المعالجة
    end note
    
    note right of completed
        اكتمال التسوية
        تم تحويل الأموال
        تم تحديث جميع السجلات
    end note
```

### مخطط العلاقات في نظام التسوية

```mermaid
erDiagram
    SETTLEMENT ||--o{ SETTLEMENT_DETAIL : يحتوي_على
    SETTLEMENT ||--o{ ACCOUNTING_ENTRY : ينشئ
    SETTLEMENT }o--|| PARTNER : يطلب
    SETTLEMENT_DETAIL }o--|| TRANSACTION : مرتبط_ب
    SETTLEMENT_DETAIL }o--|| ESCROW : مرتبط_ب
    ACCOUNTING_ENTRY }o--|| ACCOUNT : يؤثر_على
    
    SETTLEMENT {
        int id PK
        int partner_id FK
        decimal amount
        string status
        string payment_method
        datetime created_at
        datetime approved_at
    }
    
    SETTLEMENT_DETAIL {
        int id PK
        int settlement_id FK
        int transaction_id FK
        int escrow_id FK
        decimal amount
        decimal fees
        decimal taxes
    }
    
    ACCOUNTING_ENTRY {
        int id PK
        int settlement_id FK
        int account_id FK
        decimal debit_amount
        decimal credit_amount
        string trx_type
    }
```

### 1. إنشاء طلب تسوية

**Endpoint:** `POST /api/partner/settlements`

**الوصف:** إنشاء طلب تسوية جديد. يتم إنشاء:
- سجل التسوية (Settlement) بحالة `pending`
- تفاصيل التسوية (SettlementDetails) من المعاملات غير المسوية
- حساب الرسوم والضرائب تلقائياً
- **القيود المحاسبية المزدوجة (Double Entry Transactions) تُنشأ فقط بعد موافقة الإدارة**

**HTTP Method:** `POST`

**URL:** `/api/partner/settlements`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | decimal | ✅ | المبلغ المراد تسويته (أقل قيمة: 0.01) |
| `currency` | string | ✅ | العملة (3 أحرف: SAR, USD, etc.) |
| `settlement_method` | string | ❌ | طريقة التسوية: `bank_transfer`, `check`, `cash` |
| `bank_account` | object | ❌ | بيانات الحساب البنكي (مطلوب إذا كانت الطريقة bank_transfer) |
| `reason` | string | ❌ | سبب التسوية (حد أقصى 500 حرف) |
| `scheduled_at` | date | ❌ | تاريخ الجدولة (يجب أن يكون بعد اليوم) |
| `transaction_ids` | array | ❌ | قائمة معرفات المعاملات المراد ربطها بالتسوية |
| `escrow_ids` | array | ❌ | قائمة معرفات الضمانات المراد ربطها بالتسوية |
| `apply_to_existing` | boolean | ❌ | تطبيق الإعدادات على المشاريع الموجودة (افتراضي: false) |

**Bank Account Object:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_number` | string | ✅ | رقم الحساب |
| `bank_name` | string | ✅ | اسم البنك |
| `iban` | string | ❌ | رقم IBAN |

#### Request Example (تسوية بسيطة)

```json
{
  "amount": 10000.00,
  "currency": "SAR",
  "settlement_method": "bank_transfer",
  "bank_account": {
    "account_number": "1234567890",
    "bank_name": "البنك الأهلي",
    "iban": "SA1234567890123456789012"
  },
  "reason": "طلب تسوية من الشريك"
}
```

#### Request Example (تسوية مع معاملات وضمانات محددة)

```json
{
  "amount": 50000.00,
  "currency": "SAR",
  "settlement_method": "bank_transfer",
  "bank_account": {
    "account_number": "1234567890",
    "bank_name": "البنك الأهلي",
    "iban": "SA1234567890123456789012"
  },
  "transaction_ids": [1001, 1002, 1003],
  "escrow_ids": [50, 51],
  "reason": "تسوية معاملات وضمانات محددة"
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "message": "تم إنشاء طلب التسوية بنجاح - في انتظار الموافقة من الإدارة",
  "data": {
    "settlement_id": 10,
    "amount": 10000.00,
    "currency": "SAR",
    "status": "pending",
    "settlement_method": "bank_transfer",
    "scheduled_at": "2025-01-25T10:00:00.000Z",
    "accounting_entries_created": false,
    "requires_approval": true,
    "note": "سيتم إنشاء القيود المحاسبية بعد الموافقة من الإدارة",
    "created_at": "2025-01-24T10:00:00.000Z"
  }
}
```

#### Response Example (Success - 201 Created - مع تفاصيل)

```json
{
  "success": true,
  "message": "تم إنشاء طلب التسوية الكاملة بنجاح - في انتظار الموافقة من الإدارة",
  "data": {
    "settlement_id": 10,
    "amount": 50000.00,
    "currency": "SAR",
    "status": "pending",
    "details_count": 5,
    "accounting_entries_created": false,
    "requires_approval": true,
    "note": "سيتم إنشاء القيود المحاسبية بعد الموافقة من الإدارة"
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `201` | Created - تم إنشاء طلب التسوية بنجاح (في انتظار الموافقة) |
| `400` | Bad Request - بيانات غير صحيحة |
| `401` | Unauthorized - API Key غير صالح |
| `422` | Validation Error - أخطاء في التحقق |
| `500` | Internal Server Error - خطأ في الخادم أو رصيد غير كافي |

#### Notes / Constraints

- ⚠️ **التسويات التي ينشئها الشريك تكون بحالة `pending` وتحتاج موافقة من الإدارة**
- ✅ يتم إنشاء **تفاصيل التسوية (SettlementDetails)** تلقائياً عند الإنشاء
- ✅ يتم حساب **الرسوم والضرائب** تلقائياً (VAT، Processing Fees)
- ⚠️ **القيود المحاسبية المزدوجة تُنشأ فقط بعد موافقة الإدارة** (عند الموافقة يتم إنشاء القيود تلقائياً)
- ✅ إذا تم تحديد `transaction_ids` أو `escrow_ids`، يتم ربط المعاملات/الضمانات المحددة بالتسوية
- ✅ الحد الأدنى للمبلغ: **0.01** من العملة المحددة
- ✅ يتم تسجيل العملية في Activity Log تلقائياً
- 📝 **ملاحظة:** بعد موافقة الإدارة، يتم إنشاء القيود المحاسبية المزدوجة تلقائياً وتصبح التسوية بحالة `processing` أو `completed`

---

### 2. قائمة طلبات التسوية

**Endpoint:** `GET /api/partner/settlements`

**الوصف:** الحصول على قائمة طلبات التسوية.

**HTTP Method:** `GET`

**URL:** `/api/partner/settlements`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `status` | string | ❌ | حالة التسوية: `pending`, `processing`, `completed`, `cancelled` |
| `per_page` | integer | ❌ | عدد النتائج في الصفحة |
| `page` | integer | ❌ | رقم الصفحة |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "settlements": [
      {
        "id": 10,
        "amount": 10000.00,
        "status": "completed",
        "settlement_method": "bank_transfer",
        "created_at": "2025-01-24T10:00:00.000Z",
        "completed_at": "2025-01-25T14:00:00.000Z"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 15,
      "total": 25,
      "last_page": 2
    }
  }
}
```

---

### 3. الحصول على حالة التسوية

**Endpoint:** `GET /api/partner/settlements/{settlement_id}`

**الوصف:** الحصول على تفاصيل طلب تسوية معين.

**HTTP Method:** `GET`

**URL:** `/api/partner/settlements/{settlement_id}`

---

### 4. إحصائيات التسويات

**Endpoint:** `GET /api/partner/settlements/stats`

**الوصف:** الحصول على إحصائيات التسويات.

**HTTP Method:** `GET`

**URL:** `/api/partner/settlements/stats`

---

## ⚙️ Escrow Configuration API

### نظرة عامة

Escrow Configuration API يتيح إدارة إعدادات الضمان للشريك. يتيح تحديث شروط الضمان، إعدادات المراحل، الرسوم، وقواعد النزاعات.

### مخطط تدفق تحديث إعدادات الضمان

```mermaid
sequenceDiagram
    participant Partner as 🔌 الشريك
    participant API as 🏦 SafePay API
    participant Validation as ✅ التحقق
    participant Approval as 📋 نظام الموافقة
    participant DB as 🗄️ قاعدة البيانات

    Partner->>API: PUT /api/partner/escrow-config
    API->>Validation: التحقق من صحة البيانات
    Validation->>Validation: التحقق من مجموع نسب المراحل (= 100%)
    Validation->>Validation: التحقق من متطلبات SafePay (SAMA)
    
    alt ❌ بيانات غير صحيحة
        Validation-->>API: أخطاء التحقق
        API-->>Partner: 422 Validation Error
    else ✅ بيانات صحيحة
        API->>DB: التحقق من طلبات الموافقة القائمة
        
        alt يوجد طلب موافقة pending
            API->>DB: تحديث طلب الموافقة الموجود
            API-->>Partner: 200 Updated (pending approval)
        else لا يوجد طلب موافقة
            alt التغييرات تتطلب موافقة
                API->>DB: إنشاء طلب موافقة جديد
                API->>Approval: إرسال طلب الموافقة
                API-->>Partner: 200 Created (pending approval)
            else التغييرات لا تتطلب موافقة
                API->>DB: تحديث الإعدادات مباشرة
                API-->>Partner: 200 Updated Successfully
            end
        end
    end
```

**ملاحظات مهمة:**
- ✅ عند تحديث الإعدادات، يتم إنشاء طلب موافقة إذا كانت هناك تغييرات تتطلب موافقة
- ✅ إذا كان هناك طلب موافقة قائم (`pending`)، يتم تحديثه بدلاً من إنشاء طلب جديد
- ✅ يتم التحقق من صحة البيانات قبل التحديث (مثل: مجموع نسب المراحل = 100%)
- ✅ يتم التحقق من متطلبات SafePay (SAMA compliance)

### 1. الحصول على إعدادات الضمان

**Endpoint:** `GET /api/partner/escrow-config`

**الوصف:** الحصول على إعدادات الضمان الحالية للشريك.

**HTTP Method:** `GET`

**URL:** `/api/partner/escrow-config`

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "id": 1,
    "partner_id": 1,
    "commission_rate": 2.5,
    "fixed_commission": 0,
    "min_amount": 10.00,
    "max_amount": 1000000.00,
    "held_percentage": 10,
    "inspection_period_days": 7,
    "dispute_period_days": 14,
    "auto_release_days": 30,
    "auto_release": true,
    "milestone_settings": [
      {
        "name": "دفعة أولى",
        "percentage": 50
      },
      {
        "name": "دفعة نهائية",
        "percentage": 50
      }
    ],
    "fee_structure": {
      "escrow_fee_percentage": 2.5
    },
    "dispute_rules": {
      "resolution_days": 14
    },
    "status": "active",
    "created_at": "2025-01-24T10:00:00.000Z"
  }
}
```

---

### 2. تحديث إعدادات الضمان

**Endpoint:** `PUT /api/partner/escrow-config`

**الوصف:** تحديث إعدادات الضمان للشريك. إذا كانت التغييرات تتطلب موافقة، يتم إنشاء طلب موافقة (أو تحديث الطلب الموجود إذا كان هناك طلب `pending`).

**HTTP Method:** `PUT`

**URL:** `/api/partner/escrow-config`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `commission_rate` | decimal | ❌ | نسبة العمولة (0-100) |
| `fixed_commission` | decimal | ❌ | عمولة ثابتة |
| `min_amount` | decimal | ❌ | الحد الأدنى للمبلغ |
| `max_amount` | decimal | ❌ | الحد الأقصى للمبلغ (يجب أن يكون أكبر من min_amount) |
| `held_percentage` | decimal | ❌ | نسبة المبلغ المحتفظ به (0-100) |
| `inspection_period_days` | integer | ❌ | فترة الفحص بالأيام (1-365) |
| `dispute_period_days` | integer | ❌ | فترة النزاع بالأيام (1-365) |
| `auto_release_days` | integer | ❌ | فترة الإفراج التلقائي بالأيام (0-365) |
| `auto_release` | boolean | ❌ | تفعيل الإفراج التلقائي |
| `milestone_settings` | array | ❌ | إعدادات المراحل (يجب أن يكون مجموع النسب = 100%) |
| `fee_structure` | object | ❌ | هيكل الرسوم |
| `dispute_rules` | object | ❌ | قواعد النزاعات |

**Milestone Settings Object:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | ✅ | اسم المرحلة |
| `percentage` | decimal | ✅ | نسبة المرحلة (0.01-100) |

#### Request Example

```json
{
  "commission_rate": 3.0,
  "min_amount": 50.00,
  "max_amount": 500000.00,
  "held_percentage": 15,
  "inspection_period_days": 10,
  "dispute_period_days": 21,
  "auto_release_days": 45,
  "auto_release": true,
  "milestone_settings": [
    {
      "name": "دفعة أولى",
      "percentage": 40
    },
    {
      "name": "دفعة ثانية",
      "percentage": 30
    },
    {
      "name": "دفعة نهائية",
      "percentage": 30
    }
  ],
  "fee_structure": {
    "escrow_fee_percentage": 3.0
  },
  "dispute_rules": {
    "resolution_days": 21
  }
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "تم تحديث إعدادات الضمان بنجاح",
  "data": {
    "id": 1,
    "partner_id": 1,
    "commission_rate": 3.0,
    "min_amount": 50.00,
    "max_amount": 500000.00,
    "status": "active",
    "updated_at": "2025-01-24T11:00:00.000Z"
  }
}
```

#### Response Example (Success - 200 OK - مع طلب موافقة)

```json
{
  "success": true,
  "message": "تم إنشاء طلب موافقة للتغييرات",
  "data": {
    "id": 1,
    "partner_id": 1,
    "status": "pending_approval",
    "approval_request": {
      "id": 5,
      "status": "pending",
      "created_at": "2025-01-24T11:00:00.000Z"
    },
    "updated_at": "2025-01-24T11:00:00.000Z"
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | OK - تم التحديث بنجاح |
| `400` | Bad Request - بيانات غير صحيحة |
| `422` | Validation Error - أخطاء في التحقق (مثل: مجموع نسب المراحل ≠ 100%) |
| `500` | Internal Server Error - خطأ في الخادم |

#### Notes / Constraints

- ⚠️ **إذا كان هناك طلب موافقة قائم (`pending`)، يتم تحديثه بدلاً من إنشاء طلب جديد** - هذا يمنع إنشاء طلبات موافقة مكررة
- ✅ مجموع نسب المراحل (`milestone_settings`) يجب أن يساوي **100%** بالضبط
- ✅ `max_amount` يجب أن يكون أكبر من `min_amount`
- ✅ جميع القيم الرقمية يجب أن تكون ضمن النطاقات المحددة
- ✅ يتم التحقق من متطلبات SafePay (SAMA compliance) تلقائياً
- 📝 **ملاحظة:** بعض التغييرات قد تتطلب موافقة من الإدارة قبل التطبيق

---

## 📊 Reports & Analytics API

### نظرة عامة

Reports & Analytics API يوفر تقارير شاملة وتحليلات متقدمة للمعاملات والمدفوعات.

### مخطط تدفق إنشاء التقارير

```mermaid
flowchart TD
    A[الشريك يطلب تقرير] --> B{نوع التقرير}
    B -->|معاملات| C[تقرير المعاملات]
    B -->|مدفوعات| D[تقرير المدفوعات]
    B -->|تسويات| E[تقرير التسويات]
    B -->|أداء| F[تقرير الأداء]
    B -->|مخاطر| G[تقرير المخاطر]
    
    C --> H[تصفية حسب التاريخ والحالة]
    D --> I[البحث في details field]
    E --> J[استخدام partner_id و payment_method]
    
    H --> K[تجميع البيانات]
    I --> K
    J --> K
    
    K --> L{تنسيق التصدير}
    L -->|JSON| M[إرجاع JSON]
    L -->|CSV| N[تصدير CSV]
    L -->|PDF| O[تصدير PDF]
    
    M --> P[إرجاع النتيجة]
    N --> P
    O --> P
    
    style A fill:#e1f5ff
    style K fill:#fff4e1
    style P fill:#e8f5e9
```

### 1. لوحة التحكم

**Endpoint:** `GET /api/partner/dashboard`

**الوصف:** الحصول على بيانات لوحة التحكم الرئيسية.

**HTTP Method:** `GET`

**URL:** `/api/partner/dashboard`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `date_from` | date | ❌ | تاريخ البداية |
| `date_to` | date | ❌ | تاريخ النهاية |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "overview": {
      "total_escrows": 150,
      "total_payments": 500,
      "total_amount": 2000000.00,
      "pending_escrows": 10,
      "active_escrows": 120,
      "completed_escrows": 20
    },
    "recent_activity": [
      {
        "type": "escrow_created",
        "description": "تم إنشاء ضمان جديد",
        "timestamp": "2025-01-24T16:00:00.000Z"
      }
    ],
    "performance_metrics": {
      "completion_rate": 95.5,
      "average_processing_time": "2.5 days"
    }
  }
}
```

---

### 2. الرصيد المتاح

**Endpoint:** `GET /api/partner/balance`

**الوصف:** الحصول على الرصيد المتاح في الحساب.

**HTTP Method:** `GET`

**URL:** `/api/partner/balance`

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "available_balance": 50000.00,
    "pending_balance": 10000.00,
    "currency": "SAR"
  }
}
```

---

### 3. التحليلات المتقدمة

**Endpoint:** `GET /api/partner/analytics`

**الوصف:** الحصول على تحليلات متقدمة.

**HTTP Method:** `GET`

**URL:** `/api/partner/analytics`

---

### 4. تقارير المعاملات

**Endpoint:** `GET /api/partner/reports/transactions`

**الوصف:** الحصول على تقرير المعاملات.

**HTTP Method:** `GET`

**URL:** `/api/partner/reports/transactions`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `date_from` | date | ✅ | تاريخ البداية |
| `date_to` | date | ✅ | تاريخ النهاية |
| `status` | string | ❌ | حالة المعاملة |
| `category_id` | integer | ❌ | معرف الفئة |
| `export_format` | string | ❌ | تنسيق التصدير: `json`, `csv`, `pdf` |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "total_transactions": 150,
    "total_amount": 500000.00,
    "average_amount": 3333.33,
    "transactions": [
      {
        "id": 1001,
        "amount": 5000.00,
        "status": 2,
        "created_at": "2025-01-24T10:00:00.000Z"
      }
    ]
  }
}
```

---

### 5. تقرير المدفوعات

**Endpoint:** `GET /api/partner/reports/payments`

**الوصف:** الحصول على تقرير المدفوعات مع إمكانية التصفية حسب طريقة الدفع والحالة.

**HTTP Method:** `GET`

**URL:** `/api/partner/reports/payments`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `date_from` | date | ✅ | تاريخ البداية |
| `date_to` | date | ✅ | تاريخ النهاية |
| `payment_method` | string | ❌ | طريقة الدفع: `wallet`, `mada`, `credit_card`, `bank_transfer` |
| `status` | string | ❌ | حالة الدفع: `pending`, `completed`, `failed`, `cancelled` |
| `export_format` | string | ❌ | تنسيق التصدير: `json`, `csv`, `pdf` |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "total_payments": 120,
    "total_amount": 350000.00,
    "payments": [
      {
        "id": 501,
        "trx": "TRX1001",
        "amount": 5000.00,
        "currency": "SAR",
        "status": "completed",
        "created_at": "2025-01-24T10:00:00.000Z",
        "details": "Payment for order ORD-12345"
      }
    ]
  }
}
```

#### Notes / Constraints

- ✅ يتم البحث عن المدفوعات من خلال `details` field في جدول `transactions`
- ✅ حالة الدفع (`status`) تُحول من رقم إلى نص: `0,1` = `pending`, `2` = `completed`, `3` = `failed`, `4` = `cancelled`

---

### 6. تقرير التسويات

**Endpoint:** `GET /api/partner/reports/settlements`

**الوصف:** الحصول على تقرير التسويات مع إمكانية التصفية حسب طريقة التسوية والحالة.

**HTTP Method:** `GET`

**URL:** `/api/partner/reports/settlements`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `date_from` | date | ✅ | تاريخ البداية |
| `date_to` | date | ✅ | تاريخ النهاية |
| `settlement_method` | string | ❌ | طريقة التسوية: `bank_transfer`, `check`, `cash` |
| `status` | string | ❌ | حالة التسوية: `pending`, `processing`, `completed`, `failed`, `cancelled` |
| `export_format` | string | ❌ | تنسيق التصدير: `json`, `csv`, `pdf` |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "total_settlements": 25,
    "total_amount": 150000.00,
    "settlements": [
      {
        "id": 10,
        "amount": 10000.00,
        "currency": "SAR",
        "status": "pending",
        "type": "payout",
        "payment_method": "bank_transfer",
        "created_at": "2025-01-24T10:00:00.000Z",
        "completed_at": null
      }
    ]
  }
}
```

#### Notes / Constraints

- ✅ يتم استخدام `partner_id` للبحث عن التسويات (وليس `beneficiary_id`)
- ✅ يتم استخدام `payment_method` للتصفية (وليس `settlement_method`)

---

## 🔔 Webhooks API

### نظرة عامة

Webhooks API يتيح إدارة Webhooks للإشعارات الفورية.

### 1. قائمة Webhooks

**Endpoint:** `GET /api/partner/webhooks`

**الوصف:** الحصول على قائمة Webhooks.

**HTTP Method:** `GET`

**URL:** `/api/partner/webhooks`

---

### 2. إنشاء Webhook

**Endpoint:** `POST /api/partner/webhooks`

**الوصف:** إنشاء Webhook جديد.

**HTTP Method:** `POST`

**URL:** `/api/partner/webhooks`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | url | ✅ | عنوان URL للـ Webhook |
| `events` | array | ✅ | قائمة الأحداث المراد الاشتراك فيها |
| `secret` | string | ❌ | Secret للتوقيع (سيتم إنشاؤه تلقائياً إذا لم يُحدد) |
| `description` | string | ❌ | وصف Webhook |
| `retry_count` | integer | ❌ | عدد المحاولات (افتراضي: 3) |
| `timeout` | integer | ❌ | المهلة الزمنية بالثواني (افتراضي: 30) |

#### Request Example

```json
{
  "url": "https://merchant.com/webhooks/safepay",
  "events": [
    "payment.completed",
    "payment.failed",
    "escrow.created",
    "escrow.released"
  ],
  "description": "Webhook للمدفوعات والضمان",
  "retry_count": 3,
  "timeout": 30
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "data": {
    "id": 1,
    "url": "https://merchant.com/webhooks/safepay",
    "events": [
      "payment.completed",
      "payment.failed",
      "escrow.created",
      "escrow.released"
    ],
    "status": "active",
    "secret": "whsec_1234567890abcdef",
    "created_at": "2025-01-24T16:00:00.000Z"
  }
}
```

---

### 3. Webhook Events

الأحداث المتاحة للاشتراك:

| Event | Description |
|-------|-------------|
| `payment.initiated` | بدء عملية دفع |
| `payment.completed` | اكتمال الدفع |
| `payment.failed` | فشل الدفع |
| `payment.refunded` | استرجاع المبلغ |
| `escrow.created` | إنشاء ضمان |
| `escrow.paid` | دفع ضمان |
| `escrow.released` | إفراج الأموال |
| `escrow.completed` | اكتمال الضمان |
| `escrow.cancelled` | إلغاء الضمان |
| `dispute.opened` | فتح نزاع |
| `dispute.resolved` | حل النزاع |

### 4. Webhook Payload Example

```json
{
  "event": "payment.completed",
  "timestamp": "2025-01-24T16:00:00.000Z",
  "data": {
    "payment_id": "PAY-ZDKZODCQM",
    "transaction_id": "TRX1001",
    "amount": 1000.00,
    "currency": "SAR",
    "status": "completed"
  }
}
```

### 5. Signature Verification

```php
$signature = hash_hmac('sha256', $payload, $webhook_secret);
if ($signature !== $request->header('X-SafePay-Signature')) {
    abort(401, 'Invalid signature');
}
```

---

## ⚖️ Disputes API

### نظرة عامة

Disputes API يتيح إدارة النزاعات المتعلقة بمعاملات الضمان. يتضمن إنشاء النزاعات، الرد عليها، وحلها.

### 1. قائمة النزاعات

**Endpoint:** `GET /api/partner/disputes`

**الوصف:** الحصول على قائمة النزاعات مع إمكانية التصفية والترقيم.

**HTTP Method:** `GET`

**URL:** `/api/partner/disputes`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `status` | string | ❌ | حالة النزاع: `pending`, `open`, `resolved`, `closed` |
| `per_page` | integer | ❌ | عدد النتائج في الصفحة (افتراضي: 10) |
| `page` | integer | ❌ | رقم الصفحة (افتراضي: 1) |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 1,
        "escrow_id": 123,
        "reason": "منتج غير مطابق للمواصفات",
        "description": "المنتج المستلم لا يطابق المواصفات المذكورة",
        "status": "open",
        "created_at": "2025-01-24T10:00:00.000Z"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 10,
      "total": 25,
      "last_page": 3
    }
  }
}
```

---

### 2. إنشاء نزاع

**Endpoint:** `POST /api/partner/escrows/{escrowId}/disputes`

**الوصف:** رفع نزاع جديد على معاملة ضمان.

**HTTP Method:** `POST`

**URL:** `/api/partner/escrows/{escrowId}/disputes`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `escrowId` | integer | ✅ | معرف معاملة الضمان |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `reason` | string | ✅ | سبب النزاع (حد أقصى 500 حرف) |
| `description` | string | ✅ | وصف تفصيلي للنزاع |
| `attachments` | array | ❌ | مرفقات (ملفات) - حد أقصى 10 ميجابايت لكل ملف |

#### Request Example

```json
{
  "reason": "منتج غير مطابق للمواصفات",
  "description": "المنتج المستلم لا يطابق المواصفات المذكورة في الوصف. اللون مختلف والمواصفات غير صحيحة.",
  "attachments": []
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "message": "تم رفع النزاع بنجاح",
  "data": {
    "id": 1,
    "escrow_id": 123,
    "reason": "منتج غير مطابق للمواصفات",
    "description": "المنتج المستلم لا يطابق المواصفات المذكورة",
    "status": "pending",
    "evidence": [],
    "messages": [
      {
        "id": 1,
        "sender_id": 10,
        "sender_type": "partner",
        "message": "المنتج المستلم لا يطابق المواصفات المذكورة",
        "created_at": "2025-01-24T10:00:00.000Z"
      }
    ],
    "created_at": "2025-01-24T10:00:00.000Z"
  }
}
```

---

### 3. عرض تفاصيل نزاع

**Endpoint:** `GET /api/partner/disputes/{id}`

**الوصف:** الحصول على تفاصيل نزاع معين مع قواعد النزاع من EscrowConfig.

**HTTP Method:** `GET`

**URL:** `/api/partner/disputes/{id}`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف النزاع |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "id": 1,
    "escrow_id": 123,
    "reason": "منتج غير مطابق للمواصفات",
    "description": "المنتج المستلم لا يطابق المواصفات المذكورة",
    "status": "open",
    "deadline": "2025-01-31T10:00:00.000Z",
    "resolution_days": 7,
    "allow_evidence_upload": true,
    "max_evidence_items": 10,
    "arbitration_fee_percentage": 7.0,
    "auto_escalate": false,
    "evidence": [
      {
        "id": 1,
        "file_path": "disputes/1/evidence1.jpg",
        "file_name": "evidence1.jpg",
        "file_type": "image/jpeg",
        "file_size": 1024000
      }
    ],
    "messages": [
      {
        "id": 1,
        "sender": {
          "id": 10,
          "name": "أحمد محمد"
        },
        "message": "المنتج المستلم لا يطابق المواصفات المذكورة",
        "created_at": "2025-01-24T10:00:00.000Z"
      }
    ],
    "created_at": "2025-01-24T10:00:00.000Z"
  }
}
```

---

### 4. إضافة دليل/مرفق للنزاع

**Endpoint:** `POST /api/partner/disputes/{id}/evidence`

**الوصف:** إضافة دليل أو مرفق جديد للنزاع.

**HTTP Method:** `POST`

**URL:** `/api/partner/disputes/{id}/evidence`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف النزاع |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `description` | string | ✅ | وصف الدليل |
| `attachments` | array | ✅ | مرفقات (ملفات) - حد أقصى 10 ميجابايت لكل ملف |

#### Request Example

```json
{
  "description": "صورة المنتج المستلم يظهر الفرق في اللون والمواصفات",
  "attachments": []
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "تم إضافة الدليل بنجاح",
  "data": [
    {
      "id": 2,
      "dispute_id": 1,
      "file_path": "disputes/1/evidence2.jpg",
      "file_name": "evidence2.jpg",
      "file_type": "image/jpeg",
      "file_size": 2048000,
      "description": "صورة المنتج المستلم يظهر الفرق في اللون والمواصفات"
    }
  ]
}
```

---

### 5. الرد على نزاع

**Endpoint:** `POST /api/partner/disputes/{id}/respond`

**الوصف:** إضافة رد على نزاع موجود. يمكن للشريك أو المشتري أو البائع الرد على النزاع.

**HTTP Method:** `POST`

**URL:** `/api/partner/disputes/{id}/respond`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف النزاع |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `response` | string | ✅ | نص الرد (حد أقصى 1000 حرف) |

#### Request Example

```json
{
  "response": "نعتذر عن الخطأ. سنقوم بإرسال المنتج الصحيح فوراً أو استرجاع المبلغ كاملاً."
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "تم إرسال الرد بنجاح",
  "data": {
    "id": 1,
    "escrow_id": 123,
    "status": "open",
    "messages": [
      {
        "id": 1,
        "sender_id": 10,
        "sender_type": "partner",
        "message": "المنتج المستلم لا يطابق المواصفات المذكورة",
        "created_at": "2025-01-24T10:00:00.000Z"
      },
      {
        "id": 2,
        "sender_id": 11,
        "sender_type": "partner",
        "message": "نعتذر عن الخطأ. سنقوم بإرسال المنتج الصحيح فوراً أو استرجاع المبلغ كاملاً.",
        "created_at": "2025-01-24T11:00:00.000Z"
      }
    ],
    "evidence": []
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | OK - تم إرسال الرد بنجاح |
| `403` | Forbidden - ليس لديك صلاحية للرد على هذا النزاع |
| `404` | Not Found - النزاع غير موجود |
| `422` | Validation Error - أخطاء في التحقق |

---

### 6. حل نزاع (للشريك فقط)

**Endpoint:** `POST /api/partner/disputes/{id}/resolve`

**الوصف:** حل نزاع موجود. **فقط صاحب الضمان (الشريك)** يمكنه حل النزاع.

**HTTP Method:** `POST`

**URL:** `/api/partner/disputes/{id}/resolve`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف النزاع |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `resolution_type` | string | ✅ | نوع الحل: `refund_buyer`, `release_seller`, `partial_refund`, `full_refund` |
| `amount` | decimal | ❌ | المبلغ (مطلوب إذا كان `resolution_type` = `partial_refund`) |

#### Request Example (استرجاع كامل للمشتري)

```json
{
  "resolution_type": "full_refund"
}
```

#### Request Example (استرجاع جزئي)

```json
{
  "resolution_type": "partial_refund",
  "amount": 2500.00
}
```

#### Request Example (إفراج الأموال للبائع)

```json
{
  "resolution_type": "release_seller"
}
```

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "تم حل النزاع بنجاح",
  "data": {
    "dispute": {
      "id": 1,
      "escrow_id": 123,
      "status": "resolved",
      "resolution_type": "full_refund",
      "resolved_at": "2025-01-24T12:00:00.000Z"
    },
    "resolution": {
      "success": true,
      "message": "تم حل النزاع بنجاح",
      "refund_amount": 7500.00,
      "refund_status": "completed"
    }
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | OK - تم حل النزاع بنجاح |
| `403` | Forbidden - فقط صاحب الضمان يمكنه حل النزاع |
| `404` | Not Found - النزاع غير موجود |
| `422` | Validation Error - أخطاء في التحقق أو فشل في حل النزاع |

#### Notes / Constraints

- ✅ **فقط صاحب الضمان (الشريك)** يمكنه حل النزاع
- ✅ يتم إرسال إشعارات للطرفين عند حل النزاع
- ✅ يتم تسجيل العملية في Activity Log تلقائياً
- ✅ أنواع الحل المتاحة:
  - `refund_buyer`: استرجاع المبلغ للمشتري
  - `release_seller`: إفراج الأموال للبائع
  - `partial_refund`: استرجاع جزئي (يتطلب `amount`)
  - `full_refund`: استرجاع كامل

---

## 🎫 Support Tickets API

### نظرة عامة

Support Tickets API يتيح إدارة تذاكر الدعم الفني.

### 1. قائمة التذاكر

**Endpoint:** `GET /api/partner/tickets`

**الوصف:** الحصول على قائمة تذاكر الدعم.

**HTTP Method:** `GET`

**URL:** `/api/partner/tickets`

---

### 2. إنشاء تذكرة

**Endpoint:** `POST /api/partner/tickets`

**الوصف:** إنشاء تذكرة دعم جديدة.

**HTTP Method:** `POST`

**URL:** `/api/partner/tickets`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `subject` | string | ✅ | موضوع التذكرة |
| `category_id` | integer | ✅ | معرف الفئة |
| `message` | string | ✅ | رسالة التذكرة |
| `attachments` | array | ❌ | مرفقات |

---

### 3. الرد على تذكرة

**Endpoint:** `POST /api/partner/tickets/{id}/reply`

**الوصف:** إضافة رد على تذكرة موجودة.

**HTTP Method:** `POST`

**URL:** `/api/partner/tickets/{id}/reply`

---

## 👥 User Management API

### نظرة عامة

User Management API يتيح إدارة المستخدمين الفرعيين للشريك. يتيح إنشاء وتحديث وإدارة المستخدمين الفرعيين مع إدارة الأدوار والصلاحيات.

### مخطط تدفق إدارة المستخدمين

```mermaid
sequenceDiagram
    participant Partner as 🔌 الشريك
    participant API as 🏦 SafePay API
    participant Validation as ✅ التحقق
    participant Account as 💳 نظام الحسابات
    participant DB as 🗄️ قاعدة البيانات

    Partner->>API: POST /api/partner/users
    API->>Validation: التحقق من البيانات
    Validation->>Validation: التحقق من البريد الإلكتروني (فريد)
    Validation->>Validation: التحقق من قوة كلمة المرور
    Validation->>Validation: التحقق من وجود الأدوار
    
    alt ❌ بيانات غير صحيحة
        Validation-->>API: أخطاء التحقق
        API-->>Partner: 422 Validation Error
    else ✅ بيانات صحيحة
        API->>DB: إنشاء سجل المستخدم (status: active)
        API->>DB: ربط المستخدم بالأدوار (user_roles)
        
        alt ❌ فشل إنشاء الحساب
            Note over Account: trigger يمنع إنشاء حساب لمستخدم غير نشط
            API->>DB: Rollback
            API-->>Partner: 500 Error
        else ✅ نجح الإنشاء
            API->>Account: إنشاء حساب المستخدم
            Account->>DB: إنشاء سجل الحساب
            API->>DB: Commit Transaction
            API-->>Partner: 201 User Created
        end
    end
```

### مخطط العلاقات في نظام المستخدمين

```mermaid
erDiagram
    PARTNER ||--o{ SUB_USER : يمتلك
    SUB_USER ||--o{ USER_ROLE : له_أدوار
    USER_ROLE }o--|| ROLE : مرتبط_ب
    ROLE ||--o{ ROLE_PERMISSION : له_صلاحيات
    SUB_USER ||--|| ACCOUNT : له_حساب
    
    PARTNER {
        int id PK
        string email
        string company_name
        int status
    }
    
    SUB_USER {
        int id PK
        int partner_id FK
        string email
        string firstname
        string lastname
        int status
    }
    
    USER_ROLE {
        int user_id FK
        int role_id FK
    }
    
    ROLE {
        int id PK
        string name
        string description
        int partner_id FK
    }
    
    ACCOUNT {
        int id PK
        int user_id FK
        string account_number
        decimal balance
    }
```

### 1. قائمة المستخدمين

**Endpoint:** `GET /api/partner/users`

**الوصف:** الحصول على قائمة المستخدمين الفرعيين للشريك مع إمكانية التصفية والترقيم.

**HTTP Method:** `GET`

**URL:** `/api/partner/users`

#### Parameters

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `search` | string | ❌ | البحث في الاسم، البريد الإلكتروني، أو اسم المستخدم |
| `status` | string | ❌ | حالة المستخدم: `active`, `inactive` |
| `role_id` | integer | ❌ | تصفية حسب الدور |
| `per_page` | integer | ❌ | عدد النتائج في الصفحة (افتراضي: 10) |
| `page` | integer | ❌ | رقم الصفحة (افتراضي: 1) |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 10,
        "firstname": "أحمد",
        "lastname": "محمد",
        "username": "ahmed_mohamed_1234567890",
        "email": "ahmed@example.com",
        "status": 1,
        "roles": [
          {
            "id": 5,
            "name": "مدير المبيعات",
            "description": "صلاحيات إدارة المبيعات"
          }
        ],
        "created_at": "2025-01-24T10:00:00.000Z"
      }
    ],
    "meta": {
      "total_users": 25,
      "active_users": 20
    },
    "pagination": {
      "current_page": 1,
      "per_page": 10,
      "total": 25,
      "last_page": 3
    }
  }
}
```

---

### 2. إنشاء مستخدم

**Endpoint:** `POST /api/partner/users`

**الوصف:** إنشاء مستخدم فرعي جديد مع تعيين الأدوار.

**HTTP Method:** `POST`

**URL:** `/api/partner/users`

#### Parameters

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | ✅ | الاسم الكامل للمستخدم |
| `email` | email | ✅ | البريد الإلكتروني (يجب أن يكون فريداً) |
| `password` | string | ✅ | كلمة المرور (8 أحرف على الأقل، أحرف كبيرة وصغيرة وأرقام) |
| `role_ids` | array | ✅ | قائمة معرفات الأدوار (يجب أن يحتوي على دور واحد على الأقل) |
| `phone` | string | ❌ | رقم الهاتف |
| `position` | string | ❌ | المنصب |
| `status` | string | ✅ | حالة المستخدم: `active`, `inactive` |

#### Request Example

```json
{
  "name": "أحمد محمد",
  "email": "ahmed@example.com",
  "password": "SecurePass123",
  "role_ids": [5, 6],
  "phone": "+966500000000",
  "position": "مدير المبيعات",
  "status": "active"
}
```

#### Response Example (Success - 201 Created)

```json
{
  "success": true,
  "message": "تم إنشاء المستخدم الفرعي بنجاح",
  "data": {
    "id": 10,
    "firstname": "أحمد",
    "lastname": "محمد",
    "username": "ahmed_mohamed_1234567890",
    "email": "ahmed@example.com",
    "status": 1,
    "name": "أحمد محمد",
    "roles": [
      {
        "id": 5,
        "name": "مدير المبيعات",
        "description": "صلاحيات إدارة المبيعات"
      },
      {
        "id": 6,
        "name": "مشرف الدعم",
        "description": "صلاحيات إدارة الدعم الفني"
      }
    ],
    "permissions": [],
    "created_at": "2025-01-24T10:00:00.000Z"
  }
}
```

#### Status Codes

| Code | Description |
|------|-------------|
| `201` | Created - تم إنشاء المستخدم بنجاح |
| `400` | Bad Request - بيانات غير صحيحة |
| `422` | Validation Error - أخطاء في التحقق (مثل: البريد الإلكتروني مستخدم مسبقاً) |
| `500` | Internal Server Error - خطأ في الخادم |

---

### 3. الحصول على تفاصيل مستخدم

**Endpoint:** `GET /api/partner/users/{id}`

**الوصف:** الحصول على تفاصيل مستخدم فرعي معين.

**HTTP Method:** `GET`

**URL:** `/api/partner/users/{id}`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف المستخدم |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "data": {
    "id": 10,
    "firstname": "أحمد",
    "lastname": "محمد",
    "username": "ahmed_mohamed_1234567890",
    "email": "ahmed@example.com",
    "status": 1,
    "name": "أحمد محمد",
    "roles": [
      {
        "id": 5,
        "name": "مدير المبيعات",
        "description": "صلاحيات إدارة المبيعات"
      }
    ],
    "permissions": [],
    "created_at": "2025-01-24T10:00:00.000Z"
  }
}
```

---

### 4. تحديث مستخدم

**Endpoint:** `PUT /api/partner/users/{id}`

**الوصف:** تحديث بيانات مستخدم فرعي موجود.

**HTTP Method:** `PUT`

**URL:** `/api/partner/users/{id}`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف المستخدم |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | ✅ | الاسم الكامل للمستخدم |
| `email` | email | ✅ | البريد الإلكتروني |
| `password` | string | ❌ | كلمة المرور الجديدة (اختياري) |
| `role_ids` | array | ✅ | قائمة معرفات الأدوار |
| `phone` | string | ❌ | رقم الهاتف |
| `position` | string | ❌ | المنصب |
| `status` | string | ✅ | حالة المستخدم: `active`, `inactive` |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "تم تحديث بيانات المستخدم بنجاح",
  "data": {
    "id": 10,
    "firstname": "أحمد",
    "lastname": "محمد",
    "username": "ahmed_mohamed_1234567890",
    "email": "ahmed@example.com",
    "status": 1,
    "name": "أحمد محمد",
    "roles": [
      {
        "id": 5,
        "name": "مدير المبيعات",
        "description": "صلاحيات إدارة المبيعات"
      }
    ],
    "permissions": [],
    "updated_at": "2025-01-24T11:00:00.000Z"
  }
}
```

---

### 5. تحديث حالة المستخدم

**Endpoint:** `POST /api/partner/users/{id}/toggle-status`

**الوصف:** تفعيل أو تعطيل مستخدم فرعي.

**HTTP Method:** `POST`

**URL:** `/api/partner/users/{id}/toggle-status`

#### Parameters

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | ✅ | معرف المستخدم |

#### Response Example (Success - 200 OK)

```json
{
  "success": true,
  "message": "تم تحديث حالة المستخدم بنجاح",
  "data": {
    "id": 10,
    "status": 0,
    "status_text": "inactive"
  }
}
```

#### Notes / Constraints

- ✅ المستخدمون يتم إنشاؤهم بحالة `active` دائماً لتجنب مشاكل قاعدة البيانات
- ✅ يمكن تحديث الحالة إلى `inactive` بعد الإنشاء
- ✅ هيكل `roles` في Response يحتوي على `id`, `name`, `description` لكل دور
- ✅ يمكن للمستخدم أن يكون له عدة أدوار (`role_ids` هو array)

---

## ⚠️ معالجة الأخطاء

### Error Response Format

جميع الأخطاء تُرجع بنفس التنسيق:

```json
{
  "success": false,
  "message": "Error message",
  "error": "Detailed error description",
  "errors": {
    "field_name": ["Error message 1", "Error message 2"]
  }
}
```

### Error Codes

| Code | Description | الحل |
|------|-------------|------|
| `400` | Bad Request | التحقق من صحة البيانات المرسلة |
| `401` | Unauthorized | التحقق من صحة API Key |
| `403` | Forbidden | التحقق من الصلاحيات |
| `404` | Not Found | التحقق من وجود المورد |
| `422` | Validation Error | التحقق من البيانات المطلوبة |
| `429` | Too Many Requests | تقليل عدد الطلبات |
| `500` | Internal Server Error | التواصل مع الدعم |

### نماذج الأخطاء العامة

#### 1. أخطاء التوثيق (401)

```json
{
  "success": false,
  "message": "Authentication required",
  "error": "API Key is missing or invalid"
}
```

#### 2. أخطاء المدخلات (422)

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": {
    "amount": ["The amount must be at least 0.01."],
    "customer_email": ["The customer email field is required."]
  }
}
```

#### 3. أخطاء 404 (Not Found)

```json
{
  "success": false,
  "message": "Resource not found",
  "error": "Payment with ID PAY-ZDKZODCQM not found"
}
```

#### 4. أخطاء 500 (Internal Server Error)

```json
{
  "success": false,
  "message": "Internal server error",
  "error": "An unexpected error occurred. Please try again later."
}
```

---

## 💻 نماذج الكود

### cURL

#### مثال 1: بدء عملية دفع مباشر

```bash
curl -X POST https://api.safepay.com/api/partner/payments/initiate \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "currency": "SAR",
    "payment_method": "wallet",
    "customer_email": "customer@example.com",
    "description": "دفع منتج"
  }'
```

#### مثال 2: إنشاء ضمان

```bash
curl -X POST https://api.safepay.com/api/partner/escrows \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "شراء لابتوب",
    "description": "شراء لابتوب gaming",
    "amount": 7500.00,
    "currency": "SAR",
    "buyer_email": "buyer@example.com",
    "seller_email": "seller@example.com",
    "category_id": 1
  }'
```

#### مثال 3: استعلام حالة الدفع

```bash
curl -X GET https://api.safepay.com/api/partner/payments/PAY-ZDKZODCQM \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

### JavaScript (Fetch API)

#### مثال 1: بدء عملية دفع مباشر

```javascript
async function initiatePayment(amount, currency, customerEmail) {
  try {
    const response = await fetch('https://api.safepay.com/api/partner/payments/initiate', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.SAFEPAY_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        amount: amount,
        currency: currency,
        payment_method: 'wallet',
        customer_email: customerEmail,
        description: 'دفع منتج'
      })
    });

    const data = await response.json();
    
    if (data.success) {
      console.log('Payment ID:', data.data.payment_id);
      return data.data;
    } else {
      throw new Error(data.message);
    }
  } catch (error) {
    console.error('Payment error:', error);
    throw error;
  }
}

// الاستخدام
initiatePayment(1000.00, 'SAR', 'customer@example.com')
  .then(payment => {
    console.log('Payment initiated:', payment);
  })
  .catch(error => {
    console.error('Error:', error);
  });
```

#### مثال 2: إنشاء ضمان مع دفع

```javascript
async function createEscrowWithPayment(escrowData) {
  try {
    const response = await fetch('https://api.safepay.com/api/partner/escrows/create-with-payment', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.SAFEPAY_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        title: escrowData.title,
        description: escrowData.description,
        amount: escrowData.amount,
        currency: 'SAR',
        buyer_email: escrowData.buyerEmail,
        seller_email: escrowData.sellerEmail,
        category_id: 1,
        type: 2,
        payment_method: 'wallet',
        payment_data: {},
        buyer_data: {
          firstname: escrowData.buyerFirstname,
          lastname: escrowData.buyerLastname,
          phone: escrowData.buyerPhone
        }
      })
    });

    const data = await response.json();
    
    if (data.success) {
      return data.data;
    } else {
      throw new Error(data.message);
    }
  } catch (error) {
    console.error('Escrow creation error:', error);
    throw error;
  }
}
```

#### مثال 3: استعلام حالة الدفع

```javascript
async function getPaymentStatus(paymentId) {
  try {
    const response = await fetch(`https://api.safepay.com/api/partner/payments/${paymentId}`, {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${process.env.SAFEPAY_API_KEY}`
      }
    });

    const data = await response.json();
    
    if (data.success) {
      return data.data;
    } else {
      throw new Error(data.message);
    }
  } catch (error) {
    console.error('Error fetching payment status:', error);
    throw error;
  }
}
```

---

### PHP

#### مثال 1: بدء عملية دفع مباشر

```php
<?php

function initiatePayment($amount, $currency, $customerEmail) {
    $apiKey = getenv('SAFEPAY_API_KEY');
    $url = 'https://api.safepay.com/api/partner/payments/initiate';
    
    $data = [
        'amount' => $amount,
        'currency' => $currency,
        'payment_method' => 'wallet',
        'customer_email' => $customerEmail,
        'description' => 'دفع منتج'
    ];
    
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiKey,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode === 201) {
        $result = json_decode($response, true);
        if ($result['success']) {
            return $result['data'];
        }
    }
    
    throw new Exception('Failed to initiate payment');
}

// الاستخدام
try {
    $payment = initiatePayment(1000.00, 'SAR', 'customer@example.com');
    echo "Payment ID: " . $payment['payment_id'] . "\n";
} catch (Exception $e) {
    echo "Error: " . $e->getMessage() . "\n";
}
```

#### مثال 2: إنشاء ضمان

```php
<?php

function createEscrow($title, $description, $amount, $buyerEmail, $sellerEmail) {
    $apiKey = getenv('SAFEPAY_API_KEY');
    $url = 'https://api.safepay.com/api/partner/escrows';
    
    $data = [
        'title' => $title,
        'description' => $description,
        'amount' => $amount,
        'currency' => 'SAR',
        'buyer_email' => $buyerEmail,
        'seller_email' => $sellerEmail,
        'category_id' => 1,
        'type' => 2
    ];
    
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiKey,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode === 201) {
        $result = json_decode($response, true);
        if ($result['success']) {
            return $result['data'];
        }
    }
    
    throw new Exception('Failed to create escrow');
}
```

#### مثال 3: استرجاع المبلغ

```php
<?php

function refundPayment($paymentId, $amount = null, $reason = null) {
    $apiKey = getenv('SAFEPAY_API_KEY');
    $url = 'https://api.safepay.com/api/partner/payments/refund';
    
    $data = [
        'payment_id' => $paymentId
    ];
    
    if ($amount !== null) {
        $data['amount'] = $amount;
    }
    
    if ($reason !== null) {
        $data['reason'] = $reason;
    }
    
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiKey,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode === 200) {
        $result = json_decode($response, true);
        if ($result['success']) {
            return $result['data'];
        }
    }
    
    throw new Exception('Failed to refund payment');
}
```

---

### Python

#### مثال 1: بدء عملية دفع مباشر

```python
import requests
import os

def initiate_payment(amount, currency, customer_email):
    api_key = os.getenv('SAFEPAY_API_KEY')
    url = 'https://api.safepay.com/api/partner/payments/initiate'
    
    headers = {
        'Authorization': f'Bearer {api_key}',
        'Content-Type': 'application/json'
    }
    
    data = {
        'amount': amount,
        'currency': currency,
        'payment_method': 'wallet',
        'customer_email': customer_email,
        'description': 'دفع منتج'
    }
    
    response = requests.post(url, json=data, headers=headers)
    
    if response.status_code == 201:
        result = response.json()
        if result['success']:
            return result['data']
    
    raise Exception('Failed to initiate payment')

# الاستخدام
try:
    payment = initiate_payment(1000.00, 'SAR', 'customer@example.com')
    print(f"Payment ID: {payment['payment_id']}")
except Exception as e:
    print(f"Error: {e}")
```

#### مثال 2: إنشاء ضمان مع دفع

```python
import requests
import os

def create_escrow_with_payment(escrow_data):
    api_key = os.getenv('SAFEPAY_API_KEY')
    url = 'https://api.safepay.com/api/partner/escrows/create-with-payment'
    
    headers = {
        'Authorization': f'Bearer {api_key}',
        'Content-Type': 'application/json'
    }
    
    data = {
        'title': escrow_data['title'],
        'description': escrow_data['description'],
        'amount': escrow_data['amount'],
        'currency': 'SAR',
        'buyer_email': escrow_data['buyer_email'],
        'seller_email': escrow_data['seller_email'],
        'category_id': 1,
        'type': 2,
        'payment_method': 'wallet',
        'payment_data': {},
        'buyer_data': {
            'firstname': escrow_data['buyer_firstname'],
            'lastname': escrow_data['buyer_lastname'],
            'phone': escrow_data['buyer_phone']
        }
    }
    
    response = requests.post(url, json=data, headers=headers)
    
    if response.status_code == 201:
        result = response.json()
        if result['success']:
            return result['data']
    
    raise Exception('Failed to create escrow with payment')
```

#### مثال 3: استعلام حالة الدفع

```python
import requests
import os

def get_payment_status(payment_id):
    api_key = os.getenv('SAFEPAY_API_KEY')
    url = f'https://api.safepay.com/api/partner/payments/{payment_id}'
    
    headers = {
        'Authorization': f'Bearer {api_key}'
    }
    
    response = requests.get(url, headers=headers)
    
    if response.status_code == 200:
        result = response.json()
        if result['success']:
            return result['data']
    
    raise Exception('Failed to get payment status')
```

---

## ✅ أفضل الممارسات

### 1. إدارة API Keys

- ✅ **استخدام Environment Variables** - لا تخزن API Keys في الكود
- ✅ **تحديث المفاتيح بانتظام** - تغيير API Keys كل 90 يوم
- ✅ **استخدام مفاتيح مختلفة** - Production و Test مفاتيح منفصلة
- ✅ **تحديد IP Whitelist** - تقييد الوصول من IPs محددة

### 2. معالجة الأخطاء

- ✅ **معالجة جميع الأخطاء** - لا تتجاهل أي استجابة خطأ
- ✅ **Retry Logic** - إعادة المحاولة عند فشل الطلبات
- ✅ **Logging** - تسجيل جميع الطلبات والأخطاء
- ✅ **Error Handling** - معالجة أخطاء الشبكة والوقت

### 3. الأمان

- ✅ **HTTPS فقط** - لا تستخدم HTTP أبداً
- ✅ **Signature Verification** - التحقق من توقيع Webhooks
- ✅ **Rate Limiting** - احترام حدود الطلبات
- ✅ **Input Validation** - التحقق من جميع المدخلات

### 4. الأداء

- ✅ **Caching** - تخزين البيانات المؤقتة عند الحاجة
- ✅ **Pagination** - استخدام الترقيم للقوائم الكبيرة
- ✅ **Async Processing** - معالجة Webhooks بشكل غير متزامن
- ✅ **Connection Pooling** - إعادة استخدام الاتصالات

### 5. الاختبار

- ✅ **Sandbox Environment** - اختبار في بيئة Sandbox أولاً
- ✅ **Unit Tests** - كتابة اختبارات للوحدات
- ✅ **Integration Tests** - اختبار التكامل الكامل
- ✅ **Error Scenarios** - اختبار سيناريوهات الأخطاء

---

## 📞 الدعم والمساعدة

### موارد الدعم

- **البريد الإلكتروني:** support@safepay.com
- **الوثائق:** https://docs.safepay.com
- **Status Page:** https://status.safepay.com
- **API Status:** https://api.safepay.com/status

### الإبلاغ عن المشاكل

عند الإبلاغ عن مشكلة، يرجى تضمين:
- API Key (Test Key فقط)
- Request/Response الكاملة
- Timestamp
- Error Messages

---

**تم إعداد المستند بواسطة:** SafePay Development Team  
**التاريخ:** 2025-01-24  
**الإصدار:** 2.0.0

</div>


