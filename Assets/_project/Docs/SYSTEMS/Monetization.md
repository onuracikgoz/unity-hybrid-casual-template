# Monetization System

## 1. Amaç

`MonetizationSystem`, oyunun para kazanma özelliklerinin teknik entegrasyonunu ve yaşam döngüsünü yönetir.

Bu sistemin sorumlulukları:

* IAP ürünlerinin başlatılması ve satın alma yaşam döngüsünün yönetilmesi
* Rewarded Ad yaşam döngüsünün yönetilmesi
* Interstitial Ad yaşam döngüsünün yönetilmesi
* Store ve reklam SDK entegrasyonlarının yönetilmesi
* Ürün kimlikleri ile uygulama içi ürün tanımlarının eşleştirilmesi
* Satın alma durumlarının yönetilmesi
* Satın alma doğrulama ve transaction recovery akışının yürütülmesi
* Reklam availability, load ve show durumlarının yönetilmesi
* Monetization ile ilgili domain event'lerinin yayınlanması

`MonetizationSystem`:

* UI değildir.
* Shop değildir.
* EconomySystem değildir.
* RewardSystem değildir.
* ProgressionSystem değildir.
* Analytics SDK'sı değildir.
* Store veya reklam SDK'sının tüm API'sini oyunun geri kalanına açan bir proxy olmamalıdır.

Temel prensip:

> MonetizationSystem para kazanma işlemini yürütür; ödülün kendisine ait state'i ilgili domain system'lerine bırakır.

---

# 2. Temel Mimari

Genel yapı:

```text
                Monetization Configuration
                         ↓
                 MonetizationSystem
                  ↙             ↘
              IAP SDK         Ad SDK
                 ↓               ↓
         Purchase Result      Ad Result
                 ↓               ↓
             RewardSystem    RewardSystem
                 ↓
        ┌────────┴─────────┐
        ↓                  ↓
 EconomySystem      ProgressionSystem
```

UI tarafında:

```text
ShopView
    ↓
Purchase Intent
    ↓
MonetizationSystem
    ↓
Store SDK
    ↓
Purchase Result
    ↓
RewardSystem
    ↓
Relevant Domain System
```

Rewarded Ad:

```text
Feature / UI
    ↓
Rewarded Ad Request
    ↓
MonetizationSystem
    ↓
Ad SDK
    ↓
Verified Reward
    ↓
RewardSystem
    ↓
Economy / Progression / Feature
```

Interstitial:

```text
Gameplay / GameFlow / Ad Policy
            ↓
      Show Interstitial
            ↓
      MonetizationSystem
            ↓
          Ad SDK
            ↓
       Ad Result
```

Interstitial reklamın kendi başına bir gameplay reward'ı yoktur.

---

# 3. Sorumluluk Sınırı

| Sistem                             | Sorumluluk                                            |
| ---------------------------------- | ----------------------------------------------------- |
| `MonetizationSystem`               | IAP ve reklam entegrasyonu                            |
| `ShopView`                         | Ürünleri ve satın alma durumunu gösterme              |
| `ShopSystem`                       | Shop tekliflerinin/availability kurallarının yönetimi |
| `RewardSystem`                     | Ödülün uygulanmasını orkestre etme                    |
| `EconomySystem`                    | Currency state ve transaction                         |
| `ProgressionSystem`                | Progression state ve unlock                           |
| `SaveSystem`                       | Kalıcı veri                                           |
| `AnalyticsSystem`                  | Analytics SDK entegrasyonu                            |
| `GameFlow`                         | Global oyun akışı                                     |
| `AdPolicy` / ilgili gameplay owner | Reklamın ne zaman gösterileceğine karar verme         |

Bu sınırlar korunmalıdır.

Örneğin:

```text
ShopView → EconomySystem.TrySpend()
```

yanlıştır.

```text
ShopView → Store SDK
```

yanlıştır.

```text
ShopView → MonetizationSystem.Purchase(productId)
```

doğrudur.

---

# 4. IAP, Reklam ve Reward Ayrımı

Monetization üç farklı problemi kapsar:

```text
IAP
├── Product
├── Purchase
├── Verification
├── Entitlement
└── Transaction recovery

Rewarded Ad
├── Placement
├── Availability
├── Show
├── Reward earned
└── Reward delivery

Interstitial Ad
├── Placement
├── Availability
├── Frequency policy
├── Show
└── Completion / close
```

Bu üç akış aynı sistem altında bulunabilir ancak aynı domain olarak ele alınmamalıdır.

---

# 5. Product Definition

Ürünlerin oyundaki anlamı configuration üzerinden tanımlanabilir.

Örneğin:

```text
ProductDefinition
├── Logical Product Id
├── Product Type
├── Platform Product Id
├── Reward Definition
└── Entitlement Information
```

Örnek:

```text
ProductDefinition
    logicalId = "product_001"
    type = Consumable
    reward = RewardDefinition
```

Buradaki ID oyunun kendi logical ID'sidir.

Platform store ID'si ayrı tutulabilir:

```text
Logical Product Id
        ↓
Platform Product Id
        ↓
Store SDK
```

Platforma göre farklı product ID gerekiyorsa mapping configuration üzerinden yapılmalıdır.

---

# 6. Store Product Metadata

Oyunun product definition'ı ile store'dan gelen ürün bilgisi aynı şey değildir.

Oyunun configuration'ı:

```text
ProductDefinition
├── logicalId
├── type
├── reward
└── platformProductIds
```

Store runtime bilgisi:

```text
StoreProductInfo
├── localizedTitle
├── localizedDescription
├── localizedPrice
├── currencyCode
└── storeProductId
```

Özellikle gerçek para fiyatları UI içerisinde hard-code edilmemelidir.

Tercih edilen akış:

```text
Store
 ↓
StoreProductInfo
 ↓
MonetizationSystem
 ↓
ShopSystem / ShopView
```

Böylece store'un bölgesel fiyatlandırması ve localization bilgisi korunur.

---

# 7. Monetization Configuration

`MonetizationConfig` aşağıdaki gibi statik ayarları içerebilir:

```text
MonetizationConfig
├── Product Definitions
├── Ad Placements
├── Initialization Settings
└── Platform Settings
```

Configuration:

* SDK key gibi client-safe public configuration içerebilir.
* Product ID mapping içerebilir.
* Ad placement ID'leri içerebilir.
* Environment bilgisi içerebilir.

Secret:

* API secret
* private server credential
* signing secret
* güvenlik açısından hassas token

gibi bilgiler client-side ScriptableObject içerisinde tutulmamalıdır.

---

# 8. Runtime State

ScriptableObject configuration mutable runtime state'in sahibi değildir.

Örneğin:

```text
MonetizationConfig
```

ürün tanımını sağlar.

Ancak:

```text
PurchaseState
AdState
StoreReady
PendingTransaction
```

gibi runtime bilgiler `MonetizationSystem` tarafından yönetilir.

---

# 9. IAP Purchase Flow

Temel satın alma akışı:

```text
User
 ↓
ShopView
 ↓
Purchase Intent
 ↓
MonetizationSystem
 ↓
Store SDK
 ↓
Purchase Result
 ↓
Validation / Resolution
 ↓
RewardSystem
 ↓
Domain Systems
 ↓
SaveSystem
```

UI satın alma işleminin sonucunu bekleyebilir ancak transaction'ın sahibi değildir.

---

# 10. Purchase State

Satın alma yaşam döngüsü açıkça modellenmelidir.

Örneğin:

```text
Idle
Pending
Success
Failed
Cancelled
Unavailable
```

Gerekirse store'a bağlı daha ayrıntılı durumlar ayrıca modellenebilir.

State'in amacı UI animasyonu üretmek değil, transaction yaşam döngüsünü doğru yönetmektir.

---

# 11. Pending Purchase

Satın alma her zaman anında sonuçlanmayabilir.

Örneğin:

```text
Purchase Request
      ↓
Pending
      ↓
Application Closed
      ↓
Application Restarted
      ↓
Transaction Recovery
```

Bu nedenle pending transaction'lar kaybolmamalıdır.

Bootstrap sırasında monetization initialization tamamlanırken daha önce başlatılmış veya bekleyen transaction'lar uygun şekilde tekrar ele alınmalıdır.

---

# 12. Duplicate Purchase Request

Aynı ürün için aynı anda birden fazla purchase request gönderilmesi engellenmelidir.

Örneğin:

```text
User Tap
User Tap
User Tap
```

üç ayrı transaction başlatmamalıdır.

UI tarafında button disable yapılabilir.

Ancak asıl güvenlik:

```text
MonetizationSystem
```

tarafında bulunmalıdır.

UI protection yalnızca UX katmanıdır.

---

# 13. Idempotency

Purchase ve reward delivery idempotent olmalıdır.

Örneğin uygulama şu sırada kapanabilir:

```text
Purchase Success
      ↓
Reward Grant
      ↓
Application Crash
```

Bir sonraki açılışta transaction tekrar işlendiğinde aynı reward'ın ikinci kez verilmesi engellenmelidir.

Bunun için transaction/product identifier gibi platformun sağladığı benzersiz bilgiler uygun bir şekilde takip edilmelidir.

---

# 14. Entitlement

Entitlement, oyuncunun bir ürüne sahip olma durumunu ifade eder.

Örneğin:

```text
Purchase
   ↓
Entitlement
```

ile:

```text
Purchase
   ↓
Consumable Reward
```

aynı şey değildir.

Özellikle:

* Non-consumable
* Subscription

gibi ürünlerde entitlement önemli hale gelir.

Örneğin:

```text
Player owns product X
```

bir entitlement state'idir.

Buna karşılık:

```text
Product X → 100 currency
```

bir consumable reward olabilir.

Bu iki kavram birbirine karıştırılmamalıdır.

---

# 15. Consumable Ürünler

Consumable ürünlerde satın alma sonucu bir reward üretilebilir.

Örneğin:

```text
Purchase
 ↓
RewardDefinition
 ↓
RewardSystem
 ↓
EconomySystem
```

Reward'ın birden fazla kez uygulanmaması için transaction idempotency mekanizması bulunmalıdır.

---

# 16. Non-Consumable Ürünler

Non-consumable ürünlerde temel sonuç çoğunlukla entitlement'tır.

Örneğin:

```text
Purchase
 ↓
Entitlement Granted
 ↓
Relevant System
```

Bu ürünlerin restore edilmesi desteklenebilir.

UI entitlement'ın kendisini hesaplamamalıdır.

---

# 17. Subscription

Subscription desteği template için zorunlu değildir.

Gerektiğinde:

```text
Subscription
    ↓
Store
    ↓
Entitlement
    ↓
Relevant System
```

şeklinde mevcut monetization mimarisine eklenebilir.

Subscription yalnızca ihtiyaç olmadığı halde ilk sürüme eklenmemelidir.

---

# 18. Restore Purchases

Restore işlemi özellikle non-consumable ve subscription ürünlerde gerekli olabilir.

Akış:

```text
User
 ↓
Restore Request
 ↓
MonetizationSystem
 ↓
Store SDK
 ↓
Restored Transactions
 ↓
Validation / Resolution
 ↓
Entitlement
```

Restore işlemi yeni bir satın alma olarak yorumlanmamalıdır.

---

# 19. Purchase Validation

Store SDK'nın client tarafındaki "success" sonucu her güvenlik senaryosunda yeterli değildir.

Özellikle yüksek değerli veya server-authoritative ürünlerde:

```text
Client
 ↓
Purchase
 ↓
Receipt / Transaction Data
 ↓
Server Validation
 ↓
Validated Result
```

gibi bir yapı gerekebilir.

İlk sürümde server validation bulunmuyorsa mimari bunu ileride eklenebilir şekilde bırakmalıdır.

Client-side receipt parsing güvenli bir server authority yerine geçmez.

---

# 20. Acknowledge / Consume

Store platformları transaction'ın acknowledge veya consume edilmesini gerektirebilir.

Bu işlemler `MonetizationSystem` sorumluluğundadır.

Ancak exact ordering platform sözleşmesine göre belirlenmelidir.

Genel prensip:

> Transaction geri döndürülemez şekilde tamamlanmadan önce reward delivery'nin crash durumunda kurtarılabilir olması gerekir.

Platformun gerektirdiği acknowledge/consume adımları bu recovery stratejisiyle birlikte tasarlanmalıdır.

---

# 21. Reward Delivery

MonetizationSystem reward'ın domain state'ini doğrudan değiştirmemelidir.

Yanlış:

```text
MonetizationSystem
    ↓
EconomySystem.AddCurrency()
```

Tercih edilen:

```text
MonetizationSystem
    ↓
RewardSystem
    ↓
EconomySystem
```

Böylece monetization ile reward composition birbirinden ayrılır.

---

# 22. Rewarded Ads

Rewarded ad akışı:

```text
Feature
 ↓
Request Rewarded Ad
 ↓
MonetizationSystem
 ↓
Ad SDK
 ↓
Ad Started
 ↓
Ad Completed
 ↓
Reward Earned
 ↓
RewardSystem
```

Ödül yalnızca SDK/provider tarafından doğrulanmış reward callback üzerinden verilmelidir.

Şu yaklaşım yanlıştır:

```text
ShowAd()
    ↓
Immediately GiveReward()
```

---

# 23. Rewarded Ad Result

Rewarded ad sonucu aşağıdaki durumları ayırt edebilmelidir:

```text
Unavailable
Loading
Showing
Completed
RewardEarned
ClosedWithoutReward
Failed
```

Provider'ın callback modeline göre daha basit veya ayrıntılı bir state machine kullanılabilir.

Ancak:

```text
AdStarted
```

ile:

```text
RewardEarned
```

aynı kabul edilmemelidir.

---

# 24. Rewarded Ad Reward Idempotency

Reward callback'in birden fazla kez gelmesi veya transaction lifecycle'ın beklenmeyen şekilde tekrar işlenmesi durumunda aynı reward ikinci kez uygulanmamalıdır.

Reward delivery için uygun bir request/transaction identifier kullanılabilir.

RewardSystem da kritik reward işlemlerinde duplicate grant'e karşı dayanıklı olmalıdır.

---

# 25. Interstitial Ads

Interstitial reklam oyuncuya doğrudan reward vermez.

Akış:

```text
Gameplay / GameFlow / Ad Policy
            ↓
    Request Interstitial
            ↓
     MonetizationSystem
            ↓
          Ad SDK
```

MonetizationSystem:

* Ad'ın gösterilebilir olup olmadığını bildirir.
* Load/show lifecycle'ını yönetir.
* SDK callback'lerini normalize eder.
* Hataları yönetir.

Ancak:

> Interstitial'ın ne zaman gösterileceği her zaman MonetizationSystem'in işi değildir.

Örneğin level completion sonrası reklam gösterme kararı gameplay veya flow policy tarafından verilebilir.

---

# 26. Ad Frequency Policy

Interstitial gibi reklamların sıklığı için gerekirse ayrı bir policy/configuration kullanılabilir.

Örneğin:

```text
AdPolicyConfig
├── Cooldown
├── Frequency Limit
├── Minimum Session Duration
└── Placement Rules
```

Bu kuralların gerçek ihtiyaç olmadan karmaşık hale getirilmesi gerekmez.

MonetizationSystem reklamı gösterebilir; ilgili feature veya policy reklamın uygun olup olmadığına karar verebilir.

---

# 27. Ad Preloading

Interstitial ve rewarded reklamlar için preload yapılabilir.

Örneğin:

```text
Initialize
    ↓
Load Ad
    ↓
Ready
    ↓
Show
    ↓
Reload
```

Preload zorunlu değildir.

Memory, provider davranışı ve ürün gereksinimine göre karar verilmelidir.

---

# 28. Ad Availability

Ad availability runtime durumudur.

Örneğin:

```text
IsRewardedAvailable()
IsInterstitialAvailable()
```

gibi query API'leri bulunabilir.

Ancak availability yalnızca anlık bir bilgidir.

UI:

```text
Available
```

görse bile show anında ad unavailable olabilir.

Bu nedenle `Show` sonucu her zaman güvenilir şekilde ele alınmalıdır.

---

# 29. Ad Failure

Aşağıdaki durumlar normal olarak ele alınmalıdır:

* No fill
* Network failure
* SDK failure
* Timeout
* Ad expired
* Provider unavailable
* User closed ad
* Reward verilmeden kapanma

Reklamın başarısız olması oyunun ana gameplay state'ini bozmamalıdır.

---

# 30. Gameplay ile Reklam İlişkisi

Gameplay'in ad SDK'sına bağımlı olması engellenmelidir.

Yanlış:

```text
GameplaySystem
    ↓
AdSDK.Show()
```

Doğru:

```text
GameplaySystem
    ↓
Ad Intent / Policy
    ↓
MonetizationSystem
```

Gameplay gerektiğinde:

```text
AdResult
```

üzerinden sonucu değerlendirebilir.

---

# 31. ShopView ile İlişki

`ShopView` yalnızca presentation ve user intent katmanıdır.

Akış:

```text
ShopView
   ↓
Purchase(productId)
   ↓
MonetizationSystem
   ↓
Store
```

ShopView:

* Store SDK çağırmaz.
* Receipt doğrulamaz.
* Reward vermez.
* Currency değiştirmez.
* Purchase state'in sahibi olmaz.
* SaveData değiştirmez.

---

# 32. ShopSystem ile İlişki

ShopSystem varsa:

```text
Shop Configuration
      ↓
ShopSystem
      ↓
Available Offers
      ↓
ShopView
```

Monetization:

```text
ShopView
      ↓
Purchase Intent
      ↓
MonetizationSystem
```

ShopSystem ile MonetizationSystem birbirinin yerine geçmez.

Shop:

> "Oyuncuya hangi teklifleri gösterelim?"

Monetization:

> "Oyuncunun seçtiği ürünü store üzerinden nasıl satın alalım?"

---

# 33. EconomySystem ile İlişki

MonetizationSystem currency'nin sahibi değildir.

Yanlış:

```text
MonetizationSystem
    ↓
Currency State
```

Doğru:

```text
MonetizationSystem
    ↓
RewardSystem
    ↓
EconomySystem
    ↓
Currency State
```

EconomySystem transaction kurallarını kendisi uygular.

---

# 34. ProgressionSystem ile İlişki

Satın alma veya rewarded ad sonucunda progression değişebiliyorsa:

```text
MonetizationSystem
      ↓
RewardSystem
      ↓
ProgressionSystem
```

ProgressionSystem kendi state'inin sahibi olmaya devam eder.

MonetizationSystem progression değerlerini doğrudan değiştirmez.

---

# 35. SaveSystem ile İlişki

MonetizationSystem JSON'a doğrudan yazmaz.

Yanlış:

```text
MonetizationSystem
    ↓
JSON File
```

Doğru:

```text
MonetizationSystem
      ↓
RewardSystem
      ↓
Domain System
      ↓
State Change
      ↓
SaveSystem
```

Kritik purchase/reward işlemlerinden sonra persistence gecikmesi gereksiz yere uzun tutulmamalıdır.

Özellikle gerçek para işlemlerinde:

```text
Purchase
 ↓
Validated Result
 ↓
Reward / Entitlement
 ↓
Persist
```

durumu güvenli şekilde ele alınmalıdır.

---

# 36. Analytics ile İlişki

MonetizationSystem analytics SDK'sını doğrudan kullanmamalıdır.

Örneğin:

```text
PurchaseSuccess
PurchaseFailed
AdStarted
AdCompleted
RewardEarned
```

gibi domain event'leri yayınlanabilir.

Sonrasında:

```text
MonetizationSystem
        ↓
Domain Event
        ↓
AnalyticsSystem
        ↓
Analytics SDK
```

şeklinde ilerlenir.

Analytics event'lerinin UI'dan gönderilmesi tercih edilmez.

---

# 37. Audio ve UI Feedback

Satın alma veya reklam sonucu oluşan feedback presentation katmanında ele alınabilir.

Örneğin:

```text
PurchaseSuccess
      ↓
ShopView
      ↓
Success Feedback
```

veya:

```text
PurchaseSuccess
      ↓
AudioSystem
      ↓
Purchase SFX
```

MonetizationSystem UI veya AudioSystem'i doğrudan kontrol etmemelidir.

---

# 38. Event Kullanımı

Eventler aşağıdaki durumlarda kullanılabilir:

```text
PurchaseStarted
PurchaseCompleted
PurchaseFailed
PurchaseCancelled

RewardedAdStarted
RewardedAdCompleted
RewardEarned
RewardedAdFailed

InterstitialStarted
InterstitialClosed
InterstitialFailed

MonetizationReady
MonetizationUnavailable
```

Event'ler communication içindir.

Event:

> "Ne oldu?"

sorusunu cevaplar.

Event:

> "State'in kendisi"

değildir.

---

# 39. Event Lifecycle

Event subscription Unity lifecycle ile uyumlu olmalıdır.

```text
OnEnable
    ↓
Subscribe

OnDisable
    ↓
Unsubscribe
```

Özellikle pooled UI veya pooled ad-related presentation object'lerde subscription leak oluşmamasına dikkat edilmelidir.

Static event'ler yalnızca gerçekten global lifecycle gerektiriyorsa kullanılmalıdır.

---

# 40. Bootstrap ile İlişki

Monetization SDK initialization genel startup sürecinin bir parçası olabilir.

Örneğin:

```text
Application Start
      ↓
Bootstrap
      ↓
Core Initialization
      ↓
Save Load
      ↓
Monetization Initialization
      ↓
Runtime Systems Ready
      ↓
GameFlow
```

Exact initialization order provider gereksinimlerine göre değişebilir.

Monetization initialization tamamlanmadan önce Shop veya ad sisteminin kullanılması gerekiyorsa readiness contract açıkça tanımlanmalıdır.

---

# 41. Initialization

Initialization:

* Tekrarlanabilir olmamalıdır.
* Idempotent olmalıdır.
* Provider SDK'nın tekrar initialize edilmesi engellenmelidir.
* Failure durumunda retry stratejisi belirlenmelidir.
* Async operasyonların owner'ı belli olmalıdır.
* Scene değişiminden etkilenmemesi gereken sistemlerin lifetime'ı açıkça tanımlanmalıdır.

Örneğin:

```text
Initialize()
Initialize()
Initialize()
```

üç bağımsız SDK initialization başlatmamalıdır.

---

# 42. Retry

Network veya provider kaynaklı başarısızlıklar için retry gerekebilir.

Retry:

* Kontrollü olmalıdır.
* Sonsuz loop olmamalıdır.
* Backoff gerekiyorsa uygulanmalıdır.
* Cancellation/lifetime dikkate alınmalıdır.
* Başarılı initialization sonrası tekrar çalışmamalıdır.

Retry yalnızca gerçekten recoverable olan hatalarda kullanılmalıdır.

---

# 43. Async Lifecycle

Store ve ad SDK'ları callback veya async API sağlayabilir.

Kullanılan async model proje standardıyla uyumlu olmalıdır.

Uzun ömürlü operasyonlarda:

```text
Start
 ↓
Owned Async Operation
 ↓
Success / Failure / Cancel
 ↓
Cleanup
```

lifecycle'ı açık olmalıdır.

Scene destroy veya application shutdown sonrasında callback'in geçersiz object'e erişmesi engellenmelidir.

---

# 44. Application Pause / Resume

Mobil platformlarda uygulama background/foreground geçişleri monetization lifecycle'ını etkileyebilir.

Özellikle:

* Purchase pending işlemleri
* Store callback'leri
* Ad lifecycle
* Provider state
* Transaction recovery

dikkate alınmalıdır.

Application pause sonrasında sistem state'i tekrar kontrol edebilmelidir.

---

# 45. Main Thread

Unity API'leri ve birçok monetization SDK callback'i main thread beklentisine sahip olabilir.

Provider callback'lerinden gelen veriler doğrudan Unity object'lerine uygulanmadan önce projenin threading modeline uygun şekilde ele alınmalıdır.

Özellikle:

```text
Background Callback
        ↓
Main Thread State Update
```

gerekiyorsa açıkça uygulanmalıdır.

---

# 46. Provider Abstraction

`IAPProvider`, `IAdProvider` gibi abstraction'lar yalnızca mimaride "decoupling" kelimesi geçtiği için eklenmemelidir.

Abstraction anlamlıdır eğer:

* Birden fazla store/provider desteklenecekse
* Testlerde gerçek SDK kullanılmaması gerekiyorsa
* Provider değişimi gerçek bir ihtimalse
* SDK API'sinin projeye sızması ciddi bir problem oluşturuyorsa

Örneğin:

```text
MonetizationSystem
       ↓
Provider Adapter
       ↓
External SDK
```

gerekliyse yalnızca kullanılan API yüzeyi kadar küçük tutulmalıdır.

---

# 47. External SDK Isolation

Üçüncü parti SDK tipleri mümkün olduğunca projenin tamamına yayılmamalıdır.

Tercih edilen:

```text
External SDK
     ↓
Monetization Integration
     ↓
Project Domain API
```

Kaçınılması gereken:

```text
ShopView
GameplaySystem
EconomySystem
SettingsPopup
    ↓
External SDK Types
```

Bu durum provider değişikliğini gereksiz yere pahalı hale getirir.

---

# 48. Product ID Kuralları

Product ID'ler stabil olmalıdır.

Bir product ID değiştirilecekse:

* Store configuration
* Existing purchase recovery
* Entitlement
* Save/migration
* Analytics
* Platform mapping

etkileri değerlendirilmelidir.

Product ID'ler UI text'i olarak hard-code edilmemelidir.

---

# 49. Monetization ve Serialization

Serialized configuration ve runtime state birbirinden ayrılmalıdır.

Örneğin:

```text
ProductDefinition
```

configuration'dır.

```text
PurchaseState
```

runtime state'tir.

```text
PlayerSaveData
```

persisted state'tir.

Unity serialized alanlarının değiştirilmesi mevcut asset'lerin bozulmasına neden olabilir.

Serialized field rename gerekiyorsa uygun durumlarda:

```csharp
[FormerlySerializedAs("oldFieldName")]
```

kullanılmalıdır.

---

# 50. Security

Client-side monetization sistemi güvenilir bir server gibi değerlendirilmemelidir.

Özellikle:

* High-value purchases
* Premium entitlement
* Competitive rewards
* Abuse-sensitive rewards

gibi durumlarda server authority gerekebilir.

Client'ta bulunan:

```text
ProductDefinition
Receipt
Price
Reward Definition
```

gibi veriler güvenlik açısından otomatik olarak gizli kabul edilmemelidir.

---

# 51. Cheat ve Manipulation

EconomySystem gibi MonetizationSystem de client'ın manipüle edilebileceği varsayımıyla tasarlanmalıdır.

Özellikle yüksek değerli entitlement'lar için:

```text
Client Purchase Success
```

tek başına server-authoritative bir doğrulama değildir.

Güvenlik gereksinimi yükseldikçe server validation/synchronization eklenebilir.

---

# 52. Performance

Monetization genellikle gameplay hot path değildir.

Buna rağmen:

* Gereksiz polling yapılmamalıdır.
* `Update()` içerisinde sürekli SDK state'i sorgulanmamalıdır.
* Gereksiz allocation yapılmamalıdır.
* Event subscription lifecycle güvenli tutulmalıdır.
* Ad availability cache'lenebiliyorsa uygun şekilde cache'lenmelidir.
* UI refresh yalnızca state değiştiğinde yapılmalıdır.

Özellikle monetization'ın UI tarafından her frame sorgulanması gereksizdir.

---

# 53. Mobile Considerations

Mobil platformlarda:

* Application pause/resume
* Network değişimi
* Store availability
* Ad provider lifecycle
* Memory pressure
* Scene transition
* Interrupted purchase
* App termination

gibi durumlar normal kabul edilmelidir.

MonetizationSystem yalnızca happy path'e göre yazılmamalıdır.

---

# 54. Testing

Monetization kodunun gerçek store veya reklam SDK'sına bağımlı olmadan test edilebilmesi tercih edilir.

## EditMode

Test edilebilecek konular:

* Product mapping
* Purchase state transitions
* Duplicate request prevention
* Reward idempotency
* Entitlement resolution
* Ad state transitions
* Invalid product handling
* Retry rules
* Transaction resolution

## PlayMode

Test edilebilecek konular:

* Initialization lifecycle
* Scene transition
* Event subscription
* Application pause/resume
* UI integration
* Fake provider integration
* Reward delivery integration

Gerçek store transaction'ları otomatik testlerin zorunlu parçası olmamalıdır.

---

# 55. Fake Provider

Test için gerçek SDK yerine fake provider kullanılabilir.

Örneğin:

```text
Fake Purchase
    ↓
Pending
    ↓
Success
```

ve:

```text
Fake Rewarded Ad
    ↓
Completed
    ↓
Reward Earned
```

senaryoları deterministic olarak test edilebilir.

Ancak fake abstraction yalnızca test ihtiyacı gerçekten oluştuğunda eklenmelidir.

---

# 56. Error Handling

Hatalar kullanıcı deneyimine uygun şekilde domain sonucu olarak ifade edilmelidir.

Örneğin:

```text
PurchaseFailed
    ├── ProductUnavailable
    ├── NetworkError
    ├── StoreError
    ├── ValidationFailed
    └── Unknown
```

UI'nin SDK-specific exception veya error code bilmesi tercih edilmez.

Gerekirse MonetizationSystem provider error'larını proje domain error'larına dönüştürür.

---

# 57. UI Feedback

ShopView örneğin:

```text
PurchasePending
PurchaseSuccess
PurchaseFailed
```

durumlarına göre görsel feedback verebilir.

Ancak UI:

```text
if (buttonClicked)
    economy.Add(...)
```

gibi domain mutation yapmamalıdır.

UI yalnızca intent gönderir ve sonucu sunar.

---

# 58. Reward Transaction Sınırı

Reward işlemi mümkün olduğunca transaction mantığıyla ele alınmalıdır.

Örneğin:

```text
Purchase
 ↓
Validated
 ↓
Reward Transaction
 ↓
Reward Applied
 ↓
Persisted
```

Reward'ın bir kısmı uygulanıp diğer kısmının başarısız olduğu multi-reward senaryoları için `RewardSystem` kendi transaction/idempotency politikasını belirlemelidir.

MonetizationSystem bu domain logic'in sahibi değildir.

---

# 59. Event Zincirleri

Aşırı event zincirlerinden kaçınılmalıdır.

Riskli:

```text
PurchaseCompleted
 ↓
Event A
 ↓
Event B
 ↓
Event C
 ↓
Event D
 ↓
Unknown Side Effect
```

Daha açık:

```text
PurchaseCompleted
 ↓
RewardSystem
 ↓
Reward Applied
 ↓
Domain Events
```

Bir event'in hangi sistemi tetiklediği ve state değişikliğinin kimin sorumluluğunda olduğu anlaşılır olmalıdır.

---

# 60. GameFlow ile İlişki

GameFlow global state'in sahibidir.

MonetizationSystem GameFlow değildir.

Örneğin:

```text
Gameplay
 ↓
Request Interstitial
 ↓
MonetizationSystem
 ↓
Ad Result
 ↓
GameFlow continues
```

GameFlow reklam SDK'sını doğrudan çağırmamalıdır.

Benzer şekilde monetization sonucu global state değişikliği gerekiyorsa GameFlow'a uygun domain sonucu veya intent ile bildirilmelidir.

---

# 61. Monetization'ın Source of Truth Olmadığı State'ler

MonetizationSystem aşağıdaki state'lerin sahibi değildir:

```text
Currency
      → EconomySystem

Progression
      → ProgressionSystem

Player Profile
      → ProfileSystem

Save Data
      → SaveSystem

Gameplay State
      → GameplayModule / Gameplay System

Level State
      → LevelSystem
```

Monetization yalnızca satın alma/reklam sürecinin teknik state'ini ve kendi lifecycle'ını yönetir.

---

# 62. Önerilen Klasör Yapısı

Başlangıç için:

```text
Runtime/
└── Systems/
    └── Monetization/
        ├── MonetizationSystem.cs
        ├── PurchaseState.cs
        ├── AdState.cs
        └── ...
```

Configuration:

```text
Configs/
└── Monetization/
    ├── MonetizationConfig.asset
    ├── ProductDefinition.asset
    ├── AdPlacementDefinition.asset
    └── ...
```

Provider entegrasyonu gerçekten gerekiyorsa:

```text
Runtime/
└── Systems/
    └── Monetization/
        └── Providers/
            └── ...
```

Provider abstraction'ları ihtiyaç doğrultusunda eklenmelidir.

---

# 63. Product Definition Nerede Olmalı?

`ProductDefinition` bir ScriptableObject ise configuration katmanında bulunması uygundur:

```text
Configs/Monetization/
    ProductDefinition
```

Runtime-only contract veya data model ise:

```text
Runtime/Systems/Monetization/
```

altında bulunabilir.

Aynı kavramın iki farklı kopyası oluşturulmamalıdır.

---

# 64. Genel IAP Akışı

```text
                Product Configuration
                        ↓
                     Shop
                        ↓
                 Purchase Intent
                        ↓
              MonetizationSystem
                        ↓
                    Store SDK
                        ↓
                 Purchase Result
                        ↓
               Validation / Resolve
                        ↓
                  RewardSystem
                        ↓
          ┌─────────────┴─────────────┐
          ↓                           ↓
   EconomySystem              ProgressionSystem
          ↓                           ↓
                    SaveSystem
```

---

# 65. Genel Rewarded Ad Akışı

```text
Feature / UI
     ↓
Request Rewarded Ad
     ↓
MonetizationSystem
     ↓
Ad Provider
     ↓
Ad Completed
     ↓
Reward Earned
     ↓
RewardSystem
     ↓
Relevant Domain System
     ↓
SaveSystem
```

---

# 66. Genel Interstitial Akışı

```text
Gameplay / GameFlow / Ad Policy
              ↓
       Request Interstitial
              ↓
       MonetizationSystem
              ↓
            Ad SDK
              ↓
        Show / Close / Fail
              ↓
       Continue Game Flow
```

Interstitial başarısız olursa ana gameplay akışı gereksiz yere kilitlenmemelidir.

---

# 67. Anti-Pattern'ler

Aşağıdakilerden kaçınılmalıdır:

```text
ShopView → Store SDK
```

```text
MonetizationSystem → EconomySystem.Add()
```

```text
MonetizationSystem → JSON
```

```text
GameplaySystem → Ad SDK
```

```text
UI → Receipt Validation
```

```text
UI → PlayerSaveData mutation
```

```text
Rewarded Ad Started → Reward Immediately
```

```text
Purchase Success → Reward without duplicate protection
```

```text
Update() → IsAdAvailable() sürekli polling
```

```text
Her feature → kendi IAP SDK integration'ı
```

```text
Her feature → kendi Ad SDK integration'ı
```

Ayrıca yalnızca "decoupled" görünmek için gereksiz provider/factory/interface katmanları eklenmemelidir.

---

# 68. Extension Rules

Yeni bir monetization özelliği eklenirken önce şu sorular sorulmalıdır:

1. Bu özellik gerçekten monetization domain'ine mi ait?
2. Existing IAP veya Ad flow ile çözülebiliyor mu?
3. Yeni runtime state gerekiyor mu?
4. State'in sahibi kim?
5. Reward gerekiyorsa RewardSystem kullanılabiliyor mu?
6. Currency değişiyorsa EconomySystem mi değiştirecek?
7. Progression değişiyorsa ProgressionSystem mi değiştirecek?
8. Persistence gerekiyorsa SaveSystem nasıl devreye girecek?
9. Analytics event'i nereden üretilecek?
10. UI bu özelliğin yalnızca presentation/intent tarafında mı kalıyor?
11. Provider abstraction gerçekten gerekli mi?
12. Existing serialized asset'ler etkileniyor mu?

---

# 69. Source of Truth

Monetization mimarisinde source of truth:

```text
Product Configuration
        ↓
Static Product Definition

Store
        ↓
Purchase / Store Metadata

MonetizationSystem
        ↓
Purchase / Ad Lifecycle

RewardSystem
        ↓
Reward Application Orchestration

EconomySystem
        ↓
Currency State

ProgressionSystem
        ↓
Progression / Entitlement-related Domain State

SaveSystem
        ↓
Persistent State
```

Eventler bu state'lerin yerine geçmez.

UI hiçbir state'in source of truth'u değildir.

---

# 70. Generic Template Rule

Bu template farklı oyun türlerine uygulanmalıdır.

Core monetization sisteminde aşağıdaki gibi oyuna özel kavramlar bulunmamalıdır:

```text
Kingdom
Star
Building
Decoration
Chest
Lives
```

Bunun yerine:

```text
Product
Offer
Reward
Currency
Entitlement
Progression
Ad Placement
Purchase
```

gibi generic kavramlar kullanılmalıdır.

Oyuna özel monetization davranışları feature seviyesinde eklenebilir.

---

# 71. Definition of Done

Bir monetization özelliği tamamlanmış sayılmadan önce:

* [ ] Sorumluluğu doğru sistemde mi?
* [ ] UI SDK'ya doğrudan erişmiyor mu?
* [ ] Shop ile Monetization ayrılmış mı?
* [ ] Reward ile Monetization ayrılmış mı?
* [ ] Economy state'i EconomySystem'de mi?
* [ ] Progression state'i ProgressionSystem'de mi?
* [ ] Save işlemleri SaveSystem üzerinden mi?
* [ ] Purchase lifecycle açıkça modellenmiş mi?
* [ ] Pending transaction recovery düşünülmüş mü?
* [ ] Duplicate purchase engellenmiş mi?
* [ ] Reward delivery idempotent mi?
* [ ] Consumable ve entitlement ayrımı yapılmış mı?
* [ ] Restore ihtiyacı değerlendirilmiş mi?
* [ ] Store metadata UI'ya doğru şekilde aktarılıyor mu?
* [ ] Gerçek para fiyatları hard-code edilmemiş mi?
* [ ] Rewarded ad yalnızca verified reward callback ile ödül veriyor mu?
* [ ] Interstitial gameplay'i gereksiz yere bloke etmiyor mu?
* [ ] Ad failure durumları ele alınıyor mu?
* [ ] Initialization idempotent mi?
* [ ] Async lifecycle güvenli mi?
* [ ] Pause/resume senaryoları düşünülmüş mü?
* [ ] Provider SDK projeye gereksiz şekilde sızmıyor mu?
* [ ] Gereksiz abstraction eklenmemiş mi?
* [ ] Analytics SDK doğrudan çağrılmıyor mu?
* [ ] Serialization etkileri kontrol edilmiş mi?
* [ ] Security/validation gereksinimi değerlendirilmiş mi?
* [ ] EditMode/PlayMode testleri mümkün olduğunca ayrıştırılmış mı?
* [ ] Mobil performans ve lifecycle dikkate alınmış mı?
* [ ] Generic template prensipleri korunmuş mu?

---

# 72. Sonuç

`MonetizationSystem`, oyunun para kazanma altyapısının teknik sınırıdır.

En önemli ayrım:

```text
Shop
    → Ne gösterilecek?

Monetization
    → Satın alma / reklam nasıl yürütülecek?

RewardSystem
    → Başarılı işlem sonucunda hangi ödül nasıl uygulanacak?

EconomySystem
    → Currency state nasıl değişecek?

ProgressionSystem
    → Progression state nasıl değişecek?

SaveSystem
    → Kalıcı olarak nasıl saklanacak?

AnalyticsSystem
    → Olay nasıl raporlanacak?
```

Bu sınırlar korunduğunda yeni bir ürün, yeni bir reklam placement'ı veya farklı bir reward türü eklemek mevcut sistemlerin birbirine dolanmasını gerektirmez.

Temel prensip:

> MonetizationSystem transaction'ın sahibidir; ödülün, currency'nin, progression'ın veya UI'ın sahibi değildir.
