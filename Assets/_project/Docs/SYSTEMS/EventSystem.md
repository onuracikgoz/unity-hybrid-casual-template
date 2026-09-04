# Event System

## 1. Amaç

`EventSystem`, sistemler ve feature'lar arasında gerektiğinde decoupled communication sağlamak için kullanılır.

Amaç:

* Sistemler arasındaki gereksiz doğrudan bağımlılıkları azaltmak
* Domain olaylarını ilgili sistemlere bildirmek
* UI ve presentation katmanlarını runtime state owner'larından ayırmak
* Gameplay, progression, economy, analytics gibi sistemlerin gerektiğinde birbirinden bağımsız çalışmasını sağlamak
* Event lifecycle ve subscription kurallarını standartlaştırmak

`EventSystem`, bütün communication'ın zorunlu merkezi değildir.

> Event kullanmak bir mimari hedef değil, bir communication aracıdır.

---

# 2. Temel Prensip

Communication için öncelik:

```text id="c7m4xp"
Direct Reference
      ↓
Local Callback / Event
      ↓
Shared Event System
```

En basit ve sorumluluğu en açık çözüm tercih edilmelidir.

Bir component başka bir component'in açıkça sahibi veya doğrudan dependency'si ise EventBus kullanmak gereksiz olabilir.

---

# 3. Event System'in Yeri

Genel mimaride:

```text id="r8q2va"
┌───────────────┐
│      UI       │
└───────┬───────┘
        │
        │ Events / Intent
        ↓
┌───────────────┐
│   Gameplay    │
└───────┬───────┘
        │
        │ Domain Events
        ↓
┌───────────────┐
│    Systems    │
└───────┬───────┘
        │
        ↓
┌───────────────┐
│ Unity/Platform│
└───────────────┘
```

Event system bu katmanlar arasındaki gerektiğinde decoupled communication'ı destekler.

---

# 4. Event Nedir?

Event, gerçekleşmiş anlamlı bir olayı ifade eder.

Örneğin:

```text id="m5k9zc"
LevelStarted
LevelCompleted
LevelFailed
CurrencyChanged
PurchaseCompleted
ObjectiveCompleted
TutorialCompleted
SettingsChanged
```

Event:

```text id="v3p7xa"
Something happened.
```

anlamına gelir.

Command ise:

```text id="q8r4mn"
Please do something.
```

anlamına gelir.

Bu ikisi birbirine karıştırılmamalıdır.

---

# 5. Event ve Command Ayrımı

Örneğin:

```text id="x6m2vc"
PurchaseProduct
```

bir command/intention olabilir.

```text id="p9k4za"
PurchaseCompleted
```

bir event'tir.

Akış:

```text id="r7c3xm"
User
 ↓
Purchase Command
 ↓
MonetizationSystem
 ↓
Purchase Completed
 ↓
PurchaseCompleted Event
```

UI event'i dinleyebilir.

Ancak UI satın alma işlemini EventBus üzerinden "yapılmasını istemek" yerine ilgili purchase owner'a açık intent gönderebilir.

---

# 6. Event Ownership

Event'in bir producer'ı ve gerektiğinde birden fazla consumer'ı olabilir.

Örneğin:

```text id="k4n8qa"
EconomySystem
      ↓
CurrencyChanged
      ↓
 ┌────┴─────────────┐
 ↓                  ↓
TopBarHeader    AnalyticsSystem
```

`EconomySystem` currency state'in owner'ıdır.

TopBarHeader currency state'in owner'ı değildir.

---

# 7. Event State'in Owner'ı Değildir

Event yalnızca state değişikliğini bildirir.

Yanlış:

```text id="v2m7xc"
CurrencyChangedEvent
    ↓
Currency = 500
```

Event payload'ı state'in kendisinin ikinci kopyasına dönüşmemelidir.

Doğru:

```text id="q5r9za"
EconomySystem
    ↓
Currency State
    ↓
CurrencyChanged
```

EconomySystem source of truth olarak kalır.

---

# 8. Event Payload

Event gerektiğinde olayı anlamak için yeterli bilgi taşıyabilir.

Örneğin:

```csharp id="n8c4vp"
public readonly struct CurrencyChangedEvent
{
    public readonly CurrencyType CurrencyType;
    public readonly int PreviousAmount;
    public readonly int CurrentAmount;
}
```

Ancak event payload gereksiz şekilde büyük tutulmamalıdır.

Şunu yapmak yerine:

```text id="m3x7qa"
CurrencyChangedEvent
    ├── PlayerSaveData
    ├── EconomyConfig
    ├── UI References
    ├── Audio References
    └── Gameplay References
```

event yalnızca olayın anlaşılması için gereken data'yı taşımalıdır.

---

# 9. Event Payload ve Allocation

Gameplay hot path'lerinde event allocation'a dikkat edilmelidir.

Sık yayınlanan event'lerde uygun olduğunda:

```csharp id="p7r2mx"
readonly struct
```

gibi value type event'ler kullanılabilir.

Ancak bunu her event için otomatik kural haline getirmemek gerekir.

Önemli olan:

* Allocation pattern
* Frequency
* Event lifetime
* Subscription model

birlikte değerlendirilmelidir.

---

# 10. EventBus

Shared EventBus, sistemlerin birbirlerinin concrete implementation'larını bilmeden event yayınlamasına veya dinlemesine olanak sağlar.

Genel model:

```text id="c9m4xa"
Producer
   ↓
EventBus
   ↓
Consumer A
Consumer B
Consumer C
```

Örneğin:

```text id="w6q8vp"
LevelSystem
    ↓
LevelCompleted
    ↓
EventBus
    ├── ProgressionSystem
    ├── AnalyticsSystem
    └── UI
```

---

# 11. EventBus Ne Zaman Kullanılmalı?

EventBus özellikle şu durumlarda uygundur:

* Bir event'in birden fazla bağımsız consumer'ı varsa
* Producer consumer'ların kim olduğunu bilmemeliyse
* Feature'lar arasında düşük coupling isteniyorsa
* Analytics gibi cross-cutting sistemler birden fazla domain event'ini dinliyorsa
* UI presentation'ın gameplay/system owner'ından ayrılması gerekiyorsa

---

# 12. EventBus Ne Zaman Kullanılmamalı?

Şu durumlarda EventBus gereksiz olabilir:

* A ve B arasında açık ve basit bir ownership ilişkisi varsa
* Tek bir consumer varsa
* Method çağrısı daha okunabilir ve doğrudansa
* Event yalnızca bir dependency'yi gizlemek için kullanılıyorsa
* Debugging'i gereksiz zorlaştırıyorsa

Örneğin:

```text id="r4n7kc"
GameplayModule
    ↓
ObjectiveSystem
```

açık bir dependency ise:

```csharp id="x8m2qa"
objectiveSystem.UpdateProgress();
```

EventBus'ten daha anlaşılır olabilir.

---

# 13. UI Event Kullanımı

UI çoğunlukla system event'lerini dinleyebilir.

Örneğin:

```text id="v5c9xm"
EconomySystem
      ↓
CurrencyChanged
      ↓
TopBarHeader
```

UI:

* Event'i dinler
* Görsel state'i günceller
* Gerekirse animation oynatır

UI:

* Economy state'in sahibi olmaz
* Save data değiştirmez
* Reward hesaplamaz

---

# 14. Initial State Sync

Event subscription tek başına yeterli değildir.

Örneğin TopBarHeader açıldığında:

```text id="q7m3za"
Subscribe
   ↓
Wait for CurrencyChanged
```

yaklaşımı mevcut currency state'i daha önce değişmişse UI'ı yanlış bırakabilir.

Bunun yerine:

```text id="n4x8vc"
OnEnable
   ↓
Subscribe
   ↓
Read Current State
   ↓
Render
```

ve sonrasında:

```text id="p6r2qa"
CurrencyChanged
   ↓
Refresh
```

kullanılmalıdır.

---

# 15. Subscription Lifecycle

Unity component'leri için temel kural:

```text id="m8c4vx"
OnEnable
    ↓
Subscribe

OnDisable
    ↓
Unsubscribe
```

Örneğin:

```csharp id="r3q7na"
private void OnEnable()
{
    eventBus.Subscribe<CurrencyChangedEvent>(OnCurrencyChanged);
}

private void OnDisable()
{
    eventBus.Unsubscribe<CurrencyChangedEvent>(OnCurrencyChanged);
}
```

Event subscription `Start()` içinde yapılıp cleanup'ın unutulması tercih edilmez.

---

# 16. OnDestroy

`OnDestroy` gerektiğinde ek cleanup için kullanılabilir.

Özellikle:

* Global/static event source
* Native/plugin callback
* Persistent subscription
* External resource

gibi lifecycle'ı Unity GameObject lifecycle'ından farklı olan kaynaklarda dikkat edilmelidir.

Normal component event subscription için `OnEnable` / `OnDisable` pattern'i tercih edilir.

---

# 17. Static Event Kullanımı

Static event'ler dikkatli kullanılmalıdır.

Örneğin:

```csharp id="k6v2px"
public static event Action SomethingChanged;
```

kullanımı:

* Global lifecycle'ı belirsizleştirebilir
* Scene değişimlerinde subscription leak oluşturabilir
* Test isolation'ı zorlaştırabilir
* Play Mode restart davranışını karmaşıklaştırabilir

Bu nedenle static event ancak gerçekten global lifecycle ihtiyacı varsa kullanılmalıdır.

---

# 18. Event Leak

Unsubscribe edilmeyen event listener:

```text id="x9m4qc"
Destroyed UI
      ↓
Still subscribed
      ↓
Event fired
      ↓
Invalid callback / memory reference / error
```

oluşturabilir.

Özellikle:

* UI popup
* Temporary gameplay object
* Pooled object
* Scene-specific object

için subscription lifecycle dikkatle yönetilmelidir.

---

# 19. Pooled Object ve Event

Pooled object'ler EventBus subscription açısından özellikle risklidir.

Örneğin:

```text id="v7r3ma"
Pool Spawn
    ↓
Subscribe
    ↓
Pool Release
    ↓
Unsubscribe
```

Object pool'a döndüğünde subscription açık kalmamalıdır.

Aksi halde aynı object tekrar spawn edildiğinde:

```text id="q5n8xc"
1 callback
2 callback
3 callback
...
```

gibi duplicate notification oluşabilir.

---

# 20. Duplicate Subscription

Bir object'in aynı event'e birden fazla kez subscribe olması engellenmelidir.

Riskli lifecycle:

```text id="m4x7qa"
OnEnable
    ↓
Subscribe
    ↓
OnDisable
    ↓
Forgot Unsubscribe
    ↓
OnEnable
    ↓
Subscribe Again
```

Bu durumda tek event birden fazla kez işlenebilir.

---

# 21. Event Order

Event sisteminde consumer execution order'a gereksiz şekilde bağımlı olunmamalıdır.

Örneğin:

```text id="c8q2va"
Event
 ↓
Consumer A
 ↓
Consumer B
```

şeklinde belirli bir sıraya güvenmek kırılgan olabilir.

Eğer bir işlem diğerinden sonra kesinlikle gerçekleşmeliyse bu ilişki event zinciri yerine açık orchestration ile modellenmelidir.

Örneğin:

```text id="p7m4xc"
Purchase
 ↓
Purchase Validation
 ↓
Apply Reward
 ↓
Publish PurchaseCompleted
```

gibi.

---

# 22. Event İçinde Event

Bir event handler'ın başka event'ler yayınlaması mümkündür.

Örneğin:

```text id="n3r8qa"
LevelCompleted
      ↓
ProgressionSystem
      ↓
ProgressionChanged
```

Ancak event zincirleri çok derinleşmemelidir.

Kötü örnek:

```text id="x6m2vp"
A
 ↓
B
 ↓
C
 ↓
D
 ↓
E
 ↓
F
```

Bu durumda sistem davranışını takip etmek zorlaşır.

---

# 23. Event Recursion

Event handler'ın doğrudan veya dolaylı olarak aynı event'i tekrar yayınlaması recursion oluşturabilir.

Örneğin:

```text id="q8v5mc"
CurrencyChanged
    ↓
Handler
    ↓
Modify Currency
    ↓
CurrencyChanged
```

Bu tür zincirler açıkça kontrol edilmelidir.

State update sırasında event publish edilen yerler iyi tanımlanmalıdır.

---

# 24. Event ve State Mutation

Event handler içinde state mutation yapılabilir, ancak event'in semantiği net olmalıdır.

Örneğin:

```text id="r4m7xa"
LevelCompleted
    ↓
ProgressionSystem
    ↓
Update Progression
```

anlamlıdır.

Ancak:

```text id="c9x2vp"
LevelCompleted
    ↓
Random unrelated state mutations
```

gibi yan etkiler event'in anlamını belirsizleştirir.

---

# 25. Domain Event

Domain event, gameplay veya system açısından anlamlı bir olayı ifade eder.

Örneğin:

```text id="v8m3qa"
LevelStarted
LevelCompleted
ObjectiveCompleted
CurrencyChanged
PurchaseCompleted
ContentUnlocked
TutorialCompleted
```

Bunlar sistemler arası communication için değerlidir.

---

# 26. Presentation Event

Bazı event'ler presentation-specific olabilir.

Örneğin:

```text id="p5q8xc"
ShowRewardAnimation
PlayCelebration
```

Bunların global EventBus'e konulması genellikle gereksizdir.

Presentation-local callback veya direct reference daha uygun olabilir.

EventBus'e domain event koymak, UI animation komutlarını global broadcast etmekten daha sağlıklıdır.

---

# 27. Event Naming

Event isimleri geçmiş zamanda veya gerçekleşmiş olay şeklinde adlandırılmalıdır.

İyi:

```text id="m7r4za"
LevelStarted
LevelCompleted
CurrencyChanged
PurchaseCompleted
EnemyDefeated
ObjectiveCompleted
```

Kötü:

```text id="x3n8vc"
StartLevel
ChangeCurrency
DoPurchase
ShowWin
UpdateUI
```

İkinci grup command veya presentation action anlamına daha yakındır.

---

# 28. Event Payload Naming

Payload property'leri event'in anlamını açıkça ifade etmelidir.

Örneğin:

```csharp id="q6m2xp"
public readonly struct LevelCompletedEvent
{
    public readonly int LevelId;
    public readonly int Score;
    public readonly LevelResult Result;
}
```

Event içinde global mutable object referansları taşımaktan kaçınılmalıdır.

---

# 29. Event Granularity

Event'ler çok genel veya çok küçük olmamalıdır.

Aşırı genel:

```text id="v4r8mc"
GameChanged
```

Bu event consumer'ların ne olduğunu anlamasını zorlaştırır.

Aşırı küçük:

```text id="n7q3xa"
PlayerNameCharacterChanged
PlayerNameLengthChanged
PlayerNameValidationChanged
```

gibi gereksiz event'ler de communication graph'ını büyütür.

Anlamlı domain event'ler tercih edilmelidir.

---

# 30. Event Bus API

EventBus API mümkün olduğunca küçük tutulmalıdır.

Örneğin konsept olarak:

```csharp id="k8p4vz"
Subscribe<TEvent>()
Unsubscribe<TEvent>()
Publish<TEvent>()
```

yeterli olabilir.

EventBus içine:

```text id="r2m7xa"
Dependency Injection
Save Logic
Gameplay Logic
UI Logic
Analytics Logic
```

gibi unrelated responsibility'ler eklenmemelidir.

---

# 31. Event Bus ve Dependency Injection

EventBus bir dependency injection container değildir.

Şunlar EventBus üzerinden yapılmamalıdır:

```text id="c5q9vm"
Resolve EconomySystem
Resolve SaveSystem
Resolve LevelSystem
```

EventBus'in görevi event communication'dır.

Dependency resolution ayrı bir sorumluluktur.

---

# 32. Event Bus ve GameFlow

GameFlow global state owner'ıdır.

EventBus GameFlow'un yerine geçmez.

Yanlış:

```text id="x7m3qa"
Publish GameplayEvent
    ↓
Some Listener
    ↓
Change Global Game State
```

GameFlow state transition'larının sahibi olarak kalmalıdır.

Örneğin:

```text id="p8r4vc"
LevelSystem
    ↓
LevelCompleted
    ↓
GameFlow
    ↓
Win State
```

Bu kullanılabilir.

Ancak global flow'un her küçük adımını EventBus'e bırakmak tercih edilmez.

---

# 33. Event Bus ve SaveSystem

SaveSystem domain event'lerini dinleyebilir.

Örneğin:

```text id="m4q7xa"
LevelCompleted
    ↓
Save-related progression update
    ↓
SaveSystem
```

Ancak SaveSystem her event'i otomatik olarak kaydetmemelidir.

Persistence kararı ilgili state owner tarafından belirlenmelidir.

---

# 34. Event Bus ve Analytics

Analytics event consumer olarak EventBus'i kullanmak için güçlü bir adaydır.

Örneğin:

```text id="v6c2mp"
LevelCompleted
        ↓
EventBus
        ↓
AnalyticsSystem
```

Böylece gameplay code'un analytics SDK'sına doğrudan dependency'si olmaz.

AnalyticsSystem event'i kendi analytics event modeline dönüştürür.

---

# 35. Event Bus ve Audio

AudioSystem de domain event'lerini dinleyebilir.

Örneğin:

```text id="q3n8za"
ObjectiveCompleted
       ↓
AudioSystem
       ↓
Play Completion SFX
```

Ancak event'in doğrudan audio implementation detayına bağımlı hale gelmemesi gerekir.

---

# 36. Event Bus ve Tutorial

TutorialSystem gameplay event'lerini dinleyebilir.

Örneğin:

```text id="r8m4xc"
EnemyDefeated
      ↓
TutorialSystem
      ↓
Advance Tutorial Step
```

Tutorial gameplay mechanics'i yeniden implement etmez.

Event yalnızca gerçekleşen gameplay sonucunu bildirir.

---

# 37. Event Bus ve UI

UI event consumer olabilir:

```text id="k7p2va"
EconomySystem
    ↓
CurrencyChanged
    ↓
TopBarHeader
```

ve:

```text id="m9x4qc"
ProgressionSystem
    ↓
ProgressionChanged
    ↓
ProfilePopup
```

UI'nın event yayınlaması da mümkündür, ancak bunun domain command'i gizleyen global bir mekanizmaya dönüşmemesine dikkat edilmelidir.

---

# 38. Event Threading

Unity gameplay event'leri varsayılan olarak Unity main thread üzerinde çalışmalıdır.

Background thread'den Unity object'lerine doğrudan event callback gönderilmemelidir.

Async/background işlem sonucunda:

```text id="v5r8mx"
Background Operation
      ↓
Main Thread
      ↓
Event Publish
```

gerekiyorsa proje standardına uygun şekilde main thread'e dönülmelidir.

---

# 39. Event ve Async

Async operation tamamlandığında event yayınlanabilir.

Örneğin:

```text id="p4c7za"
Asset Load
    ↓
Load Completed
    ↓
AssetLoaded
```

Ancak operation owner'ı lifecycle'ı kontrol etmelidir.

Object artık aktif değilse callback'in güvenli şekilde ignore/cancel edilmesi gerekir.

---

# 40. Error Events

Her exception için global event yayınlamak gerekli değildir.

Örneğin:

```text id="x8m3vq"
NullReferenceException
    ↓
GlobalErrorEvent
```

gibi bir pattern tercih edilmemelidir.

Beklenen domain failure'lar event olabilir:

```text id="q6r2ma"
PurchaseFailed
LoadFailed
InitializationFailed
```

Beklenmeyen programming error'lar ise normal error handling/logging mekanizmasına bırakılmalıdır.

---

# 41. Event Logging

Development/debug ortamında event flow'u takip etmek faydalı olabilir.

Örneğin:

```text id="m7v4xc"
[Event] LevelCompleted
[Event] ProgressionChanged
[Event] CurrencyChanged
```

Ancak production'da yüksek frekanslı event'lerin gereksiz logging üretmesine izin verilmemelidir.

Logging sistemi event system'in içine gömülmemelidir.

---

# 42. Event Debugging

EventBus debugging zorlaşıyorsa aşağıdaki bilgiler yararlı olabilir:

```text id="r3q8za"
Event Type
Publisher
Subscriber
Subscription Count
Publish Count
```

Bu bilgiler development/debug tooling için kullanılabilir.

Ancak runtime event payload'larını gereksiz şekilde loglamak performansı etkileyebilir.

---

# 43. Testability

EventBus test edilebilir olmalıdır.

Testlerde:

```text id="p8m4vc"
Publish Event
    ↓
Expected Consumer
    ↓
Expected State Change
```

doğrulanabilir.

Ayrıca test sonunda subscription state temizlenmelidir.

Static/global event state testler arasında taşınmamalıdır.

---

# 44. EventBus Lifecycle

EventBus'in kendisinin lifecycle'ı açık olmalıdır.

Eğer persistent bir EventBus kullanılıyorsa:

```text id="q5x9ma"
Bootstrap
    ↓
Create EventBus
    ↓
Persistent Lifetime
```

gibi bir model uygulanabilir.

Scene-specific EventBus gerekiyorsa scene lifecycle'a bağlı tutulabilir.

Tek bir global EventBus oluşturmak her durumda doğru değildir.

---

# 45. Global ve Local Event Bus

İhtiyaca göre:

```text id="v7c2nx"
Global EventBus
```

veya:

```text id="m4q8za"
Feature / Scene Local Event Bus
```

kullanılabilir.

Global event bus yalnızca gerçekten global communication gerektiğinde kullanılmalıdır.

Gameplay-specific event'lerin tamamını global bus'a taşımak gerekli değildir.

---

# 46. Event System ve Architecture Boundaries

EventBus architecture boundary'lerini kaldırmaz.

Örneğin:

```text id="x3r7mc"
UI
 ↓
EventBus
 ↓
SaveSystem
```

kullanılıyor diye UI SaveSystem'in responsibility'sini kazanmaz.

Event yalnızca communication mekanizmasıdır.

Ownership kuralları değişmez.

---

# 47. Source of Truth

Event sistemi hiçbir zaman source of truth değildir.

Source of truth ilgili state owner'dır.

Örneğin:

```text id="k8m3qa"
Currency
    → EconomySystem

Progression
    → ProgressionSystem

Current Level
    → LevelSystem

Settings
    → SettingsSystem

Gameplay Board
    → BoardController
```

Event yalnızca state değişikliğini duyurur.

---

# 48. Anti-Pattern: Everything Through EventBus

Aşağıdaki yaklaşım kullanılmamalıdır:

```text id="r6q2vp"
Everything
    ↓
EventBus
    ↓
Everything Else
```

Bu yaklaşım zamanla:

* Debugging'i zorlaştırır
* Execution order belirsizleştirir
* Dependency graph'ını görünmez yapar
* Event sayısını gereksiz artırır
* State ownership'i belirsizleştirir

EventBus yalnızca gerçekten decoupling sağladığında kullanılmalıdır.

---

# 49. Anti-Pattern: Event as Method Replacement

Şu pattern gereksizdir:

```text id="m9c4xa"
System A
    ↓
EventBus
    ↓
System B
```

eğer aslında:

```text id="v7p2mz"
System A
    ↓
System B.Method()
```

çok daha açık ve güvenliyse.

EventBus direct reference'ın sadece "daha modern" versiyonu değildir.

---

# 50. Anti-Pattern: UI Command Through Global Event

Örneğin:

```text id="q4n8vc"
Shop Button
    ↓
Global EventBus
    ↓
BuyProductCommand
    ↓
MonetizationSystem
```

yerine, açık bir ownership/dependency varsa:

```text id="x6m3ra"
ShopView
    ↓
Purchase Intent
    ↓
Shop / Purchase Owner
```

daha anlaşılır olabilir.

Global EventBus ancak gerçekten producer'ın consumer'ı bilmemesi gereken bir durumda tercih edilmelidir.

---

# 51. Recommended Communication Matrix

| Communication                   | Tercih                            |
| ------------------------------- | --------------------------------- |
| Child → Owner                   | Direct Reference                  |
| Owner → Child                   | Direct Reference                  |
| Local feature interaction       | Direct Reference / Callback       |
| One-to-one explicit dependency  | Direct Reference                  |
| One-to-many domain notification | Event                             |
| UI state update                 | Event + Initial Sync              |
| Analytics notification          | Event                             |
| Audio reaction                  | Event                             |
| Tutorial reaction               | Event                             |
| Gameplay command                | Direct command / Input pipeline   |
| Global GameFlow transition      | GameFlow API / gerektiğinde event |
| Save persistence                | State owner + SaveSystem          |
| SDK callback                    | Relevant System                   |

---

# 52. Event Documentation

Yeni bir event eklenirken şu sorular cevaplanmalıdır:

```text id="n8r4vc"
Event neden gerekli?
Kim publish ediyor?
Kimler subscribe ediyor?
Payload ne taşıyor?
State owner kim?
Direct reference neden yeterli değil?
Lifecycle nasıl yönetiliyor?
Frequency nedir?
Hot path'te mi?
```

Event yalnızca "iki class birbirini görmesin" amacıyla eklenmemelidir.

---

# 53. Event Checklist

Yeni event eklemeden önce:

* [ ] Gerçek bir domain olayı mı?
* [ ] Command ile event ayrımı doğru mu?
* [ ] State owner belli mi?
* [ ] Direct reference yeterli değil mi?
* [ ] Birden fazla consumer var mı veya producer consumer'ı bilmemeli mi?
* [ ] Event payload minimum gerekli bilgiyi taşıyor mu?
* [ ] Event adı gerçekleşmiş olayı ifade ediyor mu?
* [ ] Subscription lifecycle güvenli mi?
* [ ] `OnEnable` / `OnDisable` kullanılıyor mu?
* [ ] Pooled object subscription'ı temizleniyor mu?
* [ ] Duplicate subscription riski yok mu?
* [ ] Event order'a gereksiz bağımlılık yok mu?
* [ ] Recursive event zinciri yok mu?
* [ ] Hot path allocation kontrol edildi mi?
* [ ] Global EventBus gerçekten gerekli mi?
* [ ] Test edilebilir mi?
* [ ] Event source of truth haline gelmemiş mi?

---

# 54. Definition of Done

EventSystem kullanımı tamamlanmış sayılmadan önce:

* [ ] Event'in amacı açıkça tanımlanmış mı?
* [ ] Producer belli mi?
* [ ] Consumer'lar belli mi?
* [ ] State ownership korunuyor mu?
* [ ] Direct reference değerlendirildi mi?
* [ ] Gereksiz EventBus kullanımı yok mu?
* [ ] Event/Command ayrımı doğru mu?
* [ ] Payload minimum tutulmuş mu?
* [ ] Subscription lifecycle güvenli mi?
* [ ] Unsubscribe garanti edilmiş mi?
* [ ] Pooled object'lerde subscription temizleniyor mu?
* [ ] Duplicate subscription riski kontrol edilmiş mi?
* [ ] Event recursion riski kontrol edilmiş mi?
* [ ] Execution order'a kırılgan dependency yok mu?
* [ ] Hot path allocation değerlendirilmiş mi?
* [ ] Static event gereksiz kullanılmamış mı?
* [ ] Global EventBus gerçekten gerekli mi?
* [ ] UI source of truth haline gelmemiş mi?
* [ ] EventBus dependency injection container gibi kullanılmıyor mu?
* [ ] Testlerde event state'i izole edilebiliyor mu?
* [ ] Generic template sınırları korunuyor mu?
