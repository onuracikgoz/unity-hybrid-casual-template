# Level System

## 1. Amaç

`LevelSystem`, oyunun level tabanlı gameplay lifecycle'ını yönetir.

Bu dokümanın amacı:

* Level configuration ile runtime level state'i ayırmak
* Level lifecycle sorumluluklarını tanımlamak
* `GameFlow` ile `LevelSystem` arasındaki sınırı belirlemek
* Level yükleme, başlatma, oynanma ve tamamlanma akışını tanımlamak
* Win/Lose sonucunun nasıl üretileceğini belirlemek
* Level progression ile gameplay state arasındaki ilişkiyi açıklamak
* Level asset'lerinin configuration ve Asset Management ile ilişkisini tanımlamak
* Level sisteminin farklı gameplay türlerine uyarlanabilir olmasını sağlamak

`LevelSystem`, gameplay'e ait level lifecycle'ın sahibidir.

Global game state'in sahibi değildir.

---

# 2. Temel Mimari

Level lifecycle:

```text id="k7r2mx"
Level Configuration
        ↓
    LevelSystem
        ↓
Current Level State
        ↓
Gameplay Session
        ↓
Win / Lose
        ↓
Level Result
```

Global flow ayrı bir sorumluluktur:

```text id="m4q8za"
GameFlow
 ├── Boot
 ├── MainMenu
 ├── Gameplay
 ├── Pause
 ├── Win
 └── Lose
```

İlişki:

```text id="p9v3cx"
GameFlow
    ↓
Enter Gameplay
    ↓
LevelSystem.StartLevel()
    ↓
Gameplay
    ↓
LevelSystem.CompleteLevel()
    ↓
GameFlow → Win
```

GameFlow, level'in kurallarını yönetmez.

LevelSystem, global GameFlow state'lerini yönetmez.

---

# 3. LevelSystem Sorumlulukları

`LevelSystem` aşağıdaki sorumluluklara sahip olabilir:

* Current level kimliğini yönetmek
* Level configuration'ı resolve etmek
* Level başlatmak
* Level gameplay session'ını oluşturmak
* Level completion durumunu takip etmek
* Win koşullarını değerlendirmek
* Lose koşullarını değerlendirmek
* Level result üretmek
* Level restart işlemini yönetmek
* Level exit/cleanup işlemini yönetmek
* Level progression'a gerekli sonucu bildirmek
* İlgili level asset/content yükleme lifecycle'ını koordine etmek

`LevelSystem` aşağıdakilerin sahibi değildir:

* Global GameFlow state
* UI presentation
* Currency state
* Save data
* Monetization
* Analytics SDK
* Gameplay object's internal behavior
* Generic input handling

---

# 4. Level Configuration

Designer tarafından değiştirilebilir level verileri configuration olarak tutulmalıdır.

Örneğin:

```text id="c8n5wb"
LevelConfig
├── LevelId
├── Gameplay Configuration
├── Difficulty
├── Initial Content
├── Goals
├── Time Limit
├── Optional Modifiers
└── Asset References
```

Bu yapı oyunun türüne göre değişebilir.

Örneğin bir oyunda:

```text id="v2k7pd"
Moves
Goals
Board Layout
```

gerekebilir.

Başka bir oyunda:

```text id="f6r1xm"
Enemy Count
Spawn Configuration
Objective
Time Limit
```

gerekebilir.

Template bu alanları zorunlu hale getirmemelidir.

---

# 5. Level ID

Level kimliği ile level sırası birbirinden ayrılmalıdır.

Örneğin:

```text id="n4z8qa"
LevelId = 1001
```

bir content kimliği olabilir.

Level index ise:

```text id="q7m2vx"
CurrentLevelIndex = 25
```

oyuncunun progression konumunu temsil edebilir.

Her projede ikisinin ayrı olması şart değildir.

Ancak runtime state ile content identity'nin birbirine karıştırılmaması önemlidir.

---

# 6. Runtime Level State

Level configuration static data'dır.

Runtime state ise oyuncunun mevcut gameplay session'ına aittir.

Örneğin:

```text id="x5c9rm"
LevelConfig
    ↓
Static Rules / Content

LevelRuntimeState
    ↓
Current Objective Progress
Moves Remaining
Time Remaining
Current Result
Session State
```

Runtime state ScriptableObject üzerinde kalıcı mutable state olarak tutulmamalıdır.

---

# 7. Source of Truth

Current level state için tek bir authoritative owner olmalıdır.

Örneğin:

```text id="b8q4nk"
LevelSystem
    ↓
CurrentLevel
CurrentLevelState
```

Diğer sistemler current level bilgisini LevelSystem'den alabilir.

UI kendi içinde:

```csharp id="w3p7zf"
_currentLevel = 25;
```

gibi ayrı bir authoritative state tutmamalıdır.

UI'daki:

```text id="r6m2va"
"Level 25"
```

yalnızca presentation'dır.

---

# 8. Level Başlatma

Level başlatma akışı:

```text id="t9x4pc"
Request Level
      ↓
Resolve LevelConfig
      ↓
Load Required Content
      ↓
Create Level Runtime State
      ↓
Initialize Gameplay
      ↓
Start Gameplay Session
```

Level gerçekten playable hale gelmeden önce gerekli dependency'ler hazır olmalıdır.

---

# 9. GameFlow ile İlişki

GameFlow level başlatma isteğini oluşturabilir:

```text id="h6m3yv"
GameFlow
    ↓
Gameplay State
    ↓
LevelSystem.StartLevel(levelId)
```

Ancak LevelSystem:

```text id="q1r8kd"
StartLevel()
    ↓
Load Content
    ↓
Initialize Gameplay
    ↓
Begin Session
```

sorumluluğundadır.

LevelSystem'in:

```text id="c5w9nb"
GameFlow.ChangeState(GameState.Gameplay)
```

gibi global flow yönetmesi tercih edilmez.

Ownership tersine çevrilmemelidir.

---

# 10. Level Restart

Restart bir level lifecycle operation'dır.

Önerilen akış:

```text id="m7v2xa"
Restart Request
      ↓
End Current Session
      ↓
Cleanup Runtime State
      ↓
Reset Gameplay
      ↓
Create New Runtime State
      ↓
Start Level
```

Restart sırasında eski runtime state'in yeni session'a sızmaması gerekir.

Özellikle:

* Event subscriptions
* Timers
* Coroutines
* Tween'ler
* Pooled objects
* Temporary modifiers
* Input state
* Board state
* Gameplay flags

temizlenmelidir.

---

# 11. Level Cleanup

Level tamamlandığında veya terk edildiğinde level-specific runtime state temizlenmelidir.

Örneğin:

```text id="f8q3zs"
End Level
    ↓
Stop Gameplay
    ↓
Unsubscribe
    ↓
Release Level Resources
    ↓
Reset Pools / Runtime Objects
    ↓
Clear Session State
```

Persistent system state temizlenmemelidir.

Örneğin:

```text id="j2v6km"
EconomySystem
ProgressionSystem
SaveSystem
```

global lifecycle'a göre yaşamaya devam edebilir.

---

# 12. Gameplay Session

Her level attempt bir gameplay session olarak düşünülebilir.

Örneğin:

```text id="u9r4cx"
Level 25
   ↓
Attempt #1
   ↓
Lose

Level 25
   ↓
Attempt #2
   ↓
Win
```

Session-specific state:

```text id="k3m7pa"
Moves
Timer
Objective Progress
Temporary Boosts
Board State
Spawn State
```

gibi verileri içerebilir.

Bunlar level attempt'ine aittir.

---

# 13. Level Result

Level completion sonucunu `LevelSystem` üretmelidir.

Örneğin:

```text id="e5n8wd"
LevelResult
├── LevelId
├── Outcome
├── Score
├── Stars / Rating
├── Duration
├── Objectives
└── Optional Metrics
```

`Outcome` örneğin:

```text id="p4c9xm"
Win
Lose
Quit
Aborted
```

olabilir.

Template belirli bir result modelini zorunlu hale getirmemelidir.

---

# 14. Win

Win koşulları gameplay'e göre değişir.

Örneğin:

```text id="x8q2vz"
All Objectives Completed
```

veya:

```text id="d5m7ra"
Target Score Reached
```

veya:

```text id="n3k9cp"
Boss Defeated
```

olabilir.

Win condition'ın gerçek sahibi gameplay/level logic olmalıdır.

UI:

```text id="q6v1mb"
if (score >= targetScore)
```

gibi win logic hesaplamamalıdır.

---

# 15. Lose

Lose koşulları da gameplay'e aittir.

Örneğin:

```text id="h4p8zs"
Moves == 0
```

veya:

```text id="w7r2nd"
Timer <= 0
```

veya:

```text id="c9m5xa"
Objective Failed
```

olabilir.

Lose kararı gameplay/level logic tarafından üretilmelidir.

UI yalnızca sonucu gösterebilir.

---

# 16. Win / Lose Event

Level sonucu ilgili katmanlara bildirilebilir.

Örneğin:

```text id="r5y8kb"
LevelSystem
    ↓
LevelCompleted
    ↓
GameFlow
    ↓
Win
```

ve:

```text id="m2c7vf"
LevelSystem
    ↓
LevelFailed
    ↓
GameFlow
    ↓
Lose
```

Burada GameFlow yalnızca global state transition'ı yönetir.

Reward:

```text id="s6n3qa"
LevelCompleted
    ↓
Reward / Progression
```

şeklinde ayrı sistemler tarafından işlenebilir.

---

# 17. Level Completion ve GameFlow

LevelSystem'in level'i tamamlaması ile GameFlow'un Win state'ine geçmesi aynı sorumluluk değildir.

```text id="y4k8pc"
LevelSystem
    ↓
Level Result = Win
```

ardından:

```text id="v7m2za"
GameFlow
    ↓
State = Win
```

Bu ayrım test edilebilirliği ve sorumluluk sınırlarını korur.

---

# 18. Level Progression

LevelSystem current level'i yönetebilir.

Ancak oyuncunun kalıcı progression state'i `ProgressionSystem` tarafından sahiplenilmelidir.

Örneğin:

```text id="k5q9xb"
LevelSystem
    ↓
Level 25 Completed
    ↓
Level Result
    ↓
ProgressionSystem
    ↓
Unlock / Advance
```

LevelSystem:

```text id="t8r3mv"
Current Gameplay Level
```

ile ilgilenirken ProgressionSystem:

```text id="a6w2zd"
Player Progression
Unlocked Content
Completed Progress
```

ile ilgilenir.

---

# 19. Next Level

Next level kararı progression modeline göre değişebilir.

Basit bir oyunda:

```text id="p3f7ck"
Level 25
   ↓
Win
   ↓
Level 26
```

olabilir.

Ancak branching progression'da:

```text id="x9m4qa"
Level Result
    ↓
ProgressionSystem
    ↓
Next Available Content
```

daha doğru olabilir.

LevelSystem sırf index arttırarak progression kurallarını kendi içine gömmemelidir.

---

# 20. Level Load ve Asset Management

Level asset'leri:

```text id="n8c5wr"
LevelConfig
    ↓
Required Assets
    ↓
Asset Management
    ↓
Loaded Level Content
```

şeklinde yönetilebilir.

Asset Management:

* Direct reference
* Addressables
* Resources
* Load
* Release
* Lifetime

stratejisini belirler.

LevelSystem ise hangi content'in gerekli olduğunu bilir.

Bu iki sorumluluk birbirinden ayrılmalıdır.

---

# 21. Level Asset Örneği

Örneğin:

```text id="j6v2ps"
Art/Gameplay/Levels/Level_025.asset
```

veya bir prefab/content reference olabilir.

Akış:

```text id="r9k3xa"
LevelConfig
    ↓
Level Content Reference
    ↓
LevelSystem
    ↓
Asset Management
    ↓
Gameplay Scene / Content
```

Asset path'in gameplay koduna hard-code edilmesi tercih edilmez.

---

# 22. Scene Kullanımı

Level'ler scene tabanlı olabilir veya runtime content olarak oluşturulabilir.

İki yaklaşım da template için geçerlidir.

Scene tabanlı:

```text id="c7m2zf"
Level Request
    ↓
Scene Load
    ↓
Level Initialization
```

Content tabanlı:

```text id="v5n8qb"
Level Request
    ↓
Load Level Definition
    ↓
Instantiate / Pool Content
    ↓
Initialize Gameplay
```

Seçim projenin ihtiyaçlarına göre yapılmalıdır.

Sırf mimari uğruna tüm level'leri scene veya tüm level'leri prefab yapmak zorunlu değildir.

---

# 23. Level ve Input

Input doğrudan LevelSystem'e gameplay detaylarını bypass edecek şekilde bağlanmamalıdır.

Genel akış:

```text id="m4x7pd"
Input
 ↓
Input Controller
 ↓
Gameplay Command
 ↓
Gameplay System
 ↓
Level State
```

LevelSystem daha üst seviyede session lifecycle'ı yönetir.

Örneğin:

```text id="q8r2va"
Tap
 ↓
Board Command
 ↓
BoardController
 ↓
Board State
```

LevelSystem'in her input türünü bilmesi gerekmez.

---

# 24. Level ve UI

UI level state'ini gösterir.

Örneğin:

```text id="w6k9mc"
LevelSystem
    ↓
Current Level / Runtime State
    ↓
Event
    ↓
Gameplay UI
```

UI:

* Current level
* Objective progress
* Timer
* Moves
* Score

gösterebilir.

Ancak bu state'lerin sahibi değildir.

---

# 25. Level ve Pause

Pause global flow state'i olabilir:

```text id="p7c3xz"
GameFlow
    ↓
Pause
```

Pause sırasında LevelSystem/gameplay session pause edilebilir.

Örneğin:

```text id="k4n8vb"
GameFlow → Pause
       ↓
Gameplay Session Pause
```

Ancak LevelSystem'in global pause state'in sahibi olması gerekmez.

Gameplay subsystem'leri pause davranışlarını kendi sorumluluklarına göre uygulamalıdır.

---

# 26. Level ve Save

Level completion persistence:

```text id="s3m7qa"
LevelSystem
    ↓
Level Result
    ↓
ProgressionSystem
    ↓
SaveSystem
    ↓
PlayerSaveData
```

UI'ın:

```csharp id="f9v2kc"
saveData.CompletedLevels.Add(levelId);
```

yapması yanlıştır.

Level completion'ın kalıcı hale gelmesi progression/save katmanları üzerinden gerçekleşmelidir.

---

# 27. Temporary State ve Save State

Her gameplay state save edilmek zorunda değildir.

Örneğin:

```text id="d6q8xp"
Temporary:
- Current animation
- Current particle state
- Temporary VFX
- Input lock
- Runtime pool state
```

bunlar normalde save data değildir.

Kalıcı progression:

```text id="m5r1vz"
Persistent:
- Completed Levels
- Unlocked Content
- Player Progress
- Relevant Rewards
```

SaveSystem üzerinden saklanabilir.

---

# 28. Level Difficulty

Difficulty designer-tunable bir configuration olabilir.

Örneğin:

```text id="x7c4nb"
LevelConfig
    ↓
Difficulty Parameters
```

Difficulty'nin runtime hesaplanması gerekiyorsa ilgili gameplay system tarafından uygulanmalıdır.

LevelSystem'in içine bütün gameplay balancing logic'i doldurmak tercih edilmez.

---

# 29. Level Metadata

Level configuration'da presentation metadata bulunabilir:

```text id="q2m8sa"
Level Name
Thumbnail
Preview Image
Difficulty Label
```

Ancak bunlar gameplay runtime state değildir.

Örneğin level selection UI için:

```text id="w9p3kc"
LevelConfig
    ↓
Level Selection UI
```

kullanılabilir.

Gameplay başladığında ise:

```text id="h6r2vz"
LevelConfig
    ↓
LevelSystem
```

kullanılır.

---

# 30. Testing

LevelSystem mümkün olduğunca deterministic şekilde test edilebilir olmalıdır.

Özellikle:

* Level start
* Restart
* Completion
* Failure
* Result generation
* Invalid level handling
* Duplicate start request
* Cleanup
* Progression hand-off

test edilmelidir.

Saf level rules scene'e ihtiyaç duymuyorsa `EditMode` testleri tercih edilebilir.

Unity lifecycle ve scene integration gerektiren durumlar `PlayMode` testleriyle doğrulanabilir.

---

# 31. Invalid Level

Geçersiz level ID durumunda LevelSystem kontrollü şekilde hata vermelidir.

Örneğin:

```text id="c8v4mp"
Invalid LevelId
      ↓
Level Resolve Failed
      ↓
Error Handling
```

UI'ın null reference ile patlaması beklenen davranış değildir.

Recovery strategy projeye göre:

* Fallback level
* MainMenu'ya dönme
* Retry
* Error state

olabilir.

---

# 32. Duplicate Start

Aynı level için birden fazla `StartLevel()` çağrısı güvenli şekilde ele alınmalıdır.

Örneğin:

```text id="r7n2qx"
StartLevel(25)
StartLevel(25)
```

çağrıları sonucunda iki gameplay session oluşmamalıdır.

System:

```text id="m4c8za"
Idle
 ↓
Loading
 ↓
Starting
 ↓
Running
```

gibi bir lifecycle state tutabilir.

State zaten `Loading` veya `Running` ise ikinci request uygun şekilde ignore, queue veya reject edilebilir.

---

# 33. Async Level Loading

Level asset/content loading asynchronous ise operation lifecycle açıkça sahiplenilmelidir.

Örneğin:

```text id="v6p3ks"
Start Level
    ↓
Async Load
    ↓
Cancellation / Failure / Success
    ↓
Initialize
    ↓
Run
```

Level terk edilirse devam eden operation kontrol altında durdurulmalı veya sonucunun artık geçerli olmadığı belirlenmelidir.

Unmanaged async operation bırakılmamalıdır.

---

# 34. Performance

Level lifecycle içinde:

* Gereksiz Instantiate/Destroy kullanılmamalı
* Frequent gameplay objects pooling kullanmalı
* Level start sırasında ağır initialization kontrol edilmeli
* Hot path logic ile level orchestration ayrılmalı
* `Find` tabanlı dependency resolution kullanılmamalı
* Gereksiz allocation yapılmamalı
* Level restart sırasında eski runtime object'ler temizlenmeli

LevelSystem'in kendisi gameplay hot path'inde her frame çalışmak zorunda değildir.

Frame-by-frame gameplay logic ilgili gameplay system'lerinde bulunmalıdır.

---

# 35. Generic Template Rule

LevelSystem generic kalmalıdır.

Core template içinde belirli bir oyunun:

```text id="e4m8vz"
Kingdom
Stars
Buildings
Decorations
Lives
```

gibi kavramları zorunlu hale getirilmemelidir.

Bunun yerine:

```text id="q7x2pn"
Level
Objective
Progress
Result
Difficulty
Content
Session
```

gibi generic kavramlar kullanılmalıdır.

Oyuna özel kurallar gameplay module'lerinde tanımlanmalıdır.

---

# 36. Responsibility Example

Örnek bir match-style gameplay için:

```text id="t5c9wr"
LevelSystem
    ↓
Level 42
    ↓
BoardController
    ↓
Gameplay State
```

BoardController:

```text id="m8v3qa"
Board State
Tile State
Move Processing
```

LevelSystem:

```text id="j4r7xc"
Level Lifecycle
Start
Restart
Complete
Fail
```

GameFlow:

```text id="p6n2vz"
Gameplay
Win
Lose
Pause
```

ProgressionSystem:

```text id="x3q8ma"
Completed Levels
Unlocks
Player Progress
```

EconomySystem:

```text id="v7k4pb"
Currency
```

SaveSystem:

```text id="c9m5xd"
Persistence
```

Bu sınırlar feature büyüdükçe korunmalıdır.

---

# 37. Source of Truth

Level mimarisinin temel sahiplik modeli:

```text id="n6r2ya"
LevelConfig
    ↓
Static Level Definition

LevelSystem
    ↓
Current Level / Session Lifecycle

Gameplay Systems
    ↓
Gameplay Runtime State

ProgressionSystem
    ↓
Persistent Player Progress

EconomySystem
    ↓
Currency State

SaveSystem
    ↓
Persistence

AssetManagement
    ↓
Asset Loading / Lifetime

GameFlow
    ↓
Global Game State

UI
    ↓
Presentation
```

Temel kural:

> `GameFlow` oyunun **hangi global durumda olduğunu**, `LevelSystem` ise **hangi level session'ının nasıl ilerlediğini** yönetir.

---

# 38. Definition of Done

`LevelSystem` tamamlanmış sayılmadan önce:

* [ ] Level configuration ile runtime state ayrılmış mı?
* [ ] Current level state'in tek sahibi belli mi?
* [ ] Level lifecycle LevelSystem'de mi?
* [ ] GameFlow ile LevelSystem sorumlulukları ayrılmış mı?
* [ ] GameFlow global state yönetiyor mu?
* [ ] LevelSystem gameplay lifecycle yönetiyor mu?
* [ ] Win/lose koşulları UI'da bulunmuyor mu?
* [ ] Reward logic LevelSystem içine gereksiz şekilde gömülmemiş mi?
* [ ] Progression state ProgressionSystem tarafından mı sahipleniliyor?
* [ ] Currency EconomySystem tarafından mı sahipleniliyor?
* [ ] Save data UI veya gameplay tarafından doğrudan mutate edilmiyor mu?
* [ ] Restart eski runtime state'i temizliyor mu?
* [ ] Event subscription lifecycle güvenli mi?
* [ ] Async loading ownership ve cancellation güvenli mi?
* [ ] Duplicate StartLevel çağrıları kontrol ediliyor mu?
* [ ] Level asset loading AssetManagement ile uyumlu mu?
* [ ] Hot path allocation yapılmıyor mu?
* [ ] Gereksiz Instantiate/Destroy kullanılmıyor mu?
* [ ] Gameplay logic ilgili gameplay system'lerinde mi?
* [ ] LevelSystem gereksiz bir god object'e dönüşmemiş mi?
* [ ] Serialization güvenliği korunuyor mu?
* [ ] Generic template sınırları korunuyor mu?
* [ ] Gereksiz abstraction eklenmemiş mi?
