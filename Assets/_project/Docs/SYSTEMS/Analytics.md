# Analytics System

## 1. Amaç

`AnalyticsSystem`, oyunun runtime davranışını ölçülebilir analytics event'lerine dönüştürür ve bu event'leri güvenilir şekilde analytics backend'ine iletir.

Analytics sistemi:

* Gameplay state'in sahibi değildir.
* Economy state'in sahibi değildir.
* Progression state'in sahibi değildir.
* Save state'in sahibi değildir.
* UI state'in sahibi değildir.
* Telegram'ın kendisini analytics database olarak kullanmaz.
* Gameplay'i network problemi nedeniyle bloklamaz.

Temel amaç:

```text
Runtime Behavior
      ↓
Analytics Event
      ↓
AnalyticsSystem
      ↓
Event Buffer / Queue
      ↓
Analytics Backend
      ↓
Dashboard / Analysis
```

Opsiyonel notification akışı:

```text
Analytics Event
      ↓
Analytics Backend
      ↓
Alert Rules
      ↓
Telegram Notifier
      ↓
Telegram
```

Telegram analytics source of truth değildir.

---

# 2. Temel Mimari

Önerilen production mimarisi:

```text
┌─────────────────────────────┐
│ Gameplay / Systems / UI      │
└──────────────┬──────────────┘
               ↓
        Domain / Analytics
             Events
               ↓
┌─────────────────────────────┐
│      AnalyticsSystem        │
│                             │
│ - Event Contract            │
│ - Common Context            │
│ - Queue / Buffer            │
│ - Batching                  │
│ - Retry                     │
└──────────────┬──────────────┘
               ↓
        Analytics Backend
               │
       ┌───────┴────────┐
       ↓                ↓
   Dashboard       Alert Engine
                         ↓
                  Telegram Bot API
```

Debug sırasında local log da kullanılabilir:

```text
AnalyticsSystem
      ↓
┌─────┼──────────────┐
↓     ↓              ↓
Backend Telegram   Local Debug
```

---

# 3. Sorumluluklar

## AnalyticsSystem Sorumlulukları

`AnalyticsSystem`:

* analytics event contract'larını yönetir
* event'leri kabul eder
* ortak context bilgilerini ekler
* event'leri queue/buffer'a alır
* event'leri batch halinde gönderebilir
* network failure durumunda retry yapabilir
* gerektiğinde local buffer kullanabilir
* analytics provider'a event gönderir
* provider entegrasyonunu gameplay kodundan izole eder

## AnalyticsSystem Sorumlulukları Değildir

AnalyticsSystem:

* level state tutmaz
* player progression hesaplamaz
* currency değiştirmez
* reward vermez
* purchase gerçekleştirmez
* gameplay sonucu belirlemez
* Telegram mesajı doğrudan göndermek zorunda değildir
* dashboard UI'ı Unity içerisinde barındırmaz

---

# 4. Analytics Source of Truth Değildir

Analytics verisi gözlem amaçlıdır.

Örneğin:

```text
LevelSystem
    ↓
Current Level = 42
```

Analytics:

```text
LevelStarted(levelId = 42)
```

event'ini kaydeder.

Analytics database'deki `levelId = 42` bilgisi level state'in sahibi değildir.

Benzer şekilde:

```text
EconomySystem
    ↓
Coins = 1500
```

Analytics:

```text
CurrencyChanged
```

event'ini kaydedebilir.

Analytics üzerinden economy state restore edilmemelidir.

---

# 5. Event Üretimi

Event mümkün olduğunca olayı gerçekten gerçekleştiren sistem tarafından üretilmelidir.

Örneğin:

```text
LevelSystem
    ↓
LevelCompleted
    ↓
AnalyticsSystem
```

Purchase:

```text
MonetizationSystem
    ↓
PurchaseCompleted
    ↓
AnalyticsSystem
```

Tutorial:

```text
TutorialSystem
    ↓
TutorialCompleted
    ↓
AnalyticsSystem
```

UI doğrudan analytics SDK'sına çağrı yapmamalıdır.

Yanlış:

```text
ShopView
    ↓
AnalyticsSDK.TrackPurchase()
```

Doğru:

```text
ShopView
    ↓
Purchase Intent
    ↓
MonetizationSystem
    ↓
PurchaseCompleted
    ↓
AnalyticsSystem
```

---

# 6. Minimal API

AnalyticsSystem mümkün olduğunca küçük bir API sunmalıdır.

Örneğin:

```csharp
public interface IAnalytics
{
    void Track(string eventName);
    void Track(string eventName, AnalyticsProperties properties);
}
```

Gerçek implementasyon:

```text
IAnalytics
    ↓
AnalyticsSystem
    ↓
Analytics Provider
```

Ancak gerçek projede interface gerekmiyorsa yalnızca:

```csharp
AnalyticsSystem.Track(...)
```

kullanılabilir.

Sırf architecture adına gereksiz abstraction eklenmemelidir.

---

# 7. Event Contract

Her analytics event'i minimum olarak şu ortak bilgileri taşıyabilir:

```text
eventName
timestampUtc
playerId
sessionId
appVersion
platform
```

Opsiyonel:

```text
levelId
attemptId
country
locale
properties
```

Örnek:

```json
{
  "eventName": "LevelStarted",
  "timestampUtc": "2026-09-05T12:34:56Z",
  "playerId": "p_7f91...",
  "sessionId": "s_2ac4...",
  "levelId": 42,
  "attemptId": 3,
  "appVersion": "1.4.0",
  "platform": "iOS"
}
```

---

# 8. Player ID

Analytics için stabil ve mümkün olduğunca pseudonymous bir player ID kullanılmalıdır.

Örneğin:

```text
playerId
```

Kimlik bilgileri analytics event'lerine gereksiz yere eklenmemelidir.

Analytics event'lerinde:

* email
* telefon
* açık isim
* adres
* authentication token
* payment information

gibi PII bilgiler gereksiz yere tutulmamalıdır.

---

# 9. Session ID

Her uygulama oturumunda yeni bir `sessionId` oluşturulabilir.

Örnek:

```text
Application Launch
       ↓
SessionStarted
       ↓
sessionId = abc123
```

Uygulama kapanır veya session sona ererse:

```text
SessionEnded
```

gönderilir.

Yeni launch:

```text
sessionId = xyz456
```

olur.

---

# 10. Event Naming

Event isimleri:

* stable
* açık
* kısa
* anlamlı
* tutarlı

olmalıdır.

Örnek:

```text
SessionStarted
SessionEnded

LevelStarted
LevelCompleted
LevelFailed
LevelRestarted
LevelAbandoned

TutorialStarted
TutorialCompleted
TutorialSkipped

PurchaseStarted
PurchaseCompleted
PurchaseFailed

RewardedAdStarted
RewardedAdCompleted
RewardedAdFailed

AppBackgrounded
AppForegrounded
```

Event isimleri sonradan değiştirilmemelidir.

Bir event'in anlamı değişiyorsa yeni event oluşturmak daha güvenlidir.

---

# 11. Temel Gameplay Event'leri

Başlangıçta şu event'ler yeterli bir temel oluşturur:

```text
LevelStarted
LevelCompleted
LevelFailed
LevelRestarted
LevelAbandoned
```

Örnek:

```text
LevelStarted
{
    levelId: 42,
    attemptId: 1
}
```

```text
LevelFailed
{
    levelId: 42,
    attemptId: 1,
    durationSeconds: 87
}
```

```text
LevelCompleted
{
    levelId: 42,
    attemptId: 2,
    durationSeconds: 64
}
```

---

# 12. Level Abandonment

`PlayerGotBored` gibi bir event doğrudan gerçek olarak gönderilmemelidir.

Oyuncunun sıkıldığını kesin olarak bilemeyiz.

Bunun yerine gözlemlenebilir event'ler gönderilir:

```text
LevelStarted
AppBackgrounded
SessionEnded
LevelCompleted
```

Analytics backend bu event'lerden abandonment türetebilir.

Örneğin:

```text
LevelStarted
     ↓
No LevelCompleted
     ↓
SessionEnded
```

belirli koşullar altında:

```text
LevelAbandoned
```

olarak raporlanabilir.

---

# 13. Stuck Player Analizi

AnalyticsSystem doğrudan:

```text
PlayerIsStuck
```

state'i üretmek zorunda değildir.

Backend observable davranışlardan bunu hesaplayabilir.

Örneğin:

```text
LevelStarted
LevelFailed
LevelRestarted
LevelFailed
LevelRestarted
LevelFailed
...
```

veya:

```text
LevelStarted
      ↓
Very Long Dwell Time
      ↓
No Completion
```

stuck level sinyali olabilir.

Dashboard configurable threshold kullanabilir.

Örneğin:

```text
Attempts >= 5
AND
Level not completed
```

veya:

```text
Level dwell time > configured threshold
```

---

# 14. Session Event'leri

Temel session event'leri:

```text
SessionStarted
SessionEnded
AppBackgrounded
AppForegrounded
```

Session duration backend tarafında hesaplanabilir:

```text
SessionStarted
        ↓
SessionEnded
        ↓
Session Duration
```

Uygulamanın crash olması durumunda `SessionEnded` gelmeyebilir.

Bu normaldir.

Backend:

```text
lastEvent timestamp
```

üzerinden incomplete session'ları analiz edebilir.

---

# 15. Purchase Analytics

Purchase event'leri MonetizationSystem tarafından üretilmelidir.

Örnek:

```text
PurchaseStarted
PurchaseCompleted
PurchaseFailed
```

Event properties:

```text
productId
transactionId
price
currency
provider
```

Transaction ID gibi provider-specific bilgiler yalnızca güvenli ve gerekli olduğu ölçüde gönderilmelidir.

Purchase analytics ile actual purchase entitlement state birbirinden ayrıdır.

```text
MonetizationSystem
      ↓
Purchase State
```

ve:

```text
MonetizationSystem
      ↓
Analytics Event
```

ayrı akışlardır.

---

# 16. Rewarded Ads

Rewarded ad event'leri:

```text
RewardedAdStarted
RewardedAdCompleted
RewardedAdFailed
```

Ödül:

```text
RewardedAdCompleted
        ↓
RewardSystem
        ↓
Reward
```

Analytics event'i ödülün kendisini vermemelidir.

Özellikle:

```text
AdStarted
```

event'i ödül verileceği anlamına gelmez.

Ödül yalnızca doğrulanmış completion sonucundan sonra verilmelidir.

---

# 17. Tutorial Analytics

Tutorial event'leri:

```text
TutorialStarted
TutorialCompleted
TutorialSkipped
```

Opsiyonel properties:

```text
tutorialId
stepId
durationSeconds
```

TutorialSystem tutorial progression'ın sahibidir.

AnalyticsSystem yalnızca gözlemler.

---

# 18. Event Priority

Analytics event'leri önem seviyelerine ayrılabilir.

## Critical

Örneğin:

```text
PurchaseCompleted
PurchaseFailed
EntitlementChanged
SecurityRelatedEvent
```

## Important

Örneğin:

```text
LevelStarted
LevelCompleted
LevelFailed
SessionStarted
SessionEnded
TutorialCompleted
```

## Debug / Verbose

Örneğin:

```text
UIOpened
ButtonClicked
MinorInteraction
```

Her event'i Telegram'a göndermek doğru değildir.

---

# 19. Queue ve Buffer

Analytics network request'leri gameplay thread'ini bloklamamalıdır.

Önerilen akış:

```text
Track(...)
   ↓
Memory Queue
   ↓
Batch
   ↓
Network
```

Örneğin:

```text
10 event
   ↓
1 HTTP request
```

yapılabilir.

Bu:

* network overhead'i azaltır
* battery kullanımını azaltır
* analytics performansını iyileştirir

---

# 20. Gameplay'i Bloklamama

Analytics gönderimi:

```text
Gameplay
   ↓
Track()
```

çağrısından sonra oyuncunun beklemesine neden olmamalıdır.

Yanlış:

```text
LevelCompleted
    ↓
await SendAnalytics()
    ↓
continue gameplay
```

Doğru:

```text
LevelCompleted
    ↓
Queue Analytics Event
    ↓
Continue Gameplay
```

---

# 21. Network Failure

Analytics backend ulaşılamıyorsa gameplay bozulmamalıdır.

Örneğin:

```text
Track()
  ↓
Queue
  ↓
Network Failed
  ↓
Retry
```

Retry mekanizması exponential backoff kullanabilir.

Örneğin:

```text
1s
2s
4s
8s
...
```

Maximum retry sınırı bulunmalıdır.

---

# 22. Offline Buffer

Gerekli görülürse event'ler kısa süreli local buffer'a yazılabilir.

```text
Analytics Event
      ↓
Local Buffer
      ↓
Network Available
      ↓
Upload
```

Ancak analytics sistemi için sınırsız local storage kullanılmamalıdır.

Buffer:

* bounded
* temizlenebilir
* version-safe

olmalıdır.

---

# 23. Event Batching

Event'ler mümkün olduğunca batch halinde gönderilebilir.

Örneğin:

```text
LevelStarted
LevelFailed
LevelRestarted
LevelFailed
```

tek request içinde gönderilebilir.

Batch boyutu ve flush interval configuration ile belirlenebilir.

---

# 24. App Lifecycle

Mobile uygulamalarda:

```text
Foreground
    ↓
Gameplay
    ↓
Background
```

geçişleri analytics açısından önemlidir.

Örnek:

```text
AppBackgrounded
```

event'i gönderilebilir.

Ancak OS uygulamayı aniden öldürebilir.

Bu nedenle analytics sistemi `SessionEnded` event'inin her zaman geleceğini varsaymamalıdır.

---

# 25. Duplicate Event

Network retry sırasında aynı event'in iki kez gönderilmesi mümkün olabilir.

Özellikle:

```text
PurchaseCompleted
```

gibi event'lerde duplicate koruması önemlidir.

Gerekirse event'lere:

```text
eventId
```

eklenebilir.

Backend:

```text
eventId
```

üzerinden deduplication yapabilir.

Analytics duplicate protection gameplay state protection'ın yerine geçmez.

---

# 26. Idempotency

Purchase analytics ve monetization event'leri idempotent tasarlanmalıdır.

Örneğin:

```text
PurchaseCompleted
transactionId = ABC
```

aynı transaction için tekrar gelirse backend duplicate olarak değerlendirebilir.

Actual entitlement logic ise MonetizationSystem tarafından korunmalıdır.

---

# 27. Analytics Backend

Production ortamında analytics event'lerinin merkezi bir backend'de tutulması önerilir.

```text
Unity
 ↓ HTTPS
Analytics API
 ↓
Event Store
 ↓
Dashboard
```

Backend:

* event ingestion
* validation
* storage
* aggregation
* retention
* dashboard query
* alerting

gibi sorumlulukları üstlenebilir.

Unity client analytics database'in kendisi değildir.

---

# 28. Dashboard

Dashboard analytics backend üzerinden çalışmalıdır.

Telegram mesajları dashboard olarak kullanılmamalıdır.

Dashboard aşağıdaki metrikleri gösterebilir:

```text
DAU
WAU
MAU

D1 Retention
D3 Retention
D7 Retention
D14 Retention
D30 Retention

Sessions / Day
Average Session Duration

Level Completion Rate
Level Failure Rate
Level Abandonment Rate

Attempts per Level
Level Dwell Time

Stuck Level Leaderboard
Player Dropoff by Level

Purchase Funnel
Rewarded Ad Funnel

Tutorial Completion Rate
```

---

# 29. Retention

Retention:

```text
Day 0
  ↓
Player Acquired
  ↓
Did player return?
```

şeklinde analiz edilebilir.

Örneğin:

```text
D1
D3
D7
D14
D30
```

retention metrikleri dashboard'da tutulabilir.

---

# 30. Level Funnel

Dashboard level funnel gösterebilir:

```text
Level Started
      ↓
Level Attempted
      ↓
Level Completed
```

Örneğin:

```text
Level 42

Started:     10,000
Completed:    7,200
Failed:       2,300
Abandoned:      500
```

Bu veriler hangi level'ların problemli olduğunu anlamaya yardımcı olur.

---

# 31. Stuck Level Leaderboard

Önemli dashboard ekranlarından biri:

```text
Level | Avg Attempts | Avg Dwell | Completion Rate
```

Örneğin:

```text
42    | 6.4 | 08:42 | 41%
57    | 5.9 | 07:51 | 48%
63    | 5.2 | 09:12 | 52%
```

Threshold'lar oyunun genre'sine göre configuration ile belirlenmelidir.

---

# 32. Player-Level Investigation

Gerekirse dashboard'da pseudonymous player-level görünüm bulunabilir:

```text
Player ID
Last Active
Current / Last Level
Attempts
Last Session Duration
Last Event
```

Örneğin:

```text
Player      Last Active    Level    Attempts    Last Event
p_123       10:42           42       7           LevelFailed
p_456       10:38           57       3           AppBackgrounded
```

Bu ekran debugging ve product analysis amacıyla kullanılmalıdır.

PII gereksiz yere gösterilmemelidir.

---

# 33. Churn

Oyuncunun "sıkıldığını" doğrudan ölçmek yerine observable behavior kullanılmalıdır.

Örneğin:

```text
LastActive
+
No subsequent SessionStarted
```

belirli bir süre sonrasında churn olarak sınıflandırılabilir.

Örneğin:

```text
No session for 3 days
```

bir analiz kuralı olabilir.

Threshold oyuna göre değişebilir.

---

# 34. Telegram Notifications

Telegram bir notification channel'dır.

Önerilen:

```text
Analytics Backend
       ↓
Alert Rule Engine
       ↓
Telegram Notifier
       ↓
Telegram Bot API
```

Unity:

```text
Unity
  X
  ↓
Telegram Bot API
```

şeklinde doğrudan Telegram'a bağlanmamalıdır.

---

# 35. Telegram Bot Security

Telegram bot token Unity client'a konulmamalıdır.

Yanlış:

```text
Unity
 ├── Telegram Bot Token
 └── Telegram API
```

Doğru:

```text
Unity
 ↓
Analytics Backend
 ↓
Secure Server Secret
 ↓
Telegram Bot API
```

Bot token:

* Unity source code'da
* ScriptableObject'te
* PlayerPrefs'te
* Remote config'te
* client-side JSON'da

bulunmamalıdır.

---

# 36. Telegram Webhook Kavramı

Telegram'da webhook genellikle botun Telegram'dan update alması için kullanılır.

Analytics notification göndermek için temel ihtiyaç:

```text
Backend
   ↓
Telegram Bot API
   ↓
sendMessage
```

akışıdır.

Inbound Telegram webhook yalnızca bot üzerinden komut veya yönetim özelliği gerekiyorsa eklenmelidir.

Basic analytics notification sistemi için zorunlu değildir.

---

# 37. Telegram Alert Örnekleri

Her analytics event'ini Telegram'a göndermek önerilmez.

Yanlış:

```text
Her LevelStarted
    ↓
Telegram
```

Bu kısa sürede notification spam'ine dönüşür.

Daha doğru:

```text
Level 42 failure rate unusually high
        ↓
Telegram Alert
```

veya:

```text
Purchase failure rate increased
        ↓
Telegram Alert
```

veya:

```text
Analytics provider unavailable
        ↓
Telegram Alert
```

---

# 38. Alert Rules

Alert sistemi aşağıdaki parametrelere sahip olabilir:

```text
condition
threshold
aggregationWindow
cooldown
severity
maxNotifications
```

Örneğin:

```text
Condition:
Level failure rate > 60%

Window:
15 minutes

Cooldown:
60 minutes

Severity:
High
```

Böylece aynı problem için her saniye Telegram mesajı gönderilmez.

---

# 39. Önerilen Telegram Alert'leri

Başlangıç için:

```text
Purchase failure spike
Purchase provider unavailable

Level failure spike
Level abandonment spike
Stuck player threshold exceeded

Analytics backend unavailable
Crash/error spike

Major retention/dropoff anomaly
```

Individual player session başlangıçlarını Telegram'a göndermek genellikle gereksizdir.

---

# 40. Environment Separation

Production ve development notification kanalları ayrılmalıdır.

```text
Development
    ↓
Dev Telegram Bot / Channel

Staging
    ↓
Staging Telegram Bot / Channel

Production
    ↓
Production Telegram Bot / Channel
```

Development event'lerinin production kanalına gitmesi engellenmelidir.

---

# 41. Analytics Context

Common context event'lere merkezi olarak eklenebilir.

Örneğin:

```text
appVersion
platform
playerId
sessionId
locale
country
```

Her feature'ın bunları tekrar tekrar üretmesi gerekmemelidir.

```text
AnalyticsSystem
      ↓
Common Context
      ↓
Event
```

---

# 42. Feature Properties

Feature-specific bilgiler event properties içerisinde tutulabilir.

Örneğin:

```text
LevelCompleted

levelId
attemptId
durationSeconds
score
```

Purchase:

```text
PurchaseCompleted

productId
transactionId
price
currency
```

Properties gereksiz derecede büyük olmamalıdır.

Analytics event'lerine büyük JSON payload'lar koyulmamalıdır.

---

# 43. Analytics ve SaveSystem

Analytics event'leri SaveSystem state'i değildir.

SaveSystem:

```text
PlayerSaveData
```

persist eder.

Analytics:

```text
Behavior Events
```

gönderir.

Yanlış:

```text
Analytics JSON
    ↓
Player Save
```

Doğru:

```text
Runtime State
    ├── SaveSystem
    └── AnalyticsSystem
```

---

# 44. Analytics ve Economy

EconomySystem currency state'in sahibidir.

Analytics yalnızca transaction'ları gözlemler.

```text
EconomySystem
      ↓
CurrencyChanged
      ↓
AnalyticsSystem
```

AnalyticsSystem currency değiştiremez.

---

# 45. Analytics ve Progression

ProgressionSystem progression state'in sahibidir.

Analytics:

```text
ProgressionChanged
UnlockChanged
MilestoneCompleted
```

gibi event'leri izleyebilir.

Analytics üzerinden progression restore edilmemelidir.

---

# 46. Analytics ve Monetization

MonetizationSystem:

* IAP
* ads
* entitlement
* purchase lifecycle

gibi konuların sahibidir.

Analytics:

```text
PurchaseStarted
PurchaseCompleted
PurchaseFailed
RewardedAdStarted
RewardedAdCompleted
```

gibi event'leri toplar.

ShopView doğrudan analytics provider'a bağlanmamalıdır.

---

# 47. Analytics ve GameFlow

GameFlow global state transition'ın sahibidir.

Örneğin:

```text
MainMenu
   ↓
Gameplay
```

geçişi GameFlow tarafından yönetilir.

Analytics bunu gözlemleyebilir:

```text
GameFlow
   ↓
GameplayEntered
   ↓
AnalyticsSystem
```

Analytics GameFlow state'ini değiştirmez.

---

# 48. Analytics ve UI

UI analytics event üretmek zorunda değildir.

Örneğin ShopView:

```text
User Click
    ↓
Purchase Intent
```

MonetizationSystem:

```text
PurchaseStarted
```

event'ini üretir.

Böylece analytics gerçek domain davranışını ölçer.

Sadece UI click'i ölçmek isteniyorsa ayrı bir interaction event'i kullanılabilir.

---

# 49. Debug Logging

Development build'de analytics event'leri console'a yazdırılabilir.

Örneğin:

```text
[Analytics]
LevelCompleted
levelId=42
attemptId=2
```

Ancak production build'de verbose analytics logları kapatılmalıdır.

---

# 50. Provider Abstraction

Analytics provider için abstraction yalnızca gerçek bir ihtiyaç varsa eklenmelidir.

Örneğin:

```text
AnalyticsSystem
      ↓
Analytics Provider
```

Birden fazla provider gerekiyorsa:

```text
AnalyticsSystem
      ↓
IAnalyticsProvider
      ├── Provider A
      └── Provider B
```

kullanılabilir.

Tek provider bulunan küçük bir projede gereksiz interface/factory zinciri oluşturulmamalıdır.

---

# 51. Dashboard Provider

Dashboard için:

* managed analytics platform
* custom analytics backend
* internal dashboard

gibi çözümler kullanılabilir.

Core Unity architecture belirli bir vendor'a bağımlı olmamalıdır.

Önemli olan:

```text
Unity Event Contract
        ↓
Analytics Backend
        ↓
Dashboard
```

kontratının stabil olmasıdır.

---

# 52. Event Versioning

Analytics event schema'sı zamanla değişebilir.

Gerekirse:

```text
schemaVersion
```

kullanılabilir.

Örneğin:

```text
LevelCompleted
schemaVersion = 2
```

Analytics dashboard eski ve yeni event'leri ayırt edebilmelidir.

Event property isimleri gereksiz yere değiştirilmemelidir.

---

# 53. Privacy

Analytics yalnızca ihtiyaç duyulan veriyi toplamalıdır.

Özellikle:

* PII
* authentication secrets
* payment credentials
* access tokens
* internal secrets

analytics event'lerine eklenmemelidir.

Platform ve bölge bilgileri gerekiyorsa privacy requirements dikkate alınmalıdır.

---

# 54. Performance

Analytics:

* `Update()` içerisinde sürekli allocation oluşturmamalı
* gereksiz string allocation yapmamalı
* her event için gereksiz network request göndermemeli
* gameplay thread'ini bloklamamalı
* büyük payload üretmemeli
* gereksiz JSON serialization yapmamalı

Hot path event'leri için allocation davranışı özellikle kontrol edilmelidir.

---

# 55. Retry ve Backoff

Retry sistemi:

```text
Request Failed
      ↓
Retry #1
      ↓
Retry #2
      ↓
Retry #3
      ↓
Drop / Persist
```

şeklinde sınırlı olmalıdır.

Sonsuz retry yapılmamalıdır.

Network uzun süre yoksa bounded local queue kullanılabilir.

---

# 56. Testing

AnalyticsSystem test edilebilir olmalıdır.

## EditMode

Test edilebilecek konular:

* event oluşturma
* common context
* serialization
* queue
* batching
* retry policy
* deduplication
* event validation

## PlayMode

Test edilebilecek konular:

* lifecycle
* initialization
* background/foreground
* integration
* provider communication

Fake provider kullanılabilir:

```text
AnalyticsSystem
      ↓
FakeAnalyticsProvider
```

Test sırasında gerçek backend'e event gönderilmemelidir.

---

# 57. Failure Isolation

Analytics başarısızlığı gameplay'i bozmamalıdır.

Örneğin:

```text
Analytics Backend DOWN
```

durumunda:

```text
Level Start
Level Complete
Reward
Save
```

normal şekilde çalışmaya devam etmelidir.

Analytics:

```text
Best Effort
```

davranışında olabilir.

Ancak purchase/entitlement gibi kritik finansal state'ler analytics'e bağlı değildir.

---

# 58. Source of Truth

| Konu                   | Source of Truth              |
| ---------------------- | ---------------------------- |
| Current Level          | LevelSystem                  |
| Gameplay State         | Gameplay Module              |
| Currency               | EconomySystem                |
| Progression            | ProgressionSystem            |
| Purchase State         | MonetizationSystem           |
| Save Data              | SaveSystem                   |
| Settings               | SettingsSystem               |
| Analytics Events       | AnalyticsSystem              |
| Dashboard Metrics      | Analytics Backend            |
| Telegram Notifications | Alert / Notification Backend |

---

# 59. Genel Akış

Normal gameplay:

```text
Player
 ↓
Input
 ↓
Gameplay
 ↓
Gameplay Result
 ↓
System State Change
 ↓
Domain Event
 ↓
AnalyticsSystem
 ↓
Queue
 ↓
Analytics Backend
 ↓
Dashboard
```

Notification gereken durumda:

```text
Analytics Backend
 ↓
Alert Rule
 ↓
Telegram Notifier
 ↓
Telegram
```

---

# 60. Örnek: Level Tamamlama

```text
Player completes level
        ↓
Gameplay Module
        ↓
LevelSystem
        ↓
LevelCompleted
        ↓
ProgressionSystem
        ↓
RewardSystem
        ↓
EconomySystem
```

Analytics paralel gözlem akışı:

```text
LevelCompleted
        ↓
AnalyticsSystem
        ↓
Queue
        ↓
Analytics Backend
```

Analytics reward işlemini yönetmez.

---

# 61. Örnek: Purchase

```text
ShopView
    ↓
Purchase Intent
    ↓
MonetizationSystem
    ↓
Store Provider
    ↓
Purchase Result
    ↓
Reward / Entitlement
```

Analytics:

```text
PurchaseStarted
       ↓
AnalyticsSystem

PurchaseCompleted
       ↓
AnalyticsSystem

PurchaseFailed
       ↓
AnalyticsSystem
```

Telegram:

```text
Purchase Failure Spike
       ↓
Alert Engine
       ↓
Telegram
```

---

# 62. Örnek: Oyuncunun Level'da Takılması

Unity tarafı:

```text
LevelStarted
LevelFailed
LevelRestarted
LevelFailed
LevelRestarted
LevelFailed
```

Analytics backend:

```text
Attempts >= threshold
+
Long dwell time
+
No completion
```

↓

```text
Stuck Level
```

Dashboard:

```text
Level 42
Completion Rate: 41%
Average Attempts: 6.4
Average Dwell: 08:42
```

Gerekirse:

```text
Stuck Level Alert
       ↓
Telegram
```

---

# 63. Analytics Data Flow

```text
                  ┌──────────────────┐
                  │ Gameplay Systems │
                  └────────┬─────────┘
                           ↓
                  ┌──────────────────┐
                  │ AnalyticsSystem  │
                  └────────┬─────────┘
                           ↓
                    Event Queue
                           ↓
                       Batching
                           ↓
                  ┌──────────────────┐
                  │ Analytics API    │
                  └────────┬─────────┘
                           ↓
                    Event Storage
                     ┌─────┴─────┐
                     ↓           ↓
                 Dashboard    Alert Engine
                                   ↓
                              Telegram
```

---

# 64. Generic Template Rule

Bu analytics sistemi herhangi bir hybrid-casual oyun türünde kullanılabilecek şekilde generic kalmalıdır.

Analytics core içine:

```text
Royal Kingdom
Kingdom
Star
Building
Decoration
```

gibi belirli bir oyuna ait domain kavramları eklenmemelidir.

Bunun yerine:

```text
Level
Progression
Reward
Economy
Gameplay
Tutorial
Purchase
Session
```

gibi generic kavramlar kullanılmalıdır.

---

# 65. Over-Engineering Kuralı

Analytics sistemi ilk versiyonda gereğinden fazla büyütülmemelidir.

Başlangıç için yeterli yapı:

```text
AnalyticsSystem
Event Contract
Queue
Batching
Provider
```

Daha sonra gerçek ihtiyaç oluştuğunda:

```text
Offline Storage
Multiple Providers
Advanced Routing
Remote Config
Alert Engine
```

gibi özellikler eklenebilir.

---

# 66. Definition of Done

Analytics özelliği tamamlanmış sayılabilmesi için:

* Event'in sahibi doğru sistem olmalı.
* AnalyticsSystem source of truth olmamalı.
* UI doğrudan analytics SDK'sına bağlanmamalı.
* Event contract stable olmalı.
* Common context merkezi olarak eklenmeli.
* Player ID pseudonymous olmalı.
* Session ID kullanılmalı.
* Analytics network request'i gameplay'i bloklamamalı.
* Event'ler queue/buffer üzerinden gönderilebilmeli.
* Batching desteklenmeli.
* Retry bounded olmalı.
* Duplicate event davranışı düşünülmeli.
* Purchase analytics MonetizationSystem'den gelmeli.
* Level analytics LevelSystem'den gelmeli.
* Tutorial analytics TutorialSystem'den gelmeli.
* Analytics failure gameplay'i bozmamalı.
* Telegram token client içine konulmamalı.
* Telegram analytics database olarak kullanılmamalı.
* Dashboard analytics backend üzerinden çalışmalı.
* Stuck/churn gibi kavramlar observable event'lerden türetilmeli.
* Gereksiz PII toplanmamalı.
* Development/Staging/Production ayrılmalı.
* Analytics provider abstraction yalnızca gerçek ihtiyaç varsa eklenmeli.
* EditMode/PlayMode testleri mümkün olduğunca eklenmeli.
* Hot path allocation'ları kontrol edilmeli.
* Generic template prensipleri korunmalı.

---

# 67. Sonuç

Analytics mimarisinin temel prensibi:

```text
Gameplay determines what happened.
Analytics records what happened.
Backend analyzes what happened.
Dashboard explains what happened.
Alert Engine decides what deserves attention.
Telegram delivers the notification.
```

Hiçbir analytics katmanı gameplay'in, economy'nin, progression'ın veya monetization'ın source of truth'u değildir.
