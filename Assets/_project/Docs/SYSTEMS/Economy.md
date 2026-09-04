# Economy System

## 1. Amaç

`EconomySystem`, oyundaki ekonomik değerlerin runtime state'ini ve ekonomik işlemlerin kurallarını yönetir.

Temel sorumlulukları:

* Currency tanımlarını kullanmak
* Runtime currency state'ini yönetmek
* Currency kazanma ve harcama işlemlerini doğrulamak
* Transaction sonuçlarını üretmek
* Reward uygulamasının economy kısmını gerçekleştirmek
* Ekonomik değişiklikleri ilgili sistemlere bildirmek
* Persistence için SaveSystem ile iletişim kurmak

`EconomySystem` UI, Shop veya Monetization sistemi değildir.

Temel akış:

```text id="e4m7qa"
Economy Configuration
        ↓
   EconomySystem
        ↓
 Runtime Currency State
        ↓
   Transaction
        ↓
 Economy Changed Event
        ↓
 SaveSystem / UI / Analytics / Other Consumers
```

---

# 2. Temel Prensip

Ekonomide tek bir authoritative runtime owner bulunmalıdır.

```text id="k8r3vc"
EconomySystem
    ↓
Currency State
```

Başka sistemler currency'nin kendi kopyasını source of truth olarak tutmamalıdır.

Örneğin:

```text id="p5m9xa"
TopBarHeader
    ✗ Currency State Owner

ShopView
    ✗ Currency State Owner

LevelSystem
    ✗ Currency State Owner

MonetizationSystem
    ✗ Currency State Owner

EconomySystem
    ✓ Currency State Owner
```

---

# 3. Configuration, Runtime ve Save

Economy üç ayrı katmana bölünür.

```text id="v7q4mc"
EconomyConfig
    ↓
Static Definitions
```

```text id="m3r8za"
EconomySystem
    ↓
Runtime Currency State
```

```text id="x6c2vp"
PlayerSaveData
    ↓
Persistent Currency State
```

Örneğin:

```text id="k9m4xa"
CurrencyConfig
    → soft currency definition

EconomySystem
    → current soft currency amount

EconomySaveData
    → persisted soft currency amount
```

---

# 4. Currency Definition

Currency türleri configuration üzerinden tanımlanabilir.

Örneğin:

```csharp id="q5r8vc"
[CreateAssetMenu]
public class CurrencyDefinition : ScriptableObject
{
    public string id;
    public Sprite icon;
    public int maxAmount;
}
```

Buradaki değerler static configuration'dır.

Runtime amount burada tutulmamalıdır.

---

# 5. Currency ID

Currency'ler stable ID ile tanımlanmalıdır.

Örneğin:

```text id="v4m7qa"
soft_currency
premium_currency
energy
```

ID:

* Save data
* Rewards
* Product definitions
* Analytics
* Gameplay results

gibi persistence veya communication gerektiren alanlarda kullanılabilir.

Asset object reference'ları persistence modeline doğrudan yazılmamalıdır.

---

# 6. Currency Runtime State

Runtime state örneği:

```text id="p8r3mc"
CurrencyId
CurrentAmount
```

Birden fazla currency varsa:

```text id="x7q4za"
Economy State
├── soft_currency → 1250
├── premium_currency → 100
└── energy → 5
```

Bu state'in owner'ı `EconomySystem`'dir.

---

# 7. Currency Read

Diğer sistemler currency miktarını `EconomySystem` üzerinden okuyabilir.

Örneğin:

```csharp id="m4v8qa"
var amount = economySystem.GetAmount(currencyId);
```

Bu, UI veya başka bir sistemin kendi currency state'ini oluşturmasından daha güvenlidir.

---

# 8. Currency Mutation

Currency değişikliği yalnızca EconomySystem üzerinden yapılmalıdır.

```text id="q6r2vc"
Add
Remove
Spend
Grant
Consume
```

gibi işlemler EconomySystem tarafından yönetilebilir.

Yanlış:

```text id="x8m3za"
shopView.currency += 100;
```

Doğru:

```text id="v5q7mc"
Shop / Reward / Gameplay
        ↓
EconomySystem
        ↓
Currency Change
```

---

# 9. Add ve Spend

Temel API mümkün olduğunca küçük tutulmalıdır.

Örneğin:

```csharp id="p3m8xa"
bool TryAdd(string currencyId, int amount);
bool TrySpend(string currencyId, int amount);
int GetAmount(string currencyId);
```

Project ihtiyacına göre transaction result daha zengin olabilir.

Gereksiz onlarca convenience method eklenmemelidir.

---

# 10. Negative Amount

Currency mutation sırasında invalid değerler engellenmelidir.

Örneğin:

```text id="k7r4vc"
TryAdd(currency, -100)
```

veya:

```text id="m9q2xa"
TrySpend(currency, -100)
```

gibi çağrılar domain açısından geçersiz olmalıdır.

Negative amount gerekiyorsa bunun semantiği açık bir operation ile ifade edilmelidir.

---

# 11. Zero Amount

Zero-value transaction'ların davranışı açık olmalıdır.

Genellikle:

```text id="v8m3qc"
Add 0
Spend 0
```

işlemleri state değiştirmeden başarılı veya no-op olarak ele alınabilir.

Ancak transaction event spam'i oluşturmamalıdır.

---

# 12. Maximum Amount

Currency için maksimum değer configuration'dan gelebilir.

Örneğin:

```text id="p6q8za"
Current = 950
Max = 1000
Add = 100
```

sonuç:

```text id="x4m7vc"
Current = 1000
```

olabilir.

Overflow veya clamp davranışı currency definition tarafından belirlenmelidir.

---

# 13. Integer Overflow

Currency miktarlarında overflow göz ardı edilmemelidir.

Özellikle:

```text id="k8r3ma"
Current + Reward
```

işleminde numeric sınırlar dikkate alınmalıdır.

Ekonomi state'i için uygun numeric type seçilmeli ve arithmetic validation yapılmalıdır.

---

# 14. TrySpend

Spend işlemi yeterli currency yoksa başarısız olmalıdır.

```text id="q5v9xc"
Current = 100

Spend = 150

Result = Failed
Current = 100
```

Başarısız işlem currency state'ini değiştirmemelidir.

---

# 15. Transaction Result

Önemli ekonomik işlemler yalnızca `bool` yerine anlamlı bir result döndürebilir.

Örneğin:

```csharp id="m7r4qa"
public enum EconomyTransactionResult
{
    Success,
    InvalidCurrency,
    InvalidAmount,
    InsufficientFunds,
    MaximumReached
}
```

Gereksiz ayrıntı eklenmeden domain ihtiyacına göre genişletilebilir.

---

# 16. Transaction Atomicity

Tek bir transaction mümkün olduğunca atomic olmalıdır.

Örneğin:

```text id="x3q8vc"
Spend 100
    ↓
Validation
    ↓
State Change
    ↓
Result
    ↓
Event
```

Validation başarısızsa state değişmemelidir.

---

# 17. Economy Event

Başarılı currency değişikliğinden sonra event yayınlanabilir.

Örneğin:

```text id="v6m2xa"
CurrencyChanged
```

Event:

```text id="p8r4qc"
CurrencyId
PreviousAmount
NewAmount
Delta
Reason
```

gibi alanlar taşıyabilir.

Event payload yalnızca ihtiyaç duyulan bilgileri içermelidir.

---

# 18. Event Source of Truth Değildir

`CurrencyChanged` event'i currency state değildir.

Doğru model:

```text id="k5q9va"
EconomySystem
    ↓
Currency State Changed
    ↓
CurrencyChanged Event
```

UI event'i kaçırırsa mevcut state'i tekrar EconomySystem'den okuyabilmelidir.

Bu nedenle UI için:

```text id="m7r3xc"
Initial Sync
+
Event Updates
```

yaklaşımı kullanılmalıdır.

---

# 19. Event Subscription

UI veya diğer event consumer'ları:

```text id="x8q4ma"
OnEnable
    ↓
Subscribe

OnDisable
    ↓
Unsubscribe
```

lifecycle'ını kullanmalıdır.

Static event kullanılıyorsa lifecycle ve cleanup özellikle dikkatle yönetilmelidir.

---

# 20. Reason

Currency değişikliğinin kaynağı gerektiğinde transaction reason ile taşınabilir.

Örneğin:

```text id="v3m7qa"
LevelReward
Purchase
DailyReward
GameplayReward
Refund
Admin
Debug
```

Ancak reason enum'u tüm oyun domain'lerini içine alan dev bir enum'a dönüştürülmemelidir.

Gerçek analytics veya domain ihtiyacı varsa kullanılmalıdır.

---

# 21. Reward System ile İlişki

`RewardSystem`, reward'ın ne olduğunu tanımlayan/orchestrate eden sistem olabilir.

Örneğin:

```text id="p5r8vc"
Reward
├── Currency
├── Amount
└── Other Reward
```

Akış:

```text id="k7m4xa"
RewardSystem
    ↓
EconomySystem
    ↓
Currency Added
```

RewardSystem currency state'ini kendisi değiştirmemelidir.

---

# 22. Gameplay ile İlişki

Gameplay sonucu currency kazanabilir.

Örneğin:

```text id="x4q9mc"
Gameplay
    ↓
Gameplay Result
    ↓
RewardSystem
    ↓
EconomySystem
```

Gameplay module:

```text id="m8r3va"
currency += 100
```

yapmamalıdır.

---

# 23. LevelSystem ile İlişki

Level completion reward üretebilir.

```text id="v6q2xa"
LevelSystem
    ↓
Level Result
    ↓
RewardSystem
    ↓
EconomySystem
```

LevelSystem currency hesabının owner'ı değildir.

---

# 24. Shop ile İlişki

ShopView currency state'ini hesaplamaz.

```text id="p9m4vc"
ShopView
    ↓
Purchase Intent
    ↓
Purchase Owner
    ↓
Reward
    ↓
EconomySystem
```

Shop ürün fiyatını UI içinde hesaplamamalıdır.

---

# 25. Monetization ile İlişki

Gerçek para ile satın alma sonucu EconomySystem'e ulaşabilir.

```text id="k3r8qa"
User
 ↓
ShopView
 ↓
Purchase Intent
 ↓
MonetizationSystem
 ↓
Purchase Success
 ↓
RewardSystem
 ↓
EconomySystem
```

MonetizationSystem:

* Store SDK
* IAP transaction
* Purchase validation

gibi konularla ilgilenir.

EconomySystem:

* Currency state
* Currency transaction

ile ilgilenir.

---

# 26. Purchase ve Economy Ayrımı

Şu iki işlem birbirinden ayrılmalıdır:

```text id="x7m2vc"
Purchase
```

ve:

```text id="q5r8za"
Reward
```

Purchase:

> Oyuncu bir ürün satın aldı mı?

Economy:

> Bu satın alma sonucunda hangi currency state değişti?

Bu ayrım ileride IAP, ads, offers veya remote products eklendiğinde sistemi temiz tutar.

---

# 27. TopBarHeader ile İlişki

TopBarHeader currency gösterir.

```text id="m8c4qa"
EconomySystem
    ↓
Currency State
    ↓
CurrencyChanged
    ↓
TopBarHeader
```

TopBarHeader:

* Currency hesaplamaz
* Currency değiştirmez
* Save data okumaz
* JSON okumaz

Sadece presentation yapar.

---

# 28. Currency Formatting

Currency value'ın presentation formatting'i EconomySystem'in zorunlu sorumluluğu değildir.

Örneğin:

```text
1250
```

UI'da:

```text
1.25K
```

olarak gösterilebilir.

Bu presentation concern'üdür.

Ancak format kuralları project-wide standardize edilmelidir.

---

# 29. Economy ve SaveSystem

EconomySystem state değiştiğinde persistence dirty olabilir.

```text id="v4q8mc"
Economy Changed
    ↓
SaveSystem
    ↓
PlayerSaveData
```

SaveSystem currency hesaplamaz.

EconomySystem JSON yazmaz.

---

# 30. Load

Bootstrap sırasında:

```text id="p7m3xa"
SaveSystem
    ↓
Load PlayerSaveData
    ↓
EconomySaveData
    ↓
EconomySystem.ApplyLoadedState()
```

uygulanabilir.

Loaded state validation'dan geçirilmelidir.

---

# 31. Load Validation

Save'den gelen currency değerleri güvenilir kabul edilmemelidir.

Kontroller:

```text id="x6r9vc"
Currency ID valid?
Amount valid?
Negative?
Overflow?
Maximum exceeded?
```

gibi kuralları içerebilir.

Invalid state kontrollü şekilde düzeltilebilir veya recovery mekanizmasına gönderilebilir.

---

# 32. Save Data

Örneğin:

```csharp id="m5q8za"
[Serializable]
public class EconomySaveData
{
    public List<CurrencySaveEntry> currencies;
}
```

ve:

```csharp id="v7r4mc"
[Serializable]
public class CurrencySaveEntry
{
    public string currencyId;
    public long amount;
}
```

kullanılabilir.

Exact model project ihtiyacına göre değişebilir.

---

# 33. Runtime Collection

Runtime state için dictionary benzeri hızlı lookup yapısı kullanılabilir:

```text id="k8m3qa"
CurrencyId
    ↓
Runtime Amount
```

Save serialization sırasında uygun DTO/model'e dönüştürülebilir.

Runtime collection ile save collection'ın aynı olmak zorunda olmadığı unutulmamalıdır.

---

# 34. Economy Configuration

Configuration içinde bulunabilecek alanlar:

```text id="p4q7vc"
Currency ID
Display Name
Icon
Maximum Amount
Initial Amount
Formatting Metadata
```

Bunlar static/designer data'dır.

Runtime amount burada bulunmamalıdır.

---

# 35. Initial Currency

İlk oyuncu save'i oluşturulurken:

```text id="x5m8za"
EconomyConfig
    ↓
Default Economy State
    ↓
PlayerSaveData
```

oluşturulabilir.

Ancak mevcut save yüklendiğinde config'teki initial amount oyuncunun mevcut currency'sinin üzerine yazılmamalıdır.

---

# 36. Currency Removal

Bir currency artık kullanılmıyorsa doğrudan silmek yerine migration strategy düşünülmelidir.

Örneğin:

```text id="v9r3mc"
Old Currency
    ↓
Migration
    ↓
New Currency
```

veya kontrollü retirement uygulanabilir.

Eski save'lerde bulunan unknown currency ID'leri bütün save'in bozulmasına neden olmamalıdır.

---

# 37. Currency Addition

Yeni currency eklemek genellikle migration gerektirmez.

Örneğin:

```text id="m7q4xa"
New Currency
```

configuration'a eklenebilir.

Oyuncunun mevcut save'inde bulunmuyorsa default runtime value:

```text id="k5r8vc"
0
```

veya configuration tarafından tanımlanan başlangıç değeri olabilir.

Bu davranış açıkça belirlenmelidir.

---

# 38. Multi-Currency Transactions

Bir işlem birden fazla currency içeriyorsa transaction bütünlüğü korunmalıdır.

Örneğin:

```text id="p8m3qa"
Soft Currency -100
Premium Currency +10
```

gibi bir işlem tek logical operation ise:

```text id="x6r7vc"
Validate All
    ↓
Apply All
```

yaklaşımı tercih edilmelidir.

İlk currency uygulanıp ikinci işlem başarısız olduğunda yarım transaction oluşmamalıdır.

---

# 39. Economy Transaction ve Event

Multi-currency transaction'da event granularity belirlenmelidir.

Seçenekler:

```text id="m4q8za"
CurrencyChanged
CurrencyChanged
```

veya:

```text id="v7r3mc"
EconomyTransactionCompleted
```

gibi bir aggregate event.

Hangisinin kullanılacağı consumer ihtiyaçlarına göre belirlenmelidir.

Gereksiz event çoğalmasından kaçınılmalıdır.

---

# 40. Batch Rewards

Birden fazla reward aynı anda verilecekse batch uygulama gerekebilir.

```text id="k9m2xa"
Reward Batch
    ↓
Validate
    ↓
Apply
    ↓
Single Logical Result
```

Özellikle level completion gibi çok sayıda reward'ın tek operation olarak uygulanması gerekiyorsa faydalıdır.

---

# 41. Economy and Events

EventBus kullanılabilir ancak her economy çağrısı EventBus üzerinden yapılmamalıdır.

Örneğin:

```text id="x5q8vc"
Shop
    ↓
EconomySystem.TrySpend()
```

gibi explicit dependency gayet geçerlidir.

EventBus daha çok:

```text id="m7r3za"
CurrencyChanged
    ↓
Multiple Independent Consumers
```

gibi decoupled communication için kullanılmalıdır.

---

# 42. Economy API

Önerilen API küçük tutulmalıdır.

Örneğin:

```csharp id="p4m8vc"
GetAmount(currencyId)

TryAdd(currencyId, amount)

TrySpend(currencyId, amount)

HasEnough(currencyId, amount)
```

Gerektiğinde:

```text id="v6q2xa"
ApplyReward
ApplyTransaction
```

eklenebilir.

Ancak aynı işi yapan çok sayıda method oluşturulmamalıdır.

---

# 43. HasEnough

`HasEnough` yalnızca query'dir.

```text id="k8r4ma"
HasEnough(currency, 100)
```

state değiştirmemelidir.

UI bunu satın alma öncesi görsel durum için kullanabilir.

Ancak gerçek spend sırasında tekrar validation yapılmalıdır.

---

# 44. Check-Then-Spend

Şu pattern güvenilir değildir:

```text id="x3m7vc"
if (HasEnough(100))
{
    Spend(100);
}
```

Çünkü arada başka bir transaction gerçekleşebilir.

Asıl mutation:

```text id="q5r8za"
TrySpend(100)
```

ile atomic şekilde doğrulanmalıdır.

`HasEnough` yalnızca informational/presentation amaçlıdır.

---

# 45. Thread Safety

Economy state normalde Unity main-thread domain state olarak düşünülebilir.

Background thread'den doğrudan mutation yapılmamalıdır.

Async/network sonuçları:

```text id="m8q4xa"
Background Result
    ↓
Main Thread / Domain Boundary
    ↓
EconomySystem
```

üzerinden uygulanmalıdır.

---

# 46. Performance

Economy operations genellikle sık çalışabilir.

Özellikle gameplay sırasında:

* Gereksiz allocation yapılmamalı
* LINQ kullanılmamalı
* Temporary collections oluşturulmamalı
* String formatting gereksiz yerde yapılmamalı
* Currency lookup optimize edilmeli
* Event payload allocation'ları kontrol edilmeli

Hot path'te GC pressure oluşturulmamalıdır.

---

# 47. Logging

Development ortamında ekonomik transaction'lar loglanabilir:

```text id="v3r7mc"
Currency Changed
CurrencyId = soft_currency
Delta = +100
Reason = LevelReward
```

Ancak yüksek frekanslı gameplay reward'larında production log spam oluşturulmamalıdır.

---

# 48. Analytics

AnalyticsSystem economy event'lerini tüketebilir.

Örneğin:

```text id="p6m2xa"
Currency Earned
Currency Spent
Reward Granted
Purchase Reward Applied
```

Analytics SDK çağrısı EconomySystem'in içine doğrudan gömülmemelidir.

Doğru:

```text id="k7q4vc"
Economy Event
    ↓
AnalyticsSystem
    ↓
Analytics SDK
```

---

# 49. Economy ve Audio

Currency değişimi ses üretebilir.

Ancak EconomySystem doğrudan Audio SDK veya UI audio implementation çağırmak zorunda değildir.

```text id="x8m3qa"
CurrencyChanged
    ↓
AudioSystem
```

gibi event consumer yaklaşımı kullanılabilir.

---

# 50. Economy and UI Feedback

Coin/currency kazanıldığında UI animasyonu gerekebilir.

```text id="v5r8mc"
EconomySystem
    ↓
CurrencyChanged
    ↓
TopBar / Floating Feedback
```

Animation economy state'in sahibi değildir.

UI feedback başarısız olsa bile economy transaction başarılı kalmalıdır.

---

# 51. Economy ve Tween

Currency değişikliğinin UI tween'i:

```text id="m4q7xa"
Presentation
```

katmanında olmalıdır.

EconomySystem:

```text id="k8r2vc"
Currency = 100
```

state değişikliğini animation completion'a bağlamamalıdır.

Örneğin:

```text
Tween Complete
    ↓
Add Currency
```

yerine:

```text
Add Currency
    ↓
Event
    ↓
Tween
```

daha güvenlidir.

---

# 52. Reward Failure

Reward uygulanırken economy transaction başarısız olabilir.

Örneğin:

```text id="p7m3za"
Currency Max Reached
```

durumunda RewardSystem sonucu doğru şekilde ele almalıdır.

EconomySystem sessizce başarısız olup reward'ın başarılı kabul edilmesine neden olmamalıdır.

---

# 53. Overflow Reward

Örneğin:

```text id="x6q8vc"
Current = 950
Max = 1000
Reward = 100
```

sonucun:

```text id="m5r3qa"
Current = 1000
Overflow = 50
```

olması gerekiyorsa overflow behavior açıkça tanımlanmalıdır.

Alternatifler:

```text
Clamp
Reject
Convert
Queue
```

game design'e göre seçilebilir.

---

# 54. Economy State Reset

Development veya test sırasında economy state resetlenebilir.

Production player reset ise SaveSystem üzerinden yönetilmelidir.

```text id="v8q4mc"
Reset Player Data
    ↓
SaveSystem
    ↓
Default State
    ↓
EconomySystem
```

EconomySystem tek başına player save file silmemelidir.

---

# 55. Economy and Tutorial

Tutorial currency gösterebilir veya reward verebilir.

Tutorial:

```text id="p3m7xa"
TutorialSystem
    ↓
RewardSystem
    ↓
EconomySystem
```

şeklinde çalışmalıdır.

Tutorial'ın kendi currency state'i olmamalıdır.

---

# 56. Economy and Progression

Progression reward verebilir.

```text id="k6r8vc"
ProgressionSystem
    ↓
RewardSystem
    ↓
EconomySystem
```

ProgressionSystem currency miktarını kendi state'inde tutmamalıdır.

---

# 57. Economy and Level

Level result reward'a dönüştürülebilir.

```text id="x4q9ma"
Level Result
    ↓
Reward Resolution
    ↓
EconomySystem
```

LevelSystem yalnızca:

```text id="m8v3qc"
Level Completed
```

sonucunu üretir.

Reward application ilgili reward/economy katmanına aittir.

---

# 58. Economy and Monetization

MonetizationSystem store purchase sonucunu üretir.

```text id="p5m7xa"
Store Purchase
    ↓
Purchase Result
    ↓
Reward Resolution
    ↓
EconomySystem
```

EconomySystem store SDK'sını bilmemelidir.

---

# 59. Economy and Ads

Rewarded ad sonucunda currency verilebilir.

```text id="v7q3mc"
Ad Completed
    ↓
MonetizationSystem
    ↓
RewardSystem
    ↓
EconomySystem
```

EconomySystem reklam SDK'sını çağırmaz.

---

# 60. Save Timing

Currency değiştiğinde SaveSystem dirty olabilir.

Ancak her transaction sonrası zorunlu synchronous disk write yapılması gerekmez.

Örneğin:

```text id="k4m8za"
10 Currency Changes
        ↓
Pending Save
        ↓
Single Save
```

kullanılabilir.

Critical transaction'larda daha agresif persistence tercih edilebilir.

---

# 61. Economy Consistency

Aşağıdaki invariant korunmalıdır:

```text id="x7r2vc"
EconomySystem Runtime State
        =
Current Authoritative Currency State
```

UI, SaveSystem, Analytics veya diğer consumer'ların tuttuğu değerler bunun kopyalarıdır.

---

# 62. Source of Truth

Currency için source of truth:

```text id="m9q4xa"
EconomySystem
```

Configuration:

```text id="v5r8mc"
CurrencyDefinition / EconomyConfig
```

Persistence:

```text id="k7m3za"
SaveSystem
```

Presentation:

```text id="p4q8vc"
UI
```

Analytics:

```text id="x6m2ra"
AnalyticsSystem
```

olarak ayrılmalıdır.

---

# 63. Generic Template Rule

EconomySystem generic kalmalıdır.

Core template'e:

```text
Star
Crown
Heart
Gem
Coin
Energy
```

gibi zorunlu game-specific kavramlar gömülmemelidir.

Bunun yerine:

```text
Currency
Reward
Cost
Transaction
```

gibi generic domain kavramları kullanılmalıdır.

---

# 64. Feature-Specific Economy

Bir oyun özel bir resource gerektiriyorsa bunu EconomySystem üzerinde generic currency olarak modelleyebilir.

Örneğin:

```text
Currency ID:
"event_token"
```

gibi.

Eğer resource ekonomi dışında farklı bir domain behavior gerektiriyorsa ayrı bir system oluşturulabilir.

Her sayısal değeri currency yapmak da doğru değildir.

---

# 65. Economy vs Score

Score ve currency aynı şey değildir.

```text id="q8m4va"
Score
    → Gameplay outcome

Currency
    → Persistent economy
```

Score yalnızca level/session sonucunda kullanılan geçici bir değer olabilir.

Currency ise persistent ekonomik state olabilir.

Bu nedenle `ScoreSystem` ile `EconomySystem` ayrılmalıdır.

---

# 66. Economy vs Progression

Currency:

```text id="m5r7xc"
Spendable Resource
```

Progression:

```text id="v8q3za"
Persistent Player Progress
```

olabilir.

Örneğin level unlock olmak için currency gerekebilir:

```text id="k4m9vc"
Progression Rule
    ↓
EconomySystem.HasEnough()
    ↓
EconomySystem.TrySpend()
    ↓
ProgressionSystem.Unlock()
```

İşlem sahipliği net tutulmalıdır.

---

# 67. Atomic Cross-System Operations

Bir operation hem currency hem progression state değiştiriyorsa orchestration owner açık olmalıdır.

Örneğin:

```text id="p7q2xa"
UpgradeSystem
    ↓
Validate
    ↓
EconomySystem.TrySpend()
    ↓
ProgressionSystem.ApplyUpgrade()
```

Bu durumda UpgradeSystem operation'ın orchestrator'ı olabilir.

EconomySystem yalnızca currency mutation'ını sahiplenir.

---

# 68. Do Not Build a God Economy Manager

`EconomySystem` içine:

```text id="x3m8vc"
Shop
IAP
Ads
Progression
Level
Tutorial
UI
Analytics
```

mantığı doldurulmamalıdır.

EconomySystem'in domain'i:

```text id="q6r4za"
Currency
Transaction
Economic State
```

ile sınırlı tutulmalıdır.

---

# 69. Definition of Done

EconomySystem implementation tamamlanmadan önce:

* [ ] Currency runtime state'inin tek owner'ı EconomySystem mi?
* [ ] Configuration runtime state'ten ayrılmış mı?
* [ ] Save data runtime state'ten ayrılmış mı?
* [ ] Currency stable ID kullanıyor mu?
* [ ] Currency mutation EconomySystem üzerinden mi?
* [ ] Negative amount validation var mı?
* [ ] Overflow kontrolü var mı?
* [ ] Maximum amount davranışı tanımlı mı?
* [ ] Insufficient funds durumu açık mı?
* [ ] Transaction sonucu anlamlı mı?
* [ ] Failed transaction state'i değiştirmiyor mu?
* [ ] Multi-currency transaction atomic mi?
* [ ] CurrencyChanged event'i gerekiyorsa kullanılıyor mu?
* [ ] Event source of truth olarak kullanılmıyor mu?
* [ ] UI initial sync + event update yapıyor mu?
* [ ] UI currency state sahibi değil mi?
* [ ] Shop currency hesaplamıyor mu?
* [ ] Monetization store SDK'sını EconomySystem çağırmıyor mu?
* [ ] Analytics SDK doğrudan EconomySystem'de değil mi?
* [ ] RewardSystem ile sorumluluklar ayrılmış mı?
* [ ] Gameplay currency'yi doğrudan mutate etmiyor mu?
* [ ] LevelSystem currency owner'ı değil mi?
* [ ] ProgressionSystem currency owner'ı değil mi?
* [ ] SaveSystem JSON'a EconomySystem yerine persistence boundary olarak hizmet ediyor mu?
* [ ] Save timing kontrollü mü?
* [ ] Her transaction'da gereksiz disk write yapılmıyor mu?
* [ ] Hot path allocation'ları kontrol edilmiş mi?
* [ ] Background thread doğrudan runtime state mutate etmiyor mu?
* [ ] Load edilen değerler validate ediliyor mu?
* [ ] Unknown currency ID kontrollü ele alınıyor mu?
* [ ] Currency removal için migration strategy düşünülmüş mü?
* [ ] Generic template'e game-specific currency kavramları gömülmemiş mi?
* [ ] Score ve currency ayrılmış mı?
* [ ] Gereksiz abstraction eklenmemiş mi?
* [ ] EconomySystem God Object'a dönüşmemiş mi?
* [ ] EditMode testleri kritik transaction logic'i kapsıyor mu?
* [ ] Mobile persistence davranışı dikkate alınmış mı?
