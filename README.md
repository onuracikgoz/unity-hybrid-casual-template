# Unity Hybrid-Casual Template

Reusable, scalable and data-driven **Unity Hybrid-Casual Game Template**.

Bu repository, farklı hybrid-casual oyun projelerinde tekrar kullanılabilecek ortak bir temel mimari sağlar.

Amaç tek bir oyunun mekaniklerini geliştirmek değil, farklı oyun fikirlerinin üzerine inşa edilebileceği:

* temiz
* modüler
* data-driven
* performans odaklı
* test edilebilir
* AI coding agent friendly

bir Unity proje altyapısı oluşturmaktır.

---

## 1. Project Goals

Bu template'in temel hedefleri:

* Gameplay ve system katmanlarını ayırmak
* UI'ı presentation katmanı olarak tutmak
* Runtime state için net ownership sağlamak
* Designer tarafından değiştirilebilir değerleri configuration üzerinden yönetmek
* Save, Economy, Progression, Monetization ve Analytics gibi ortak sistemleri ayrıştırmak
* Mobile performansını baştan dikkate almak
* Gereksiz abstraction ve over-engineering'den kaçınmak
* Yeni gameplay mekaniklerinin kolayca eklenebilmesini sağlamak
* AI coding agent'ların mimari kurallara uygun kod üretebilmesini sağlamak

Bu template belirli bir oyuna bağlı değildir.

Örneğin aşağıdaki kavramlar core architecture'a dahil edilmez:

```text
Kingdom
Building
Star
Decoration
Castle
```

Bunların yerine generic extension point'ler kullanılır:

```text
Progression
Reward
Economy
Upgrade
Content Unlock
Gameplay Module
Level
```

---

# 2. Architecture

Genel dependency yönü:

```text
UI
 ↓
GAMEPLAY
 ↓
SYSTEMS
 ↓
UNITY / PLATFORM
```

Daha detaylı yapı:

```text
                 ┌───────────────────┐
                 │        UI         │
                 │ Presentation      │
                 └─────────┬─────────┘
                           ↓
                 ┌───────────────────┐
                 │     GAMEPLAY      │
                 │ Level / Tutorial  │
                 │ Gameplay Modules  │
                 └─────────┬─────────┘
                           ↓
                 ┌───────────────────┐
                 │      SYSTEMS      │
                 │ Save              │
                 │ Economy           │
                 │ Progression       │
                 │ Monetization      │
                 │ Analytics         │
                 │ Audio             │
                 │ Pooling            │
                 │ Assets            │
                 └─────────┬─────────┘
                           ↓
                 ┌───────────────────┐
                 │ UNITY / PLATFORM  │
                 │ Unity APIs        │
                 │ Store SDK         │
                 │ Ad SDK            │
                 │ Analytics Provider│
                 └───────────────────┘
```

Her katmanın sorumluluğu mümkün olduğunca açık tutulur.

---

# 3. Core Architecture Principles

## Single Source of Truth

Her runtime state'in tek authoritative owner'ı vardır.

Örnek:

```text
EconomySystem
    ↓
Currency State
```

```text
LevelSystem
    ↓
Current Level / Level Session
```

```text
Gameplay Module
    ↓
Gameplay State
```

```text
ProgressionSystem
    ↓
Player Progression
```

UI bu state'lerin sahibi değildir.

---

## Configuration vs Runtime State

ScriptableObject'ler configuration ve static definition için kullanılır.

```text
ScriptableObject
      ↓
Configuration
```

Runtime state ayrı tutulur:

```text
Configuration
      ↓
System
      ↓
Runtime State
```

ScriptableObject üzerinde session sırasında değişen mutable gameplay state tutulmaz.

---

## Event-Driven Communication

EventBus her iletişim için kullanılmaz.

Tercih sırası:

```text
Direct Reference
      ↓
Local Callback / Event
      ↓
Shared Event System
```

Event gerçekten decoupled communication gerektiğinde kullanılır.

Event:

```text
"Bir şey oldu."
```

Command:

```text
"Lütfen bunu yap."
```

şeklinde düşünülmelidir.

---

## UI Is Presentation

UI:

* gameplay state sahibi değildir
* economy hesaplamaz
* reward hesaplamaz
* save data doğrudan değiştirmez
* SDK çağrısı yapmaz

Örneğin:

```text
ShopView
   ↓
Purchase Intent
   ↓
MonetizationSystem
   ↓
Purchase Result
   ↓
Reward / Economy
   ↓
ShopView
```

UI yalnızca sonucu gösterir.

---

# 4. Project Structure

```text
Assets/
└── _Project/
    │
    ├── Docs/
    │   ├── AGENTS.md
    │   ├── ARCHITECTURE.md
    │   ├── BOOTSTRAP.md
    │   │
    │   ├── SYSTEMS/
    │   │   ├── EventSystem.md
    │   │   ├── Pooling.md
    │   │   ├── SaveSystem.md
    │   │   ├── Economy.md
    │   │   ├── Progression.md
    │   │   ├── Monetization.md
    │   │   ├── Analytics.md
    │   │   ├── Audio.md
    │   │   └── AssetManagement.md
    │   │
    │   ├── GAMEPLAY/
    │   │   ├── LevelSystem.md
    │   │   ├── TutorialSystem.md
    │   │   └── GameplayModule.md
    │   │
    │   └── UI/
    │       ├── SplashAndLoading.md
    │       ├── TopBarHeader.md
    │       ├── BottomNavigationBar.md
    │       ├── SettingsPopup.md
    │       ├── UserProfilePopup.md
    │       └── ShopView.md
    │
    ├── Runtime/
    │   ├── Core/
    │   │   ├── Bootstrap/
    │   │   ├── GameFlow/
    │   │   ├── Events/
    │   │   ├── Time/
    │   │   └── Utilities/
    │   │
    │   ├── Systems/
    │   │   ├── Save/
    │   │   ├── Economy/
    │   │   ├── Progression/
    │   │   ├── Monetization/
    │   │   ├── Analytics/
    │   │   ├── Audio/
    │   │   ├── Pooling/
    │   │   └── Assets/
    │   │
    │   ├── Gameplay/
    │   │   ├── Level/
    │   │   ├── Tutorial/
    │   │   └── Modules/
    │   │
    │   └── UI/
    │       ├── Navigation/
    │       ├── Common/
    │       ├── Splash/
    │       ├── TopBar/
    │       ├── Settings/
    │       ├── Profile/
    │       └── Shop/
    │
    ├── Configs/
    │   ├── Game/
    │   ├── Gameplay/
    │   ├── Economy/
    │   ├── Progression/
    │   ├── UI/
    │   ├── Audio/
    │   └── Monetization/
    │
    ├── Data/
    │   ├── Definitions/
    │   ├── Save/
    │   └── Runtime/
    │
    ├── Scenes/
    │   ├── Bootstrap.unity
    │   ├── MainMenu.unity
    │   └── Gameplay.unity
    │
    ├── Prefabs/
    │   ├── UI/
    │   ├── Gameplay/
    │   ├── VFX/
    │   └── Common/
    │
    ├── Art/
    │   ├── UI/
    │   ├── Gameplay/
    │   ├── Characters/
    │   ├── Environment/
    │   └── VFX/
    │
    ├── Audio/
    │   ├── Music/
    │   ├── SFX/
    │   └── Voice/
    │
    ├── Addressables/
    │   ├── Gameplay/
    │   ├── UI/
    │   └── Remote/
    │
    └── Tests/
        ├── EditMode/
        └── PlayMode/
```

---

# 5. Runtime Flow

Oyunun genel lifecycle'ı:

```text
Application Start
       ↓
Bootstrap
       ↓
Core Systems Initialization
       ↓
Configuration Ready
       ↓
Persistent Data Load
       ↓
Runtime Systems Ready
       ↓
Game Flow Ready
       ↓
MainMenu
```

Gameplay:

```text
MainMenu
   ↓
Gameplay State
   ↓
LevelSystem.StartLevel()
   ↓
Gameplay Module
   ↓
Win / Lose
   ↓
Level Result
   ↓
Progression / Reward / Economy
```

---

# 6. Game Flow

Global game flow FSM tarafından yönetilir.

Temel state'ler:

```text
Boot
MainMenu
Gameplay
Pause
Win
Lose
```

GameFlow yalnızca global flow'dan sorumludur.

GameFlow:

* gameplay mechanic yönetmez
* economy yönetmez
* UI state yönetmez
* level configuration çözmez
* reward hesaplamaz

Örneğin:

```text
GameFlow
   ↓
Gameplay
   ↓
LevelSystem.StartLevel()
```

Level lifecycle LevelSystem'e aittir.

---

# 7. Bootstrap

Bootstrap uygulamanın başlangıç orchestration katmanıdır.

Temel akış:

```text
Bootstrap
   ↓
Core Systems
   ↓
Configuration
   ↓
Save
   ↓
Runtime Systems
   ↓
GameFlow
```

Bootstrap:

* initialization sırasını yönetir
* persistent sistemlerin hazırlanmasını sağlar
* duplicate initialization'ı önler
* failure/retry davranışını yönetebilir

Splash UI initialization logic'in sahibi değildir.

---

# 8. Gameplay Architecture

Gameplay generic bir extension point olarak tasarlanmıştır.

Input pipeline:

```text
Device Input
     ↓
Input Controller
     ↓
Gameplay Command
     ↓
Gameplay Module
     ↓
Gameplay State
     ↓
Gameplay Result
```

Aynı gameplay logic:

```text
Touch
Mouse
Keyboard
Gamepad
```

gibi farklı input kaynaklarıyla çalışabilmelidir.

Gameplay-specific mekanikler:

```text
Runtime/Gameplay/Modules/
```

altında bulunur.

---

# 9. Level System

LevelSystem:

* current level lifecycle
* level configuration
* level start
* restart
* cleanup
* win
* lose

sorumluluğunu taşır.

Örnek:

```text
GameFlow
   ↓
Gameplay
   ↓
LevelSystem
   ↓
Start Level
   ↓
Gameplay Module
```

LevelSystem gameplay mekaniklerinin kendisini implement etmez.

---

# 10. Save System

İlk versiyonda local save:

```text
JSON
+
Application.persistentDataPath
```

üzerinden yapılır.

Akış:

```text
Configuration
      ↓
System
      ↓
Runtime State
      ↓
SaveSystem
      ↓
JSON
      ↓
persistentDataPath
```

Feature sistemleri JSON'a doğrudan erişmez.

SaveSystem:

* persistence
* serialization
* versioning
* migration
* corruption recovery
* save/load lifecycle

sorumluluğunu taşır.

Save schema baştan versioned tasarlanır.

---

# 11. Economy

EconomySystem currency state'in sahibidir.

Minimal API:

```text
GetAmount()
TryAdd()
TrySpend()
HasEnough()
```

Özellikle `TrySpend()` atomic davranmalıdır.

Reward işlemi:

```text
RewardSystem
    ↓
EconomySystem
```

şeklinde çalışır.

UI currency hesaplamaz.

---

# 12. Progression

ProgressionSystem:

* player progression
* unlocks
* milestones
* progression rules

sorumluluğunu taşır.

LevelSystem:

```text
Level lifecycle
```

ProgressionSystem:

```text
Persistent progression
```

sahibidir.

Bu iki sorumluluk birbirine karıştırılmaz.

---

# 13. Monetization

MonetizationSystem:

* IAP
* purchase lifecycle
* entitlement
* rewarded ads
* interstitial ads
* provider SDK
* restore
* transaction recovery

sorumluluğunu taşır.

Shop yalnızca presentation ve purchase intent üretir.

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
```

Store SDK doğrudan UI'dan çağrılmaz.

Purchase state analytics'e bağlı değildir.

---

# 14. Analytics

AnalyticsSystem runtime davranışını ölçülebilir event'lere dönüştürür.

```text
Gameplay / Systems
        ↓
Analytics Events
        ↓
AnalyticsSystem
        ↓
Queue / Buffer
        ↓
Analytics Backend
        ↓
Dashboard
```

Önemli event'ler:

```text
SessionStarted
SessionEnded

LevelStarted
LevelCompleted
LevelFailed
LevelRestarted
LevelAbandoned

PurchaseStarted
PurchaseCompleted
PurchaseFailed

RewardedAdStarted
RewardedAdCompleted
RewardedAdFailed

TutorialStarted
TutorialCompleted
TutorialSkipped

AppBackgrounded
AppForegrounded
```

Analytics source of truth değildir.

---

## Analytics Dashboard

Dashboard ile aşağıdaki metrikler analiz edilebilir:

```text
DAU / WAU / MAU
D1 / D3 / D7 / D14 / D30 Retention
Session Duration
Level Completion Rate
Level Failure Rate
Level Abandonment
Attempts per Level
Level Dwell Time
Stuck Levels
Player Dropoff
Purchase Funnel
Rewarded Ad Funnel
Tutorial Funnel
```

Örneğin stuck player analizi:

```text
Player ID
Last Active
Current / Last Level
Attempts
Last Session Duration
Last Event
```

şeklinde incelenebilir.

---

# 15. Telegram Notifications

Telegram analytics database değildir.

Önerilen production flow:

```text
Unity
 ↓
Analytics Backend
 ↓
Alert Engine
 ↓
Telegram Notifier
 ↓
Telegram
```

Telegram'a yalnızca önemli alert'ler gönderilir.

Örneğin:

```text
Purchase Failure Spike
Level Failure Spike
Stuck Level Threshold
Analytics Backend Down
Crash / Error Spike
```

Bot token kesinlikle Unity client içine konulmaz.

---

# 16. Audio

AudioSystem:

* music
* SFX
* voice
* playback lifecycle

sorumluluğunu taşır.

SettingsSystem:

```text
Music Volume
SFX Volume
Voice Volume
Mute
```

gibi runtime settings'leri yönetebilir.

AudioSystem playback'in sahibidir.

UI doğrudan audio provider/SDK yönetmez.

---

# 17. Pooling

Sık oluşturulup yok edilen objeler için pooling kullanılır.

Örnek:

```text
VFX
Particles
Projectiles
Floating Numbers
Enemies
Collectibles
Tiles
Temporary UI
```

Pool object disable edildiğinde tamamen resetlenmelidir.

Reset kapsamı:

```text
Transform
Animation
Physics
Particles
Trails
Timers
Flags
Runtime Data
Subscriptions
Tweens
Coroutines
```

Pool'dan çıkan obje temiz bir runtime state ile başlamalıdır.

---

# 18. Asset Management

Asset integration chain:

```text
Asset
  ↓
Configuration
  ↓
Feature / System
  ↓
Runtime Usage
```

Örneğin:

```text
Art/UI/Logo.png
      ↓
SplashConfig
      ↓
SplashView
```

Asset loading ve asset selection birbirinden ayrıdır.

Küçük ve boot-critical asset'lerde direct serialized reference tercih edilir.

Büyük veya dynamic content için Addressables kullanılabilir.

`Resources` kullanımı sınırlı tutulmalıdır.

---

# 19. UI Architecture

UI presentation katmanıdır.

Temel UI alanları:

```text
Splash
TopBar
BottomNavigation
Settings
Profile
Shop
```

UI sistem state'lerini sahiplenmez.

Örneğin TopBar:

```text
EconomySystem
      ↓
Currency State
      ↓
Event
      ↓
TopBarHeader
      ↓
Currency Widget
```

Settings:

```text
SettingsPopup
      ↓
User Intent
      ↓
SettingsSystem
      ↓
Runtime Settings
```

Shop:

```text
ShopView
      ↓
Purchase Intent
      ↓
MonetizationSystem
```

---

# 20. Performance

Bu template mobile-first düşünülerek hazırlanmıştır.

Hot path'lerde mümkün olduğunca kaçınılması gerekenler:

```text
LINQ
Closures
Boxing
Temporary allocations
String concatenation
Reflection
Repeated GetComponent
GameObject.Find
FindObjectOfType
FindFirstObjectByType
FindAnyObjectByType
Instantiate
Destroy
```

özellikle:

```text
Update()
FixedUpdate()
LateUpdate()
```

ve yüksek frekanslı gameplay/input/collision/spawn işlemlerinde dikkat edilmelidir.

Reference'lar cache edilmelidir.

---

# 21. Unity Lifecycle

Genel lifecycle kuralı:

```text
Awake
  ↓
Local initialization / reference cache

OnEnable
  ↓
Subscribe

Start
  ↓
Dependency-based initialization

OnDisable
  ↓
Unsubscribe

OnDestroy
  ↓
Cleanup
```

Event subscription yapıldıysa lifecycle boyunca güvenli şekilde unsubscribe edilmelidir.

---

# 22. Async / Timing

Projede belirlenmiş async/timing yaklaşımı tutarlı kullanılmalıdır.

Aynı sistem içerisinde rastgele:

```text
Coroutine
Task
async/await
UniTask
```

karıştırılmamalıdır.

Uzun yaşayan async işlemler:

* owner
* cancellation
* lifetime
* failure handling

tanımına sahip olmalıdır.

---

# 23. Tweening

Tek bir established tween sistemi kullanılmalıdır.

Tween:

```text
OnEnable
   ↓
Start
```

ile oluşturuluyorsa:

```text
OnDisable
   ↓
Kill / Reset
```

yapılmalıdır.

Gameplay state yalnızca tween completion callback'ine bağlı olmamalıdır.

---

# 24. Testing

Testler iki ana kategoriye ayrılır:

```text
Tests/
├── EditMode/
└── PlayMode/
```

### EditMode

Scene gerektirmeyen deterministic logic:

```text
Economy
Progression
Rewards
Level Rules
Serialization
Migration
Analytics Contracts
```

### PlayMode

Unity integration:

```text
Lifecycle
Bootstrap
Scene Flow
Pooling
UI Integration
System Integration
```

Mümkün olduğunda gameplay logic scene'den bağımsız test edilebilir olmalıdır.

---

# 25. Documentation

Projenin architecture documentation'ı:

```text
Assets/_Project/Docs/
```

altındadır.

Ana doküman:

```text
Docs/ARCHITECTURE.md
```

AI coding agent kuralları:

```text
Docs/AGENTS.md
```

Bootstrap:

```text
Docs/BOOTSTRAP.md
```

Feature/system detayları ilgili dokümanda tutulmalıdır.

Örneğin:

```text
Docs/SYSTEMS/SaveSystem.md
Docs/SYSTEMS/Economy.md
Docs/SYSTEMS/Analytics.md
```

---

# 26. AI Coding Agent Guidelines

AI coding agent herhangi bir değişiklik yapmadan önce:

1. İlgili documentation'ı okumalı.
2. Mevcut implementation'ı incelemeli.
3. Runtime state owner'ını belirlemeli.
4. Configuration ile runtime state'i ayırmalı.
5. Mevcut event communication modelini kontrol etmeli.
6. Serialization etkisini değerlendirmeli.
7. Hot path performance etkisini kontrol etmeli.
8. Gereksiz abstraction eklememeli.
9. İlgisiz dosyalara dokunmamalı.
10. Mümkünse test eklemeli.

Öncelik sırası:

```text
Correctness
    ↓
Existing Architecture
    ↓
Serialization Safety
    ↓
Clear Ownership
    ↓
Data-Driven Configuration
    ↓
Decoupled Communication
    ↓
Performance
    ↓
Simplicity
```

Detaylı kurallar:

```text
Assets/_Project/Docs/AGENTS.md
```

dosyasındadır.

---

# 27. Adding a New Feature

Yeni bir feature eklerken şu akış kullanılmalıdır:

```text
1. Requirement
      ↓
2. Identify Owner
      ↓
3. Identify Runtime State
      ↓
4. Identify Configuration
      ↓
5. Define Communication
      ↓
6. Implement
      ↓
7. Test
      ↓
8. Update Documentation
```

Örneğin yeni bir gameplay mechanic:

```text
Gameplay Module
      ↓
Gameplay State
      ↓
Gameplay Result
      ↓
Progression / Reward / Economy
```

Yeni bir UI:

```text
UI
 ↓
User Intent
 ↓
Relevant System
 ↓
Runtime State
 ↓
UI Update
```

---

# 28. Adding Configuration

Designer tarafından değiştirilebilecek değerler mümkün olduğunca configuration üzerinden tanımlanmalıdır.

Örneğin:

```text
LevelDefinition
CurrencyConfig
AudioConfig
MonetizationConfig
SplashConfig
```

Akış:

```text
ScriptableObject
      ↓
System
      ↓
Runtime State
```

Configuration session state'in yerine kullanılmamalıdır.

---

# 29. Serialization Safety

Unity serialized data bir contract olarak kabul edilir.

Dikkat edilmesi gerekenler:

* serialized field isimleri
* serialized field tipleri
* enum değerleri
* ScriptableObject yapıları
* prefab dependencies
* save schema

Serialized field değişikliklerinde gerektiğinde:

```csharp
[FormerlySerializedAs("oldFieldName")]
```

kullanılmalıdır.

Save data version migration desteklemelidir.

---

# 30. Over-Engineering Policy

Bu template modülerdir ancak framework değildir.

Her problem için:

```text
Interface
Factory
Service Locator
Dependency Container
Generic Manager
Event
Wrapper
```

eklenmez.

Abstraction yalnızca gerçek bir ihtiyaç varsa eklenir.

Küçük ve açık bir implementation, gereksiz abstraction katmanlarından daha değerlidir.

---

# 31. Genericity

Bu repository bir oyun değil, oyunların üzerine kurulacağı bir template'tir.

Core architecture:

```text
Generic
Reusable
Content-Agnostic
```

olmalıdır.

Game-specific code:

```text
Runtime/Gameplay/Modules/
```

ve ilgili configuration/data alanlarında tutulmalıdır.

Core sistemlere belirli bir oyunun domain terminology'si sızdırılmamalıdır.

---

# 32. Definition of Done

Bir feature tamamlanmadan önce:

* Responsibility doğru mu?
* Runtime state'in owner'ı belli mi?
* Configuration ayrılmış mı?
* UI source of truth olmuş mu?
* Event subscription temizleniyor mu?
* Pool reset uygulanıyor mu?
* Hot path allocation var mı?
* Hot path Instantiate/Destroy var mı?
* Gereksiz Find/GetComponent kullanılıyor mu?
* SaveSystem dışında persistence var mı?
* SDK çağrısı yanlış katmanda mı?
* Serialization etkisi kontrol edildi mi?
* Mobile performance değerlendirildi mi?
* Test eklenebiliyor mu?
* İlgili documentation güncellendi mi?
* Gereksiz abstraction eklendi mi?
* İlgisiz kod değiştirildi mi?

kontrol edilmelidir.

---

# 33. Documentation Map

```text
Docs/
│
├── AGENTS.md
│   └── AI coding rules
│
├── ARCHITECTURE.md
│   └── Genel architecture
│
├── BOOTSTRAP.md
│   └── Startup / initialization
│
├── SYSTEMS/
│   ├── EventSystem.md
│   ├── Pooling.md
│   ├── SaveSystem.md
│   ├── Economy.md
│   ├── Progression.md
│   ├── Monetization.md
│   ├── Analytics.md
│   ├── Audio.md
│   └── AssetManagement.md
│
├── GAMEPLAY/
│   ├── LevelSystem.md
│   ├── TutorialSystem.md
│   └── GameplayModule.md
│
└── UI/
    ├── SplashAndLoading.md
    ├── TopBarHeader.md
    ├── BottomNavigationBar.md
    ├── SettingsPopup.md
    ├── UserProfilePopup.md
    └── ShopView.md
```

---

# 34. Quick Start

Unity projesini açtıktan sonra:

```text
Bootstrap.unity
      ↓
Bootstrap
      ↓
Core Systems
      ↓
MainMenu
```

başlangıç noktasıdır.

Gameplay:

```text
MainMenu
      ↓
Gameplay.unity
      ↓
GameFlow
      ↓
LevelSystem
      ↓
Gameplay Module
```

şeklinde ilerler.

---

# 35. Recommended Development Order

Yeni bir oyun bu template üzerine kurulurken önerilen sıra:

```text
1. Core Bootstrap
2. GameFlow
3. SaveSystem
4. Economy
5. Progression
6. LevelSystem
7. Gameplay Module
8. Tutorial
9. UI
10. Monetization
11. Analytics
12. Audio
13. Content / Polish
```

Gameplay mechanic'in gerektirdiği özel sistemler bu sıraya göre değişebilir.

---

# 36. Repository Philosophy

Bu template'in temel felsefesi:

```text
Simple where possible.
Modular where necessary.
Data-driven where useful.
Decoupled where valuable.
Performant by default.
Explicit ownership everywhere.
```

Amaç maksimum abstraction değil, **minimum gerekli complexity ile uzun süre yaşayabilecek bir architecture** oluşturmaktır.

---

# 37. Status

Bu repository bir reusable foundation olarak tasarlanmıştır.

Core architecture:

```text
[✓] Project Structure
[✓] Documentation Structure
[✓] Bootstrap Architecture
[✓] GameFlow
[✓] Event System
[✓] Pooling
[✓] Save System
[✓] Economy
[✓] Progression
[✓] Monetization
[✓] Analytics
[✓] Audio
[✓] Asset Management
[✓] Level System
[✓] Tutorial System
[✓] Gameplay Module
[✓] UI Architecture
```

Game-specific gameplay mechanics bu template'in üzerine eklenir.

---

# 38. Final Principle

Bu projenin en önemli kuralı:

```text
Know who owns the state.
Know who changes the state.
Know who only observes the state.
```

Bir feature eklerken önce kod yazmak yerine:

```text
"Bu state'in sahibi kim?"
```

sorusu cevaplanmalıdır.

Owner belli olduktan sonra:

```text
Configuration
      ↓
System / Gameplay
      ↓
Runtime State
      ↓
Event
      ↓
Presentation / Analytics / Persistence
```

akışı mümkün olduğunca açık tutulmalıdır.

Bu yaklaşım template'in farklı oyunlara uyarlanmasını, ekip içinde anlaşılmasını ve AI coding agent'ların güvenli değişiklik yapmasını kolaylaştırır.
