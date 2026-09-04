# ARCHITECTURE.md

Bu doküman, Unity Hybrid-Casual Template'in genel mimari yapısını tanımlar.

Amaç; farklı oyun türleri, gameplay mekanikleri ve içerikler üzerinde tekrar kullanılabilecek, açık sorumluluklara sahip, düşük coupling'li ve data-driven bir temel oluşturmaktır.

Bu doküman **hangi sistemlerin bulunduğunu, sorumluluklarını ve birbirleriyle nasıl ilişkili olduklarını** tanımlar.

Detaylı implementation kuralları ilgili `SYSTEMS/`, `GAMEPLAY/` ve `UI/` dokümanlarında bulunur.

---

# 1. Architectural Goals

Template aşağıdaki hedeflere göre tasarlanmalıdır:

* Farklı hybrid-casual oyun türlerine uyarlanabilmek.
* Gameplay kodunu infrastructure kodundan ayırmak.
* UI ile gameplay state'ini birbirinden ayırmak.
* Runtime state için tek authoritative owner kullanmak.
* Designer-tunable değerleri data-driven hale getirmek.
* Sistemler arası gereksiz coupling'i azaltmak.
* Mobile platformlarda performanslı çalışmak.
* Gameplay logic'i mümkün olduğunca test edilebilir tutmak.
* Yeni feature eklerken mevcut sistemleri gereksiz yere değiştirmemek.
* Oyun içeriği değiştiğinde core architecture'ın değişmesini gerektirmemek.

Template belirli bir oyunun domain'ine bağlı olmamalıdır.

Örneğin aşağıdaki gibi oyun-spesifik sistemler core template'in parçası olmamalıdır:

```text id="m0a8y2"
KingdomManager
StarManager
BuildingSystem
DecorationSystem
```

Bunların yerine daha genel kavramlar kullanılmalıdır:

```text id="5v0q9x"
ProgressionSystem
RewardSystem
EconomySystem
UpgradeSystem
ContentUnlockSystem
LevelSystem
```

Oyun özelindeki sistemler gerektiğinde `GAMEPLAY/` altında oluşturulabilir.

---

# 2. High-Level Architecture

Template aşağıdaki ana katmanlardan oluşur:

```text id="h4xk7m"
┌─────────────────────────────────────────────┐
│                    UI                       │
│   Views / Popups / Navigation / HUD         │
└──────────────────────┬──────────────────────┘
                       │
                       │ Presentation / Commands
                       ▼
┌─────────────────────────────────────────────┐
│                 GAMEPLAY                    │
│ Gameplay Modules / Level / Tutorial         │
└──────────────────────┬──────────────────────┘
                       │
                       │ Domain Operations
                       ▼
┌─────────────────────────────────────────────┐
│                  SYSTEMS                    │
│ Economy / Save / Progression / Audio /      │
│ Monetization / Analytics / Pooling / Events │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│             UNITY / PLATFORM               │
│ Unity API / Input / Addressables / SDKs     │
└─────────────────────────────────────────────┘
```

Bu katmanlar katı bir inheritance hierarchy değildir.

Amaç sorumlulukların ayrılmasıdır.

Bir feature gerektiğinde birden fazla katmana dokunabilir. Ancak her katman kendi sorumluluğunu korumalıdır.

---

# 3. Core Architectural Concepts

Template'in temel mimari prensipleri:

```text id="1p3j6n"
Game Flow
    ↓
Systems
    ↓
Gameplay
    ↓
Presentation
```

Ancak gerçek dependency yönü feature'a göre değişebilir.

Temel prensip:

> Bir sistem yalnızca ihtiyaç duyduğu abstraction veya public API'ye bağımlı olmalı, başka sistemlerin internal state'ine erişmemelidir.

---

# 4. System Categories

Sistemler genel olarak iki kategoriye ayrılır.

## Core / Infrastructure Systems

Bunlar gameplay'den bağımsız veya minimum gameplay bilgisine sahip sistemlerdir.

Örnek:

```text id="j6k4g9"
GameFlow
EventSystem
Pooling
SaveSystem
EconomySystem
ProgressionSystem
MonetizationSystem
AnalyticsSystem
AudioSystem
```

Bu sistemlerin amacı farklı gameplay modüllerinin tekrar kullanabileceği ortak altyapıyı sağlamaktır.

## Gameplay Systems

Gameplay'e özgü davranışları yönetir.

Örnek:

```text id="xj9s7c"
LevelSystem
TutorialSystem
GameplayModule
```

Gerçek projede oyunun ihtiyacına göre yeni gameplay sistemleri eklenebilir.

Core template'i gereksiz şekilde oyun-spesifik hale getirme.

---

# 5. Game Flow

Global game flow state-based bir yapı ile yönetilir.

Temel state'ler:

```text id="j7r3m8"
Boot
MainMenu
Gameplay
Pause
Win
Lose
```

Genel flow:

```text id="x0e4a7"
Boot
  ↓
MainMenu
  ↓
Gameplay
  ├── Pause
  │     ↓
  │   Gameplay
  │
  ├── Win
  │
  └── Lose
```

Game flow'un sahibi `GameFlow` / ilgili flow controller'dır.

`GameManager` kullanılıyorsa yalnızca global orchestration responsibility taşımalıdır.

Game flow sistemi:

* Level mechanics yönetmez.
* Board state yönetmez.
* Currency yönetmez.
* UI presentation yönetmez.
* Reward calculation yapmaz.

Detaylar:

`SYSTEMS/GameFlow.md`

---

# 6. Boot and Initialization

Oyunun başlangıç süreci ayrı bir initialization flow olarak ele alınmalıdır.

Genel olarak:

```text id="q8m2c1"
Application Start
      ↓
Bootstrap
      ↓
Core Systems Initialization
      ↓
Persistent Data Load
      ↓
Configuration Ready
      ↓
Game Flow Ready
      ↓
MainMenu
```

Initialization sırası açık ve deterministik olmalıdır.

Bir sistem başka bir sistemin hazır olmasına ihtiyaç duyuyorsa dependency açık şekilde tanımlanmalıdır.

Bootstrap içerisinde gameplay-specific initialization mümkün olduğunca yapılmamalıdır.

Detaylar:

`BOOTSTRAP.md`

---

# 7. Runtime State Ownership

Her runtime state'in tek authoritative owner'ı bulunmalıdır.

Örnek:

```text id="v5x1qz"
Score
→ ScoreSystem

Currency
→ EconomySystem

Progression
→ ProgressionSystem

Current Level
→ LevelSystem / LevelManager

Board State
→ BoardController

Tutorial State
→ TutorialSystem
```

Diğer sistemler bu state'i kopyalayıp kendi authoritative state'lerini oluşturmamalıdır.

Örneğin:

```text id="4d9p2v"
EconomySystem
→ owns currency

ShopView
→ displays currency

ShopView
→ does NOT own currency
```

Bu ayrım tüm architecture için geçerlidir.

---

# 8. Configuration and Data

Template üç farklı data kategorisini birbirinden ayırır.

```text id="f6u2n8"
Configuration
    ↓
ScriptableObject / Serialized Configuration

Runtime State
    ↓
Runtime System / Controller / Model

Persistent Data
    ↓
Save Data Model
```

Örneğin:

```text id="r3c9k1"
EconomyConfig
→ Currency definitions
→ Reward configuration
→ Limits

EconomySystem
→ Current currency
→ Transactions

PlayerSaveData
→ Persistent currency
```

Configuration ile runtime state aynı obje içerisinde tutulmamalıdır.

---

# 9. Event-Driven Communication

Sistemler arasında loose coupling gerektiğinde EventBus / EventManager kullanılabilir.

Örnek:

```text id="w8y3m2"
Gameplay System
      │
      │ PlayerWon
      ▼
Event System
      │
      ├── UI
      ├── Audio
      ├── VFX
      ├── Analytics
      └── Progression
```

Event system'in amacı:

* Global veya cross-system communication
* Presentation notification
* Decoupled system communication

olmalıdır.

EventBus her method call'un yerine geçmez.

Aşağıdaki durumda direct reference daha uygun olabilir:

```text id="m2v7q0"
Controller
    ↓
Owned Component
```

Ownership açık ve lifecycle local ise event kullanmak gereksiz abstraction oluşturabilir.

Detaylar:

`SYSTEMS/EventSystem.md`

---

# 10. Gameplay Architecture

Gameplay modülleri template'in değişken bölümüdür.

Core systems mümkün olduğunca gameplay'den bağımsız tutulurken, gameplay module oyunun temel mekaniklerini yönetir.

Örnek:

```text id="t1p8w4"
GameplayModule
├── BoardController
├── InputController
├── Gameplay Rules
├── Score / Objective Logic
└── Gameplay-specific Components
```

Bu yapı örnektir.

Her oyun aynı component'leri kullanmak zorunda değildir.

Örneğin bir oyun board-based olabilirken başka bir oyun runner veya idle mechanic kullanabilir.

Bu nedenle template gameplay architecture'ı **extension point** olarak tasarlanmalıdır.

Detaylar:

`GAMEPLAY/GameplayModule.md`

---

# 11. Level Architecture

Level sistemi global game flow'dan ayrıdır.

Game Flow:

```text id="b2n6j4"
Gameplay State
```

Level System:

```text id="e7q4w1"
Current Level
Level Configuration
Level Loading
Level Completion
Level Progression
```

Örnek flow:

```text id="a6k9r2"
GameFlow
    ↓
Gameplay State
    ↓
LevelSystem
    ↓
GameplayModule
```

LevelSystem gameplay mechanic'in kendisi değildir.

Örneğin board movement, enemy AI veya puzzle rules LevelSystem içerisinde tutulmamalıdır.

Detaylar:

`GAMEPLAY/LevelSystem.md`

---

# 12. UI Architecture

UI presentation layer'dır.

UI sistemleri:

* State gösterir.
* User input alır.
* Command başlatabilir.
* System sonuçlarını sunar.

UI gameplay state'in authoritative owner'ı değildir.

Genel flow:

```text id="r8w1c5"
User Input
    ↓
UI
    ↓
System / Gameplay Command
    ↓
System
    ↓
State Change
    ↓
UI Update
```

Örnek:

```text id="x5m0q7"
ShopView
    ↓
Purchase Request
    ↓
Economy / Monetization
    ↓
Purchase Result
    ↓
ShopView Update
```

UI kendi içinde fiyat, reward veya currency hesaplamamalıdır.

Detaylar:

`UI/*.md`

---

# 13. Input Architecture

Input source ile gameplay logic birbirinden ayrılmalıdır.

```text id="p3k8v2"
Touch / Mouse / Keyboard / Gamepad
                ↓
        Input Controller
                ↓
       Gameplay Command
                ↓
        Gameplay System
```

Input implementation gameplay rules'e doğrudan gömülmemelidir.

Bu sayede aynı gameplay logic farklı platformlarda kullanılabilir.

---

# 14. Economy Architecture

Economy sistemi currency ve transaction state'inin sahibidir.

Genel flow:

```text id="u6m2x9"
Gameplay / UI
      ↓
Reward / Purchase Request
      ↓
EconomySystem
      ↓
Currency State
      ↓
SaveSystem
```

UI currency değerini hesaplamaz.

Gameplay component'leri currency değerini doğrudan değiştirmemelidir.

Economy configuration ve runtime state ayrılmalıdır.

Detaylar:

`SYSTEMS/Economy.md`

---

# 15. Progression Architecture

ProgressionSystem oyuncunun persistent veya meta progression state'ini yönetir.

Örnek:

```text id="q4r8n1"
Level Progress
Unlocks
Upgrades
Milestones
Player Progress
```

Progression, gameplay state ile aynı şey değildir.

Örneğin:

```text id="p8k1w6"
Current Gameplay Level
≠
Persistent Player Progression
```

Gameplay sırasında oluşan sonuçlar progression sistemine aktarılabilir.

---

# 16. Save Architecture

SaveSystem persistent data'nın okunması ve yazılmasından sorumludur.

Genel ilişki:

```text id="j3v7m5"
Runtime Systems
      ↓
PlayerSaveData
      ↓
SaveSystem
      ↓
Persistent Storage
```

SaveSystem gameplay logic'in sahibi değildir.

SaveSystem:

* Gameplay rules hesaplamaz.
* Economy rules hesaplamaz.
* UI state yönetmez.
* Progression logic'in tamamını içermez.

Persistent data modeli ayrı tutulmalıdır.

Detaylar:

`SYSTEMS/SaveSystem.md`

---

# 17. Monetization Architecture

MonetizationSystem external monetization SDK'larını izole eder.

Örnek:

```text id="z4c9m1"
UI / Gameplay
      ↓
Monetization Request
      ↓
MonetizationSystem
      ↓
Ad / IAP SDK
      ↓
Result
```

UI doğrudan SDK ile konuşmamalıdır.

Economy ve monetization ilişkili olabilir ancak aynı responsibility değildir.

```text id="s7x2p5"
Monetization
→ "Purchase / Ad completed"

Economy
→ "Grant currency / reward"
```

Detaylar:

`SYSTEMS/Monetization.md`

---

# 18. Analytics Architecture

AnalyticsSystem analytics provider / SDK communication'ını izole eder.

Gameplay veya UI component'leri doğrudan analytics SDK çağırmamalıdır.

Örnek:

```text id="k8n3v6"
Gameplay Event
      ↓
AnalyticsSystem
      ↓
Analytics Provider
```

Analytics implementation provider-specific olmamalıdır.

Provider değiştiğinde gameplay kodunun değişmesi minimum seviyede kalmalıdır.

Detaylar:

`SYSTEMS/Analytics.md`

---

# 19. Audio Architecture

AudioSystem audio playback ve audio state'in sorumluluğunu taşır.

Gameplay sistemleri:

```text id="m9q4x2"
"Player Won"
```

gibi bir domain event oluşturabilir.

AudioSystem:

```text id="v1c7k5"
"Play Win Sound"
```

gibi presentation davranışını yönetir.

Gameplay component'lerinin audio implementation detaylarını bilmesine gerek olmamalıdır.

Detaylar:

`SYSTEMS/Audio.md`

---

# 20. Object Pooling Architecture

Sık oluşturulup yok edilen runtime object'ler pooling üzerinden yönetilebilir.

Örnek:

```text id="y5r2m8"
Gameplay
    ↓
Pool Request
    ↓
Pool
    ↓
Pooled Object
```

Pooling bir gameplay mechanic değildir.

Infrastructure system olarak çalışır.

Pool lifecycle:

```text id="f3w8q1"
Get
 ↓
Initialize / Reset
 ↓
Use
 ↓
Release
 ↓
Reset
 ↓
Available
```

Pool'dan çıkan object temiz bir runtime state ile başlamalıdır.

Detaylar:

`SYSTEMS/Pooling.md`

---

# 21. Performance Architecture

Performance kuralları tüm katmanlar için geçerlidir.

Özellikle mobile cihazlarda:

* Hot path allocation azaltılmalı.
* Gereksiz `Instantiate` / `Destroy` kullanılmamalı.
* Sık kullanılan object'ler pool edilmeli.
* Repeated `Find` kullanılmamalı.
* Component reference'ları cache edilmeli.
* Gereksiz LINQ ve temporary collections kullanılmamalı.

Performance optimizasyonu architecture'ın sonradan eklenecek bir katmanı değildir.

Sistem tasarımının bir parçasıdır.

---

# 22. Testing Architecture

Pure logic mümkün olduğunca Unity scene lifecycle'ından ayrılmalıdır.

Genel yaklaşım:

```text id="u2m7x9"
Pure Logic
    ↓
EditMode Tests

Unity Integration
    ↓
PlayMode Tests
```

Gameplay rules mümkün olduğunca deterministic ve test edilebilir olmalıdır.

Ancak her class için yapay abstraction oluşturup test edilebilirliği zorla artırma.

Testability ile simplicity arasında denge korunmalıdır.

---

# 23. Serialization Boundaries

Unity serialization architecture'ın önemli bir parçasıdır.

Aşağıdaki yapılar serialization contract olarak kabul edilmelidir:

```text id="c5v9n2"
MonoBehaviour serialized fields
ScriptableObject fields
Prefab references
Scene references
Enum values
Save data
```

Bu yapılarda yapılan değişiklikler mevcut asset'leri etkileyebilir.

Architecture değişikliği yapılırken yalnızca runtime kodu değil, serialized asset dependency'leri de değerlendirilmelidir.

---

# 24. Folder Responsibilities

Template'in temel klasör yapısı:

```text id="r7m3x1"
Assets/_Project/
│
├── Docs/
│   ├── AGENTS.md
│   ├── ARCHITECTURE.md
│   ├── BOOTSTRAP.md
│   ├── SYSTEMS/
│   ├── GAMEPLAY/
│   └── UI/
│
├── Scripts/
│   ├── Core/
│   ├── Systems/
│   ├── Gameplay/
│   └── UI/
│
├── Prefabs/
├── ScriptableObjects/
├── Scenes/
├── Audio/
├── VFX/
└── ...
```

Folder yapısı projeye göre genişletilebilir.

Ancak klasörler responsibility boundary olarak kullanılmalı, yalnızca dosya depolamak için rastgele kategoriler oluşturulmamalıdır.

---

# 25. Feature Dependency Model

Yeni bir feature aşağıdaki şekilde tasarlanmalıdır:

```text id="n4c8y2"
Feature
  │
  ├── Configuration
  │       ↓
  │   ScriptableObject
  │
  ├── Runtime Logic
  │       ↓
  │   System / Gameplay Module
  │
  ├── Presentation
  │       ↓
  │   UI / VFX / Audio
  │
  └── Persistence
          ↓
      SaveSystem
```

Feature'ın tüm sorumluluklarını tek bir `Manager` içine toplama.

Örneğin bir Shop feature:

```text id="q7m1v5"
ShopConfig
ShopView
EconomySystem
MonetizationSystem
SaveSystem
```

olarak birden fazla mevcut sistemle çalışabilir.

`ShopManager` oluşturup tüm bu sorumlulukları tek yerde toplamak varsayılan yaklaşım değildir.

---

# 26. Communication Decision

Sistemler arası iletişimde şu karar sırası kullanılmalıdır:

```text id="x8p4k2"
1. Local ownership var mı?
        ↓
      Evet
        ↓
   Direct reference

2. Cross-system communication mı?
        ↓
      Evet
        ↓
   Public API / Command / Event

3. Communication gerçekten decoupled olmalı mı?
        ↓
      Evet
        ↓
       Event
```

Her iletişimi EventBus'a taşımak architecture'ı otomatik olarak daha iyi hale getirmez.

Amaç **minimum gerekli coupling** olmalıdır.

---

# 27. Architecture Extension Rules

Template'e yeni bir system eklerken:

1. Mevcut bir system'in sorumluluğunu tekrar edip etmediğini kontrol et.
2. Yeni system'in authoritative state'i olup olmadığını belirle.
3. Configuration ihtiyacını belirle.
4. Runtime state'i belirle.
5. Save gereksinimini belirle.
6. Event / direct reference ihtiyacını belirle.
7. UI ve gameplay dependency'lerini belirle.
8. İlgili documentation'ı oluştur veya güncelle.

Yeni system yalnızca "bir şeyleri organize etmek" için oluşturulmamalıdır.

Gerçek bir responsibility boundary sağlamalıdır.

---

# 28. Generic Template Rule

Bu template'in amacı belirli bir oyunun mimarisini kopyalamak değil, oyun geliştirme altyapısını tekrar kullanılabilir hale getirmektir.

Bu nedenle core architecture:

```text id="w6t2n9"
Game
├── Economy
├── Progression
├── Save
├── Monetization
├── Analytics
├── Audio
├── Game Flow
└── Gameplay Extension Points
```

gibi generic kavramlar üzerine kurulmalıdır.

Oyun-specific content ve mechanics `GAMEPLAY/` içerisinde genişletilmelidir.

Örneğin farklı projeler:

```text id="g1r8c4"
Puzzle Game
→ BoardGameplayModule

Runner Game
→ RunnerGameplayModule

Idle Game
→ IdleGameplayModule

Merge Game
→ MergeGameplayModule
```

kullanabilir.

Core architecture'ın bu farklılıklardan minimum seviyede etkilenmesi hedeflenir.

---

# 29. Architecture Change Rule

Architecture değişikliği, normal feature değişikliğinden farklı değerlendirilmelidir.

Aşağıdaki değişiklikler architecture change olarak kabul edilir:

* System responsibility değiştirmek.
* Yeni core system eklemek.
* System dependency değiştirmek.
* Runtime state ownership değiştirmek.
* Event contract değiştirmek.
* Save data structure değiştirmek.
* Bootstrap sırasını değiştirmek.
* Global game flow değiştirmek.
* Core folder structure değiştirmek.

Böyle bir değişiklik yapılmadan önce:

```text id="k3v9p2"
AGENTS.md
↓
ARCHITECTURE.md
↓
İlgili System / Gameplay / UI docs
↓
Mevcut implementation
```

incelenmelidir.

Değişiklik tamamlandığında ilgili documentation güncellenmelidir.

---

# 30. Source of Truth

Mimari sorularda source of truth:

```text id="z5m8r1"
AGENTS.md
→ Coding / agent behavior rules

ARCHITECTURE.md
→ System topology and architectural boundaries

BOOTSTRAP.md
→ Initialization flow

SYSTEMS/*.md
→ System-specific contracts

GAMEPLAY/*.md
→ Gameplay-specific contracts

UI/*.md
→ UI-specific contracts

Code
→ Actual implementation
```

Dokümantasyon ile implementation farklıysa bu bir hata veya migration ihtiyacı olabilir.

Agent hiçbirini otomatik olarak görmezden gelmemelidir.

Farkı anlamalı ve gerekli durumda documentation'ı güncellemelidir.

---

# Final Principle

Bu architecture'ın temel amacı mümkün olduğunca çok system oluşturmak değildir.

Amaç:

```text id="e2x7k4"
Clear Ownership
+
Low Coupling
+
Data-Driven Configuration
+
Testable Gameplay
+
Predictable Lifecycle
+
Mobile Performance
+
Simple Extension
```

sağlayan bir temel oluşturmaktır.

Yeni bir feature eklerken önce:

**"Hangi system'e ait?"**

sorusunu sor.

Sonra:

**"Bu state'in sahibi kim?"**

Sonra:

**"Mevcut hangi system'leri kullanabilir?"**

Ve en son:

**"Yeni bir abstraction gerçekten gerekli mi?"**

sorusunu değerlendir.
