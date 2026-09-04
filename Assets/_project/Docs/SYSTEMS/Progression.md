# Progression System

## 1. Amaç

`ProgressionSystem`, oyuncunun oyun boyunca biriken ve kalıcı olarak saklanması gereken ilerleme durumunu yönetir.

Temel sorumlulukları:

* Player progression state'ini yönetmek
* Unlock state'lerini yönetmek
* Milestone ve progression koşullarını değerlendirmek
* Progression değişikliklerini ilgili sistemlere bildirmek
* Progression state'inin persistence için SaveSystem ile entegre olmasını sağlamak
* Configuration ile runtime progression state'ini birbirinden ayırmak

Temel akış:

```text id="n4v7qa"
Progression Configuration
        ↓
 ProgressionSystem
        ↓
 Progression Runtime State
        ↓
 Progression Change
        ↓
 SaveSystem / UI / Analytics / Other Consumers
```

---

# 2. Temel Prensip

Progression state'inin tek authoritative owner'ı `ProgressionSystem` olmalıdır.

```text id="m8r3vc"
ProgressionSystem
        ↓
Player Progression State
```

Başka sistemler progression'ın kendi kopyasını source of truth olarak tutmamalıdır.

Örneğin:

```text id="q5x7za"
LevelSystem
    ✗ Persistent Progression Owner

TopBarHeader
    ✗ Persistent Progression Owner

ShopView
    ✗ Persistent Progression Owner

GameplayModule
    ✗ Persistent Progression Owner

ProgressionSystem
    ✓ Persistent Progression Owner
```

---

# 3. Progression Nedir?

Progression, oyuncunun oyun içerisindeki kalıcı ilerlemesini ifade eder.

Örnekler:

```text id="v6m2qa"
Completed Content
Unlocked Content
Milestones
Player Rank
Experience
Progression Level
Upgrade Level
Feature Unlocks
```

Bunların tamamının aynı projede bulunması gerekmez.

Template yalnızca generic progression modelini tanımlar.

---

# 4. Progression Configuration

Designer tarafından değiştirilebilecek progression kuralları configuration olarak tutulabilir.

Örneğin:

```text id="p8r4mc"
ProgressionConfig
    ↓
Milestone Definitions
Unlock Definitions
Thresholds
Requirements
```

ScriptableObject configuration static data'dır.

Runtime progression state burada tutulmamalıdır.

---

# 5. Runtime State

Runtime progression state örneği:

```text id="x7q3va"
Progression State
├── Current Progression Level
├── Experience
├── Completed Milestones
└── Unlocked Content IDs
```

Bu state `ProgressionSystem` tarafından yönetilir.

---

# 6. Save Data

Persistence için ayrı bir model kullanılmalıdır.

```text id="m4r8xc"
ProgressionRuntimeState
        ↓
ProgressionSaveData
        ↓
PlayerSaveData
```

Save model yalnızca persistence gerektiren state'i taşımalıdır.

Temporary runtime state save edilmemelidir.

---

# 7. Configuration, Runtime ve Save Ayrımı

Örneğin:

```text id="q6v2za"
ProgressionConfig
    → Level 10 unlocks Feature A

ProgressionRuntimeState
    → Player is currently at Level 8

ProgressionSaveData
    → Level 8
```

Bunlar aynı data değildir.

---

# 8. Progression ve LevelSystem Ayrımı

İki sistem farklı sorulara cevap verir.

`LevelSystem`:

> Hangi level oynanıyor ve bu level'ın lifecycle'ı ne?

`ProgressionSystem`:

> Oyuncunun kalıcı ilerlemesi nedir?

Örneğin:

```text id="v5m8qa"
GameFlow
    ↓
LevelSystem.StartLevel()
    ↓
Gameplay
    ↓
LevelSystem.CompleteLevel()
    ↓
Level Result
    ↓
ProgressionSystem.ApplyProgress()
```

---

# 9. Level Completion

Bir level'ın tamamlanması progression değişikliğine neden olabilir.

Örneğin:

```text id="p3q7vc"
Level Completed
    ↓
ProgressionSystem
    ↓
Completed Level State
```

Ancak `LevelSystem` progression state'ini kendisi mutate etmemelidir.

---

# 10. Progression ve Gameplay

Gameplay bir progression sonucu üretebilir.

Örneğin:

```text id="x8m4za"
Gameplay Result
    ↓
Objective / Progression Update
    ↓
ProgressionSystem
```

GameplaySystem:

```text id="m7r2vc"
progressionLevel++;
```

gibi doğrudan mutation yapmamalıdır.

---

# 11. Progression Increment

Basit progression için:

```csharp id="q4v8ma"
bool TryAddProgress(string progressionId, int amount);
```

gibi bir API kullanılabilir.

Ancak progression'ın türüne göre daha domain-specific methodlar gerekebilir.

API gereksiz şekilde büyütülmemelidir.

---

# 12. Progression Query

Diğer sistemler progression state'i okuyabilir.

Örneğin:

```csharp id="v6r3xc"
var level = progressionSystem.GetProgressionLevel();
```

veya:

```csharp id="p8m4za"
var unlocked = progressionSystem.IsUnlocked(contentId);
```

Query methodları state değiştirmemelidir.

---

# 13. Unlock State

Unlock state progression'ın yaygın bir parçasıdır.

Örneğin:

```text id="x5q7vc"
Content ID
    ↓
Unlocked?
```

Runtime state:

```text id="m3r8qa"
UnlockedContent
├── content_01
├── content_02
└── content_07
```

olabilir.

---

# 14. Unlock Ownership

Unlock state'in owner'ı `ProgressionSystem` olmalıdır.

UI:

```text id="q8v4mc"
✗ unlocked = true
```

yapmamalıdır.

Gameplay:

```text id="k6r2za"
✗ content.unlocked = true
```

yapmamalıdır.

Doğru:

```text id="v7m3xa"
ProgressionSystem.Unlock(contentId)
```

---

# 15. Unlock Conditions

Unlock condition configuration'da tanımlanabilir.

Örneğin:

```text id="p4q8vc"
Unlock Definition
├── Content ID
├── Required Progression
└── Requirements
```

ProgressionSystem condition'ı değerlendirip sonucu runtime state'e uygulayabilir.

---

# 16. Requirement

Bir unlock birden fazla requirement gerektirebilir.

Örneğin:

```text id="x6m2qa"
Unlock Content
    ├── Progression >= 10
    ├── Required Content Unlocked
    └── Optional Requirement
```

Requirement değerlendirmesi ilgili domain owner'larından bilgi okuyabilir.

---

# 17. Cross-System Requirements

Örneğin unlock için currency gerekiyorsa:

```text id="m8r4vc"
ProgressionSystem
    ↓
EconomySystem.HasEnough()
```

kullanılabilir.

Ancak progression system:

```text id="q5v7za"
currency -= cost
```

yapmamalıdır.

Currency mutation yine EconomySystem'e aittir.

---

# 18. Atomic Unlock

Unlock birden fazla state değişikliği gerektiriyorsa operation'ın owner'ı açık olmalıdır.

Örneğin:

```text id="v3m8xa"
Unlock Content
    ↓
Validate Requirements
    ↓
EconomySystem.TrySpend()
    ↓
ProgressionSystem.Unlock()
```

Buradaki orchestration ayrı bir feature system'e ait olabilir.

Her cross-system operation ProgressionSystem'e doldurulmamalıdır.

---

# 19. Milestones

Milestone, belirli bir progression koşulunun tamamlandığını ifade eder.

Örneğin:

```text id="p7q3vc"
Milestone
    ↓
Requirement Met
    ↓
Completed
```

Milestone state kalıcıysa SaveSystem'e yazılmalıdır.

---

# 20. Milestone Configuration

Örneğin:

```text id="x4m8qa"
MilestoneDefinition
├── ID
├── Requirement
├── Reward
└── Metadata
```

Reward bilgisi gerekiyorsa RewardSystem ile entegre olabilir.

ProgressionSystem reward'ın currency mutation'ını kendisi yapmamalıdır.

---

# 21. Reward İlişkisi

Progression sonucu reward oluşabilir.

```text id="m6r2vc"
ProgressionSystem
    ↓
Milestone Completed
    ↓
RewardSystem
    ↓
EconomySystem
```

Bu ayrım önemlidir.

Progression:

> Oyuncu milestone'ı tamamladı mı?

RewardSystem:

> Bu milestone hangi ödülü üretir?

EconomySystem:

> Currency state nasıl değişir?

---

# 22. Progression Event

Progression değişiklikleri event ile duyurulabilir.

Örneğin:

```text id="q8v4ma"
ProgressionChanged
UnlockChanged
MilestoneCompleted
```

Event yalnızca gerekli consumer'lar varsa kullanılmalıdır.

---

# 23. Event Payload

Örneğin:

```text id="v5m7xc"
ProgressionChanged
├── ProgressionId
├── PreviousValue
└── NewValue
```

ve:

```text id="p3r8za"
UnlockChanged
├── ContentId
└── IsUnlocked
```

gibi payload'lar kullanılabilir.

Event payload gereksiz data taşımamalıdır.

---

# 24. Event ve Source of Truth

Event:

```text id="x7q4mc"
ProgressionChanged
```

progression state değildir.

UI event'i kaçırırsa:

```text id="m5r2va"
ProgressionSystem.Get...
```

ile mevcut state'i okuyabilmelidir.

Bu nedenle:

```text id="q8m3xc"
Initial Sync
+
Event Updates
```

kullanılmalıdır.

---

# 25. Event Lifecycle

Consumer'lar:

```text id="v4r7qa"
OnEnable
    ↓
Subscribe

OnDisable
    ↓
Unsubscribe
```

uygulamalıdır.

Pooled UI veya runtime object'ler için bu özellikle önemlidir.

---

# 26. Progression Save

Progression değiştiğinde SaveSystem dirty olabilir.

```text id="p6m8vc"
Progression Changed
    ↓
Save Dirty
    ↓
SaveSystem
    ↓
PlayerSaveData
```

ProgressionSystem JSON'a doğrudan yazmamalıdır.

---

# 27. Progression Load

Bootstrap sırasında:

```text id="x3q7ma"
SaveSystem
    ↓
ProgressionSaveData
    ↓
ProgressionSystem.ApplyLoadedState()
```

şeklinde state uygulanabilir.

Loaded state validation'dan geçirilmelidir.

---

# 28. Progression Validation

Load edilen progression state doğrulanmalıdır.

Örneğin:

```text id="m8r4vc"
Progression Level >= Minimum
Experience >= 0
Content ID valid
Milestone ID valid
Unlock state valid
```

Invalid state kontrollü şekilde düzeltilmelidir.

---

# 29. Progression Consistency

Aşağıdaki invariant korunmalıdır:

```text id="q5v8xa"
ProgressionSystem Runtime State
        =
Current Authoritative Progression State
```

Save, UI ve analytics bunun farklı representation'larıdır.

---

# 30. Progression Level

Bazı oyunlarda progression bir level/rank değeri içerebilir.

Örneğin:

```text id="v7m3mc"
Progression Level = 12
Experience = 450
```

Bu değer gameplay level'dan farklıdır.

Template'te:

```text id="p4q8za"
Player Progression Level
```

ve:

```text id="x6r2vc"
Gameplay Level
```

birbirine karıştırılmamalıdır.

---

# 31. Experience

Experience gerekiyorsa ProgressionSystem owner olabilir.

Örneğin:

```text id="m7q4xa"
Add Experience
    ↓
Threshold Check
    ↓
Progression Level Up
```

Experience değişikliği progression state'inin parçasıdır.

---

# 32. Level Up

Progression level-up gerçekleştiğinde:

```text id="v8m3qc"
Experience Changed
    ↓
Threshold Reached
    ↓
Progression Level Increased
    ↓
Unlock Evaluation
    ↓
Events
    ↓
Save
```

olabilir.

UI level-up animasyonunu gösterir.

Level-up state'in sahibi UI değildir.

---

# 33. Unlock Evaluation

Progression değiştikten sonra unlock condition'ları yeniden değerlendirmek gerekebilir.

Örneğin:

```text id="q4r7mc"
Progression Level 9
    ↓
No Unlock

Progression Level 10
    ↓
Evaluate
    ↓
Content A Unlocked
```

Evaluation yalnızca gerçekten gerekli content'ler için yapılmalıdır.

Her frame bütün unlock tree taranmamalıdır.

---

# 34. Performance

Progression checks gameplay hot path'e gereksiz yük bindirmemelidir.

Kaçınılması gerekenler:

```text id="x8m2va"
Every Frame
    ↓
Evaluate All Unlocks
```

Bunun yerine:

```text id="m5r7vc"
Relevant State Changed
    ↓
Evaluate Relevant Conditions
```

yaklaşımı tercih edilmelidir.

---

# 35. Dependency Direction

Genel dependency:

```text id="v6q3za"
Gameplay
   ↓
Progression
   ↓
Save
```

veya:

```text id="p8r4mc"
Progression
   ↓
Economy Query
```

gibi olabilir.

Ancak circular dependency oluşturulmamalıdır.

Örneğin:

```text id="q7m3xa"
Progression → Economy
Economy → Progression
```

zorunlu hale geliyorsa orchestration layer değerlendirilmelidir.

---

# 36. Progression ve GameFlow

ProgressionSystem global game state'in owner'ı değildir.

Yanlış:

```text id="x4r8vc"
ProgressionSystem
    ↓
Set GameFlow State
```

GameFlow:

```text id="m7q2za"
Boot
MainMenu
Gameplay
Pause
Win
Lose
```

global flow'u yönetir.

Progression yalnızca progression state'i yönetir.

---

# 37. Progression ve UI

UI progression bilgisini gösterebilir.

Örneğin:

```text id="v5m8qa"
ProgressionSystem
    ↓
Progression Event
    ↓
Profile UI
```

UI:

* Progression hesaplamaz
* Unlock state değiştirmez
* Save yazmaz
* Reward uygulamaz

---

# 38. Profile ile İlişki

ProfileSystem oyuncunun profil state'ini yönetir.

ProgressionSystem:

```text id="p4q7vc"
Progression
```

yönetir.

Profile UI ikisini gösterebilir:

```text id="x8m3za"
Profile
├── Player Identity
├── Avatar
└── Progression Summary
```

Ancak tek bir system içine birleştirilmek zorunda değildir.

---

# 39. TopBar ile İlişki

TopBar progression özeti gösterebilir.

```text id="m6r2qa"
ProgressionSystem
    ↓
Progression State
    ↓
TopBarHeader
```

TopBar kendi progression state'ini tutmamalıdır.

---

# 40. Tutorial ile İlişki

Tutorial tamamlanması progression'a kaydedilebilir.

```text id="v7q4mc"
TutorialSystem
    ↓
Tutorial Completed
    ↓
ProgressionSystem
    ↓
Tutorial Progress Saved
```

TutorialSystem kendi progression persistence mekanizmasını oluşturmamalıdır.

---

# 41. Content Unlock

Yeni content unlock olduğunda:

```text id="p8m3xa"
ProgressionSystem
    ↓
Content Unlocked
    ↓
Relevant Consumer
```

Örneğin UI:

```text id="x5q7vc"
Locked
    ↓
Unlocked
```

olarak presentation'ı günceller.

Content'in kendisinin progression state'i source of truth değildir.

---

# 42. Locked Content

Bir content'in locked olması configuration'dan gelen bir default olabilir.

Runtime unlock state bunu override eder.

Örneğin:

```text id="m4r8za"
Configuration
    ↓
Content exists
    ↓
Progression
    ↓
Unlocked?
```

Content asset'in içine player-specific unlocked boolean yazılmamalıdır.

---

# 43. New Content

Yeni content eklenmesi mevcut save'i bozmak zorunda değildir.

Örneğin:

```text id="q7m3vc"
Content 101 Added
```

eski save'de bulunmuyorsa:

```text id="v8r4xa"
Default = Locked
```

veya configuration'ın belirlediği başlangıç state'i kullanılabilir.

---

# 44. Removed Content

Content kaldırıldığında eski save'de onun ID'si bulunabilir.

ProgressionSystem:

```text id="p5q8mc"
Unknown Content ID
```

durumunu kontrollü ele almalıdır.

Seçenekler:

```text id="x4m7za"
Ignore
Migrate
Retire
Replace
```

game design'e göre belirlenebilir.

---

# 45. Migration

Progression save schema'sı değiştiğinde migration gerekir.

Örneğin:

```text id="m8r3vc"
V1
    ↓
V2
    ↓
V3
```

Migration SaveSystem/SaveMigration sınırında yapılmalıdır.

ProgressionSystem eski save version'larını bilmemelidir.

---

# 46. Migration Example

Örneğin V1:

```text id="q6v2ma"
completedLevels
```

V2:

```text id="p8r4xc"
completedContent
```

olduysa:

```text id="v5m7za"
Migration V1 → V2
```

mapping işlemini gerçekleştirir.

Runtime system yalnızca current model ile çalışır.

---

# 47. Reset

Development sırasında progression resetlenebilir.

Production player reset:

```text id="x7m3vc"
SaveSystem
    ↓
Default Save
    ↓
ProgressionSystem
```

üzerinden yapılmalıdır.

ProgressionSystem doğrudan save file silmemelidir.

---

# 48. Debug Tools

Development build için:

```text id="m4q8xa"
Set Progression
Unlock Content
Complete Milestone
Reset Progression
Add Experience
Validate Progression
```

gibi debug araçları faydalı olabilir.

Bunlar core gameplay API'sine zorunlu olarak eklenmek zorunda değildir.

---

# 49. Analytics

Progression event'leri AnalyticsSystem tarafından tüketilebilir.

Örneğin:

```text id="v8r3mc"
Progression Level Up
Content Unlocked
Milestone Completed
```

Analytics SDK çağrıları ProgressionSystem'e doğrudan gömülmemelidir.

---

# 50. Audio / Presentation

Unlock veya level-up sesleri:

```text id="p6m2qa"
Progression Event
    ↓
AudioSystem
```

üzerinden tetiklenebilir.

ProgressionSystem ses efektinin nasıl çalındığını bilmemelidir.

---

# 51. Reward Timing

Progression event'i ile reward application sırası açık olmalıdır.

Örneğin:

```text id="x4q7vc"
Milestone Completed
    ↓
Reward Granted
```

veya:

```text id="m8r3za"
Reward Granted
    ↓
Milestone Completed Event
```

gibi bir contract seçilmelidir.

Consumer'lar event ordering'e gizlice bağımlı hale getirilmemelidir.

---

# 52. Idempotency

Aynı progression command'in tekrar gelmesi gerekiyorsa sonuç kontrollü olmalıdır.

Örneğin:

```text id="v7m4xc"
Unlock(content_01)
Unlock(content_01)
```

ikinci çağrı:

```text id="q5r8za"
No-op
```

olabilir.

Aynı milestone'ın iki kez reward vermesi engellenmelidir.

---

# 53. Duplicate Rewards

Progression milestone'ları özellikle duplicate reward açısından dikkatle tasarlanmalıdır.

Yanlış:

```text id="x6m3va"
Milestone Completed
    ↓
Reward
Milestone Completed
    ↓
Reward
```

Doğru:

```text id="p8q4mc"
Check Completed?
    ↓
No
    ↓
Mark Completed
    ↓
Grant Reward
```

Transaction ordering açık olmalıdır.

---

# 54. Progression Commands

Command ve event ayrımı korunmalıdır.

Command:

```text id="m7r2vc"
UnlockContent
AddProgress
CompleteMilestone
```

> Bir işlem yapılmasını ister.

Event:

```text id="v4q8za"
ContentUnlocked
ProgressionChanged
MilestoneCompleted
```

> Bir işlemin gerçekleştiğini bildirir.

EventBus command dispatcher olarak kullanılmak zorunda değildir.

---

# 55. Direct Reference vs Event

Basit local ownership varsa direct reference kullanılabilir.

Örneğin:

```text id="x8m3qa"
ProgressionFeature
    ↓
ProgressionSystem
```

Birden fazla bağımsız consumer varsa event kullanılabilir.

Örneğin:

```text id="q6r4va"
ProgressionChanged
    ├── UI
    ├── Analytics
    └── Audio
```

Her interaction'ı EventBus üzerinden geçirmek gerekmez.

---

# 56. Persistence Frequency

Progression değişiklikleri önemli state change olduğundan save dirty oluşturabilir.

Ancak:

```text id="m5q8vc"
Every Progress Tick
    ↓
Disk Write
```

yapılmamalıdır.

Birden fazla değişiklik batch/debounced save ile birleştirilebilir.

---

# 57. Hot Path

Progression update sırasında:

* LINQ
* Temporary allocations
* Reflection
* Repeated Find
* Unnecessary string creation
* Full unlock tree scans

gibi işlemlerden kaçınılmalıdır.

Özellikle gameplay'in sık tetiklediği progression checks optimize edilmelidir.

---

# 58. Testing

ProgressionSystem için test edilmesi gerekenler:

```text id="v7m3qa"
Add Progress
Level Up
Unlock
Milestone Completion
Duplicate Completion
Invalid Requirement
Invalid Content ID
Load State
Reset
Save Conversion
```

---

# 59. EditMode Tests

Scene gerektirmeyen progression logic `EditMode` testleriyle test edilmelidir.

Özellikle:

* Requirement evaluation
* Progress calculation
* Level thresholds
* Unlock rules
* Milestone completion
* Idempotency

test edilebilir.

---

# 60. PlayMode Tests

Unity lifecycle veya system integration gerektiren durumlar:

* Bootstrap
* Save load
* Runtime state application
* Event subscription
* UI synchronization
* Cross-system integration

için `PlayMode` testleri kullanılabilir.

---

# 61. Deterministic Progression

Aynı input ve aynı başlangıç state'i mümkün olduğunca aynı progression sonucunu üretmelidir.

Örneğin:

```text id="q4m8vc"
Initial Progression
+
Same Completed Content
+
Same Rewards
=
Same Progression Result
```

Random progression logic gerekiyorsa randomness açıkça yönetilmelidir.

---

# 62. Progression and Time

Time-based progression gerekiyorsa timestamp veya time service kullanılabilir.

Örneğin:

```text id="x7r3ma"
Last Reset Time
Next Available Time
```

gibi persistence state'i tutulabilir.

Raw `DateTime.Now` kullanımının sistemlere dağılması yerine project'in Time/Clock yaklaşımı kullanılmalıdır.

---

# 63. Progression and Remote Config

Remote configuration progression threshold'larını değiştirebilir.

Ancak runtime progression state ile config birbirine karıştırılmamalıdır.

```text id="m8q4vc"
Remote / Static Config
    ↓
Progression Rules
```

ve:

```text id="v5r7za"
Player Progression
    ↓
Runtime State
```

ayrı kalır.

Config değişikliği mevcut oyuncu state'inin semantic anlamını etkiliyorsa migration/compatibility ayrıca değerlendirilmelidir.

---

# 64. Progression and Offline State

Progression local olarak çalışabiliyorsa:

```text id="p4m8xa"
ProgressionSystem
    ↓
SaveSystem
    ↓
Local JSON
```

yeterli olabilir.

Online authoritative progression gerekiyorsa server synchronization ayrıca tasarlanmalıdır.

Local SaveSystem tek başına server authority değildir.

---

# 65. Progression and Cloud Save

İleride:

```text id="x6r3vc"
ProgressionSystem
    ↓
SaveSystem
    ↓
Local / Cloud Persistence
```

kullanılabilir.

ProgressionSystem'in local JSON veya cloud SDK detaylarını bilmesi gerekmemelidir.

---

# 66. No God Progression Manager

ProgressionSystem içine:

```text id="m7q2za"
Shop
Economy
IAP
Ads
Gameplay
UI
Audio
Analytics
```

mantığı doldurulmamalıdır.

ProgressionSystem'in domain'i:

```text id="v8r4mc"
Player Progress
Unlocks
Milestones
Progression Rules
```

ile sınırlı tutulmalıdır.

---

# 67. Generic Template Rule

Core template'e game-specific progression isimleri gömülmemelidir.

Kaçınılması gerekenler:

```text id="q5m8za"
KingdomStars
CastleLevel
DecorationStars
SpecificBuildingUnlock
```

Bunun yerine:

```text id="x4r7vc"
Progression
Milestone
Unlock
Requirement
Content
```

gibi generic kavramlar kullanılmalıdır.

---

# 68. Source of Truth

Progression için:

```text id="m8q3xa"
Configuration
    → ProgressionConfig

Runtime State
    → ProgressionSystem

Persistence
    → SaveSystem

Presentation
    → UI

Analytics
    → AnalyticsSystem
```

şeklinde net ownership korunmalıdır.

---

# 69. Responsibility Matrix

| Sorumluluk                | Owner                        |
| ------------------------- | ---------------------------- |
| Progression runtime state | ProgressionSystem            |
| Progression rules         | ProgressionConfig            |
| Unlock state              | ProgressionSystem            |
| Milestone state           | ProgressionSystem            |
| Gameplay session state    | Gameplay / LevelSystem       |
| Currency state            | EconomySystem                |
| Reward application        | RewardSystem / ilgili system |
| Persistence               | SaveSystem                   |
| Presentation              | UI                           |
| Analytics                 | AnalyticsSystem              |
| Global game state         | GameFlow                     |

---

# 70. Definition of Done

ProgressionSystem implementation tamamlanmadan önce:

* [ ] Progression state'in tek owner'ı ProgressionSystem mi?
* [ ] Configuration runtime state'ten ayrılmış mı?
* [ ] Save data runtime state'ten ayrılmış mı?
* [ ] LevelSystem ile progression ayrımı net mi?
* [ ] Gameplay progression state'i doğrudan mutate etmiyor mu?
* [ ] Unlock state merkezi olarak yönetiliyor mu?
* [ ] Unlock condition'ları configuration'dan geliyor mu?
* [ ] Milestone state persistence gerektiriyorsa save ediliyor mu?
* [ ] Reward application ProgressionSystem'e gereksiz şekilde gömülmemiş mi?
* [ ] Currency mutation EconomySystem'de mi?
* [ ] Progression event'leri gerekiyorsa kullanılıyor mu?
* [ ] Event source of truth olarak kullanılmıyor mu?
* [ ] UI initial state sync yapıyor mu?
* [ ] UI progression state owner'ı değil mi?
* [ ] SaveSystem'e doğrudan JSON yazılmıyor mu?
* [ ] Load sırasında progression state validate ediliyor mu?
* [ ] Migration SaveSystem sınırında mı?
* [ ] Duplicate unlock engelleniyor mu?
* [ ] Duplicate milestone reward engelleniyor mu?
* [ ] Idempotency gereken operations güvenli mi?
* [ ] Unlock evaluation gereksiz şekilde her frame çalışmıyor mu?
* [ ] Hot path allocation'ları kontrol edilmiş mi?
* [ ] Cross-system circular dependency yok mu?
* [ ] GameFlow progression tarafından yönetilmiyor mu?
* [ ] Analytics SDK doğrudan ProgressionSystem'de değil mi?
* [ ] Audio/presentation logic progression'a gömülmemiş mi?
* [ ] Content ID'leri stable mı?
* [ ] Unknown content ID kontrollü ele alınıyor mu?
* [ ] Removed content için migration/retirement stratejisi düşünülmüş mü?
* [ ] New content mevcut save'leri bozmuyor mu?
* [ ] Progression reset davranışı tanımlı mı?
* [ ] Local/cloud persistence ayrımı korunmuş mu?
* [ ] Competitive/server-authoritative ihtiyaçlar local save'e bırakılmamış mı?
* [ ] Generic template'e game-specific progression kavramları gömülmemiş mi?
* [ ] EditMode testleri kritik progression logic'i kapsıyor mu?
* [ ] PlayMode integration testleri gerektiğinde mevcut mu?
* [ ] ProgressionSystem God Object'a dönüşmemiş mi?
* [ ] Gereksiz abstraction eklenmemiş mi?
