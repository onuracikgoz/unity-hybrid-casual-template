# Gameplay Module

## 1. Amaç

`GameplayModule`, oyunun gerçek gameplay mekaniklerinin template mimarisine bağlandığı feature katmanıdır.

Bu dokümanın amacı:

* Gameplay feature'larının mimaride nereye yerleşeceğini tanımlamak
* `LevelSystem` ile gameplay mechanics arasındaki sınırı belirlemek
* Gameplay state ownership kurallarını tanımlamak
* Input → Command → Gameplay akışını standartlaştırmak
* Gameplay module'lerinin birbirleriyle nasıl iletişim kuracağını belirlemek
* Gameplay-specific configuration ve runtime state ayrımını açıklamak
* Gameplay module'lerinin pooling, event, async ve performance kurallarını tanımlamak
* Template'in farklı oyun türlerine uyarlanmasını sağlamak

`GameplayModule`, template'in **oyuna özel kısmının ana extension point'idir**.

---

# 2. Gameplay Module Nedir?

Gameplay module, oyuncunun oyunda yaptığı temel aksiyonları ve bu aksiyonların sonuçlarını yöneten feature/system grubudur.

Örneğin bir oyunda:

```text id="v8m3qa"
BoardController
MatchSystem
MoveSystem
ObjectiveSystem
```

bulunabilir.

Başka bir oyunda:

```text id="r4k7px"
CharacterController
CombatSystem
EnemySpawner
AbilitySystem
```

bulunabilir.

Başka bir oyunda:

```text id="n6c2zw"
MergeSystem
ItemSpawner
InventorySystem
ProductionSystem
```

bulunabilir.

Bunların hiçbiri generic template core'a zorunlu olarak dahil edilmemelidir.

---

# 3. Temel Mimari

Gameplay module'ün genel konumu:

```text id="q5r8mc"
GameFlow
    ↓
LevelSystem
    ↓
Gameplay Module
    ↓
Gameplay Runtime State
```

Input akışı:

```text id="w7p2za"
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

UI:

```text id="k4m9vx"
Gameplay State / Event
        ↓
Gameplay UI
```

UI gameplay state'in sahibi değildir.

---

# 4. Gameplay Module Sorumlulukları

Gameplay module aşağıdaki sorumluluklara sahip olabilir:

* Gameplay state yönetmek
* Gameplay command'lerini işlemek
* Gameplay kurallarını uygulamak
* Gameplay action sonuçlarını üretmek
* Gameplay objective'lerini ilerletmek
* Gameplay-specific event'ler yayınlamak
* Gameplay-specific runtime object'leri yönetmek
* LevelSystem ile gameplay session lifecycle'ını entegre etmek

Gameplay module aşağıdakilerin sahibi değildir:

* Global GameFlow
* Persistent save data
* Currency
* Monetization
* Analytics SDK
* UI presentation
* Global navigation
* Genel asset loading policy

---

# 5. LevelSystem ve GameplayModule Ayrımı

Bu ayrım template'in en önemli sınırlarından biridir.

`LevelSystem`:

```text id="m3x7qa"
Which level?
When does it start?
When does it end?
What is the result?
```

sorularına cevap verir.

GameplayModule:

```text id="p8v2kc"
What does the player do?
How does the game mechanic work?
What happens after an action?
```

sorularına cevap verir.

Örnek:

```text id="r5n9wb"
LevelSystem
    ↓
Start Level 25
    ↓
BoardGameplayModule
    ↓
Initialize Board
```

LevelSystem board match kurallarını bilmemelidir.

---

# 6. Gameplay Module Örneği

Match-style bir oyun için:

```text id="c7m2vx"
GameplayModule
├── BoardController
├── MatchSystem
├── MoveSystem
└── ObjectiveSystem
```

Combat-style bir oyun için:

```text id="n4q8za"
GameplayModule
├── CharacterController
├── CombatSystem
├── EnemySystem
└── ObjectiveSystem
```

Runner-style bir oyun için:

```text id="p6r3yc"
GameplayModule
├── RunnerController
├── ObstacleSystem
├── SpawnSystem
└── ObjectiveSystem
```

Aynı template core kullanılabilir.

---

# 7. Tek GameplayModule mı, Birden Fazla mı?

Basit projede:

```text id="w2k7mx"
GameplayModule
```

tek bir orchestration component olabilir.

Daha karmaşık projede:

```text id="f9c3va"
GameplayModule
├── MovementSystem
├── CombatSystem
├── EnemySystem
├── ObjectiveSystem
└── SpawnSystem
```

şeklinde ayrılabilir.

Ancak her küçük davranış için yeni system oluşturmak zorunlu değildir.

Amaç:

> Sorumlulukları netleştirmek, class sayısını artırmak değildir.

---

# 8. Gameplay State Ownership

Her önemli runtime state'in tek bir sahibi olmalıdır.

Örneğin:

```text id="q8m4zc"
BoardController
    ↓
Board State
```

```text id="v5r2xa"
CombatSystem
    ↓
Combat State
```

```text id="k7n3wp"
ObjectiveSystem
    ↓
Objective Progress
```

```text id="m4c9zb"
SpawnSystem
    ↓
Spawn State
```

Başka component'ler bu state'i kullanabilir.

Ancak aynı state'i ikinci kez sahiplenmemelidir.

---

# 9. Gameplay Configuration

Designer tarafından değiştirilebilen gameplay değerleri configuration olarak tanımlanmalıdır.

Örneğin:

```text id="x3p7qa"
GameplayConfig
├── Rules
├── Balance
├── Spawn Settings
├── Objective Definitions
├── Movement Parameters
└── Visual References
```

Ancak gerçek yapı gameplay türüne göre değişir.

Örneğin combat için:

```text id="d8m2vk"
Character Stats
Enemy Definitions
Ability Definitions
```

gerekebilir.

Match-style gameplay için:

```text id="q5n8xa"
Board Size
Match Rules
Move Rules
Objective Definitions
```

gerekebilir.

---

# 10. ScriptableObject ve Gameplay Configuration

ScriptableObject gameplay configuration için kullanılabilir.

Örneğin:

```csharp id="r7c2mb"
[CreateAssetMenu]
public class GameplayConfig : ScriptableObject
{
    public float ActionDuration;
    public int MaxActions;
}
```

Bu data:

```text id="n4v8zp"
Static Configuration
```

olmalıdır.

Runtime state:

```text id="x6k3qa"
CurrentActions
CurrentCombo
CurrentObjectiveProgress
```

gibi değerleri ScriptableObject üzerinde mutable olarak tutmamalıdır.

---

# 11. Gameplay Session

Gameplay module bir level session'ı sırasında aktif olabilir.

Akış:

```text id="m8q4vc"
LevelSystem.StartLevel()
        ↓
GameplayModule.Initialize()
        ↓
GameplayModule.StartSession()
        ↓
Running
        ↓
Gameplay Result
        ↓
GameplayModule.StopSession()
```

Session-specific state level lifecycle'a bağlıdır.

Level restart edildiğinde yeni session oluşturulmalıdır.

---

# 12. Initialize ve Start Ayrımı

Gameplay module initialization ile gameplay'in başlaması ayrılabilir.

Örneğin:

```text id="k2v7pn"
Initialize
    ↓
Resolve Configuration
    ↓
Cache Dependencies
    ↓
Prepare Runtime
```

ardından:

```text id="c5m9xa"
Start Session
    ↓
Spawn / Setup
    ↓
Enable Input
    ↓
Running
```

Bu ayrım özellikle async level loading veya bootstrap sonrası initialization karmaşık olduğunda faydalıdır.

Basit projelerde daha küçük bir lifecycle yeterli olabilir.

---

# 13. Input Pipeline

Gameplay input için önerilen pipeline:

```text id="p4x8za"
Touch / Mouse / Keyboard / Gamepad
                ↓
          Input Controller
                ↓
        Gameplay Command
                ↓
         Gameplay Module
                ↓
          Gameplay State
```

Gameplay module doğrudan cihaz API'sine bağımlı olmamalıdır.

Örneğin gameplay logic:

```csharp id="n7m2vc"
HandleMove(position);
```

gibi bir gameplay command alabilir.

Touch handling:

```text id="x8q4pa"
Touch Input
```

input layer'a ait olmalıdır.

---

# 14. Gameplay Command

Command gameplay action'ını ifade eder.

Örneğin:

```text id="f6r9km"
SelectTarget
Move
Attack
Merge
Place
UseAbility
Collect
```

gibi action'lar olabilir.

Command:

```text id="w3c7xa"
Gameplay Intent
```

anlamına gelir.

Command'in gerçekten uygulanıp uygulanamayacağı gameplay module tarafından belirlenir.

---

# 15. Command Validation

Gameplay command validation gameplay logic'e aittir.

Örneğin:

```text id="m5q8vr"
Move Command
    ↓
Can Move?
    ├── No → Reject
    └── Yes → Execute
```

UI'ın:

```csharp id="z7n3kp"
if (moves > 0)
```

gibi gameplay rule validation yapması tercih edilmez.

UI yalnızca input intent üretir.

---

# 16. Gameplay Action Lifecycle

Bir gameplay action şu lifecycle'a sahip olabilir:

```text id="q9c4mx"
Command Received
      ↓
Validate
      ↓
Execute
      ↓
Update State
      ↓
Resolve Consequences
      ↓
Publish Result
```

Örneğin:

```text id="v6k2za"
Attack Command
    ↓
Validate
    ↓
Apply Damage
    ↓
Enemy Defeated
    ↓
Objective Updated
    ↓
Gameplay Event
```

Bu zincir gameplay module içinde veya ilgili subsystem'lerde yönetilir.

---

# 17. Gameplay Result

Gameplay action sonucu ilgili system'lere event veya result olarak bildirilebilir.

Örneğin:

```text id="p3m7xc"
MoveResult
AttackResult
MergeResult
ObjectiveResult
```

Sonuçlar yalnızca ihtiyaç varsa oluşturulmalıdır.

Basit synchronous method return değeri yeterliyse event sistemi zorunlu değildir.

---

# 18. Event Kullanımı

Gameplay event'leri gerçekten decoupling gerekiyorsa kullanılmalıdır.

Örneğin:

```text id="r8x2vn"
EnemyDefeated
       ↓
ObjectiveSystem
       ↓
Objective Progress
```

ve:

```text id="m4c7za"
ObjectiveCompleted
       ↓
LevelSystem
       ↓
Level Result
```

EventBus bütün gameplay communication'ın zorunlu taşıyıcısı değildir.

Aynı owner'a ait açık method çağrısı daha basitse direct reference tercih edilebilir.

---

# 19. Gameplay Module Communication

İletişim için karar sırası:

```text id="k7p3xa"
1. Direct Reference
2. Local Event / Callback
3. Shared Event System
```

Örneğin:

```text id="q2m8vc"
GameplayModule
    ↓
ObjectiveSystem
```

açık ownership ilişkisi varsa direct reference kullanılabilir.

Ancak:

```text id="n5r9zb"
Gameplay
    ↓
Analytics
```

gibi decoupled communication için event/system sınırı daha uygun olabilir.

---

# 20. Gameplay ve Objective

ObjectiveSystem varsa objective state'in sahibi o olmalıdır.

Örneğin:

```text id="c8x4mp"
Gameplay Action
    ↓
ObjectiveSystem
    ↓
Objective Progress
```

Gameplay module objective UI'ını yönetmez.

Örneğin:

```text id="v3q7ka"
ObjectiveSystem
    ↓
ObjectiveChanged
    ↓
Gameplay UI
```

UI yalnızca presentation yapar.

---

# 21. Gameplay ve Score

Score sistemi ayrıysa score state'in sahibi `ScoreSystem` olabilir.

Örneğin:

```text id="m6r2xb"
Gameplay Result
    ↓
ScoreSystem
    ↓
Score State
```

Gameplay module score değerini kendi içinde ikinci kez tutmamalıdır.

Eğer score gameplay module'ün ayrılmaz bir parçasıysa module içinde bulunabilir.

Bu karar gerçek ownership'e göre verilmelidir.

---

# 22. Gameplay ve Economy

Gameplay reward doğurabilir:

```text id="q4n8vc"
Gameplay Completed
    ↓
RewardSystem
    ↓
EconomySystem
```

Gameplay module doğrudan:

```csharp id="w9m3ka"
economy.AddCurrency(100);
```

yapmamalıdır, eğer reward/economy sistemi bu sorumluluğu zaten sahipleniyorsa.

Gameplay yalnızca anlamlı sonucu üretir.

---

# 23. Gameplay ve Progression

Gameplay sonucu progression'a aktarılabilir:

```text id="f7c2xp"
Level Completed
    ↓
ProgressionSystem
```

Gameplay module:

```text id="j5m8qa"
Unlock Next Level
```

gibi persistent progression kurallarını kendi içine gömmemelidir.

---

# 24. Gameplay ve Save

Gameplay runtime state normalde save data değildir.

Örneğin:

```text id="v4r7mc"
Current Animation
Current Spawn Timer
Temporary Combo
Active Projectile
```

save edilmeyebilir.

Persistence gerektiren state:

```text id="q8n3za"
Completed Level
Unlocked Content
Player Progress
```

SaveSystem üzerinden tutulmalıdır.

Eğer oyunda mid-level resume gerçekten gerekiyorsa bunun için explicit bir save model tasarlanmalıdır.

---

# 25. Gameplay Object Lifecycle

Gameplay runtime object'leri:

```text id="m2x7pc"
Create
Initialize
Run
Reset
Release
```

lifecycle'ına sahip olmalıdır.

Pooling kullanılan object'lerde:

```text id="k9v4za"
OnSpawn
    ↓
Reset
    ↓
Initialize
```

ve:

```text id="p6c3xm"
Release
    ↓
Cleanup
    ↓
Return To Pool
```

uygulanmalıdır.

---

# 26. Pooling

Sık oluşturulan gameplay object'leri pooling kullanmalıdır.

Örneğin:

```text id="x5m8qb"
Projectile
Enemy
Collectible
VFX
Temporary Object
Tile
```

Pooling kararı object'in gerçek kullanım pattern'ine göre verilmelidir.

Bir kez oluşturulan static gameplay object için pooling zorunlu değildir.

---

# 27. Pool Reset

Pooled gameplay object tamamen resetlenmelidir.

Kontrol edilmesi gerekenler:

```text id="r3q7va"
Position
Rotation
Scale
Velocity
Animation
Timers
Flags
Health
Target
Runtime Data
Subscriptions
Visual State
```

Örneğin enemy pool'a döndüğünde:

```text id="w8n2mc"
Health = MaxHealth
Target = null
Velocity = zero
IsDead = false
```

gibi gerekli state'ler yeniden kurulmalıdır.

---

# 28. Gameplay Animation

Gameplay animation presentation olsa da bazı animation state'leri gameplay sonucunu etkileyebilir.

Bu durumda gameplay state animation'dan bağımsız tutulmalıdır.

Örneğin:

```text id="c4m9xp"
Attack Requested
    ↓
Gameplay State = Attack Executed
    ↓
Animation = Presentation
```

Gameplay:

```text id="n7r2za"
Attack Animation Finished
```

event'ine gereksiz şekilde bağımlı hale getirilmemelidir.

Animation gerçekten gameplay timing'in bir parçasıysa açık bir gameplay event/command ile modellenmelidir.

---

# 29. Tween ve Gameplay

Tween'ler gameplay logic'in source of truth'u olmamalıdır.

Yanlış:

```text id="f6p3vc"
Tween Complete
    ↓
Assume Gameplay Completed
```

Daha güvenli:

```text id="q9m5xa"
Gameplay State Changed
    ↓
Tween Presentation
```

Tween cleanup:

```text id="v2k8nr"
OnDisable
OnDestroy
Pool Release
```

durumlarında yapılmalıdır.

---

# 30. Timing

Gameplay timing için:

* Coroutine
* Task
* async/await
* UniTask
* Timer system

gibi yaklaşımlardan proje standardı neyse o kullanılmalıdır.

Aynı feature içinde gereksiz biçimde farklı async modelleri karıştırılmamalıdır.

Uzun süre yaşayan operation'ların sahibi belli olmalıdır.

Level restart veya gameplay shutdown sırasında pending operation'lar güvenli şekilde iptal edilmeli veya geçersiz hale getirilmelidir.

---

# 31. Update Kullanımı

Gameplay'de `Update()` yalnızca gerçekten frame-by-frame logic gerekiyorsa kullanılmalıdır.

Örneğin:

```text id="m4x7qa"
Continuous Movement
Real-time Timer
Physics-like Simulation
```

gibi durumlarda gerekli olabilir.

Ancak:

```csharp id="p7n3vz"
Update()
{
    CheckWin();
    RefreshUI();
    FindTargets();
}
```

gibi sürekli polling tercih edilmemelidir.

Event/state transition tabanlı yaklaşım kullanılmalıdır.

---

# 32. Hot Path Performance

Gameplay hot path'lerinde:

* LINQ
* Closure
* Boxing
* Gereksiz delegate allocation
* Temporary collection
* String concatenation
* Reflection
* Repeated `GetComponent`
* `Find`
* Instantiate
* Destroy

gereksiz yere kullanılmamalıdır.

Özellikle:

```text id="c8r2mw"
Update
FixedUpdate
LateUpdate
Input
Collision
Spawn
Board Resolution
AI
```

gibi sık çalışan alanlarda allocation dikkatle kontrol edilmelidir.

---

# 33. Dependency Resolution

Gameplay module dependency'lerini:

```text id="w5q9xb"
GameObject.Find
FindObjectOfType
FindFirstObjectByType
```

ile runtime'da keşfetmek tercih edilmez.

Dependency:

```text id="m7c3za"
Serialized Reference
Explicit Initialization
Existing Dependency Provider
```

gibi yöntemlerle sağlanmalıdır.

Gereksiz dependency injection framework'ü eklenmemelidir.

---

# 34. Gameplay Scene

Gameplay scene kullanılabilir.

Örneğin:

```text id="p4n8vc"
Gameplay Scene
├── GameplayRoot
├── GameplayModule
├── Camera
├── Gameplay Content
└── Presentation
```

Ancak scene hierarchy gameplay architecture'ın kendisi değildir.

Scene'deki GameObject sayısı ile system responsibility birebir eşleşmek zorunda değildir.

---

# 35. Gameplay Root

Gameplay-specific object'leri ortak bir root altında toplamak faydalı olabilir.

Örneğin:

```text id="x2m7qa"
GameplayRoot
    ├── GameplayModule
    ├── Runtime Content
    ├── Spawned Objects
    └── Gameplay Presentation
```

Level restart veya cleanup sırasında bu root'un lifecycle'ı yönetilebilir.

Ancak global persistent system'ler bu root'un altında tutulmamalıdır.

---

# 36. Camera

Camera gameplay'e özel olabilir.

Örneğin:

```text id="n6r3xm"
Gameplay Camera
```

gameplay presentation veya scene layer'da bulunabilir.

Camera logic'in gameplay state ownership'ine dönüşmemesine dikkat edilmelidir.

Örneğin camera:

```text id="q8v4kc"
Follow Target
Zoom
Shake
```

yönetebilir.

Ancak:

```text id="f5m9za"
Camera detects target → Level Won
```

gibi gameplay rules camera'ya verilmemelidir.

---

# 37. Gameplay Presentation

Gameplay presentation ayrı tutulmalıdır.

Örneğin:

```text id="r7x2mb"
Gameplay State
      ↓
Gameplay Presentation
      ↓
Visual Result
```

Gameplay system:

```text id="k3p8vc"
Enemy defeated
```

der.

Presentation:

```text id="m5q9xa"
Play death animation
Show VFX
```

yapar.

Bu ayrım gameplay logic'in test edilebilirliğini artırır.

---

# 38. Input Lock

Gameplay sırasında input geçici olarak kilitlenebilir.

Örneğin:

```text id="v8c2pn"
Action In Progress
    ↓
Input Policy
    ↓
Reject New Commands
```

Bu state gameplay/input katmanında yönetilmelidir.

UI button'larını tek tek disable etmek:

```text id="j4m7qa"
Button A disabled
Button B disabled
Button C disabled
```

şeklinde dağınık bir çözüm haline gelmemelidir.

---

# 39. Gameplay State Machine

Gameplay complexity arttığında state machine kullanılabilir.

Örneğin:

```text id="q6x3mz"
Idle
 ↓
Input
 ↓
Resolving
 ↓
Animating / Processing
 ↓
Checking Result
 ↓
Idle
```

veya:

```text id="p9r4vc"
Ready
 ↓
Playing
 ↓
Completed
```

Ancak her gameplay için FSM zorunlu değildir.

Basit state boolean/enum ile güvenli şekilde ifade edilebiliyorsa gereksiz abstraction oluşturulmamalıdır.

---

# 40. Multiple Gameplay Modules

Bir level birden fazla module kullanabilir.

Örneğin:

```text id="c5m8xa"
LevelSystem
    ↓
GameplaySession
    ├── CharacterSystem
    ├── CombatSystem
    ├── ObjectiveSystem
    └── SpawnSystem
```

Bu durumda her module'ün ownership'i açık olmalıdır.

Örneğin:

```text id="r8q2vn"
CombatSystem
    → Combat State

SpawnSystem
    → Spawn State

ObjectiveSystem
    → Objective State
```

Tek bir `GameplayManager` içine her şeyi doldurmak tercih edilmez.

---

# 41. GameplayManager Kullanımı

`GameplayManager` yalnızca gerçek orchestration ihtiyacı varsa kullanılmalıdır.

Örneğin:

```text id="m7c4xa"
GameplayManager
    ↓
Initialize Modules
    ↓
Start Session
    ↓
Stop Session
```

kabul edilebilir.

Ancak:

```text id="q3n8vp"
GameplayManager
    ├── Score
    ├── Economy
    ├── Save
    ├── Audio
    ├── UI
    ├── Ads
    ├── Tutorial
    └── Level
```

gibi bir god object oluşturulmamalıdır.

---

# 42. Generic Gameplay Interface

Bazı projelerde ortak lifecycle abstraction'ı faydalı olabilir.

Örneğin:

```csharp id="v6r2mx"
public interface IGameplayModule
{
    void Initialize();
    void StartSession();
    void StopSession();
}
```

Ancak bu interface yalnızca gerçekten birden fazla module aynı lifecycle contract'ını paylaşıyorsa kullanılmalıdır.

Sırf "modüler mimari" görünsün diye interface eklenmemelidir.

---

# 43. Module Dependencies

Bir gameplay module başka bir module'e ihtiyaç duyuyorsa dependency açık olmalıdır.

Örneğin:

```text id="k8p4za"
CombatSystem
    ↓
TargetSystem
```

Bu dependency:

* Explicit reference
* Initialization parameter
* Existing system provider

gibi sade yöntemlerden biriyle sağlanabilir.

Her dependency için EventBus kullanılması doğru değildir.

---

# 44. Gameplay Event Naming

Gameplay event'leri gerçekleşen olayı ifade etmelidir.

İyi:

```text id="r4m7xc"
EnemyDefeated
MoveExecuted
ObjectiveCompleted
ItemCollected
AbilityUsed
```

Kötü:

```text id="x8q2vn"
DoSomething
UpdateEverything
RefreshGame
```

Event isimleri implementation detayını değil anlamlı domain olayını ifade etmelidir.

---

# 45. Gameplay ve Tutorial

Tutorial gameplay module'ünü yeniden implement etmez.

Örneğin:

```text id="m3v8qa"
TutorialSystem
    ↓
Expected Action
    ↓
Gameplay Command
    ↓
GameplayModule
    ↓
Action Result
    ↓
TutorialSystem
```

Tutorial'ın gerçek gameplay davranışını bypass etmesine izin verilmemelidir.

---

# 46. Gameplay ve Audio

Gameplay module ses efektlerinin implementation detayını doğrudan yönetmemelidir.

Örneğin:

```text id="q7c2mx"
Gameplay Event
    ↓
AudioSystem
    ↓
Audio Asset
```

kullanılabilir.

Gameplay:

```csharp id="n5r8va"
audioSource.Play();
```

yapmak zorunda değildir.

Ancak gameplay object'i açıkça kendi presentation audio'sunun owner'ıysa direct reference kabul edilebilir.

Amaç gereksiz abstraction değil, responsibility clarity'dir.

---

# 47. Gameplay ve Haptic

Haptic feedback gerekiyorsa:

```text id="v4m9xa"
Gameplay Event
    ↓
HapticSystem
```

yaklaşımı kullanılabilir.

Gameplay module doğrudan platform-specific haptic API'sine bağımlı olmamalıdır.

---

# 48. Gameplay ve Analytics

Gameplay event'leri analytics için kullanılabilir:

```text id="p6x3zk"
Gameplay Event
    ↓
AnalyticsSystem
```

Gameplay module doğrudan analytics SDK'sına bağlanmamalıdır.

Analytics event'leri anlamlı domain event'lerinden türetilmelidir.

---

# 49. Error Handling

Gameplay module beklenmeyen durumlarda kontrollü davranmalıdır.

Örneğin:

```text id="c8r4mb"
Invalid Command
    ↓
Reject
```

veya:

```text id="n5q7xa"
Missing Gameplay Configuration
    ↓
Initialization Failure
```

Runtime null reference ile zincirleme şekilde patlamak yerine mümkün olduğunda açık validation yapılmalıdır.

Development sırasında configuration hataları erken ve görünür şekilde raporlanmalıdır.

---

# 50. Testing

Gameplay logic mümkün olduğunca scene'den bağımsız test edilebilir olmalıdır.

Özellikle:

* Command validation
* State transitions
* Gameplay rules
* Objective progression
* Win conditions
* Lose conditions
* Spawn rules
* Combat calculations
* Resource calculations

gibi deterministic logic'ler `EditMode` testlerine uygun tasarlanmalıdır.

Unity-specific integration:

* Physics
* Animator
* Scene lifecycle
* Prefab references
* Pool integration

gibi konular `PlayMode` testleriyle doğrulanabilir.

---

# 51. Determinism

Gameplay mümkün olduğunca deterministic sonuç üretmelidir.

Örneğin aynı:

```text id="w2m8vc"
Initial State
+
Same Command Sequence
```

aynı gameplay sonucunu üretmelidir.

Randomness gerekiyorsa random state/seed yönetimi gerektiğinde açıkça tasarlanmalıdır.

Bu özellikle:

* Replay
* Debugging
* Testing
* Level validation

için faydalıdır.

Her projede tam deterministic simulation zorunlu değildir.

---

# 52. Randomness

Random gameplay behavior varsa random üretimi dağınık şekilde farklı component'lere bırakılmamalıdır.

Örneğin:

```text id="q9r4xa"
Random Provider / Controlled Random Source
        ↓
Gameplay System
```

kullanılabilir.

Ancak yalnızca ihtiyaç varsa abstraction eklenmelidir.

Basit bir `Random.Range` kullanımı için gereksiz random service oluşturulmamalıdır.

---

# 53. Serialization Safety

Gameplay configuration veya prefab serialized data değiştiriliyorsa dikkat edilmelidir.

Özellikle:

* Serialized field rename
* Enum values
* ScriptableObject structure
* Prefab references
* Level configuration references
* Gameplay content IDs

kontrollü değiştirilmelidir.

Gerekli durumlarda:

```csharp id="h7m3vc"
[FormerlySerializedAs("oldField")]
```

kullanılmalıdır.

---

# 54. Generic Template Rule

Template'in generic core'u gameplay module'ünün kendisini değil, **extension modelini** tanımlar.

Core içinde zorunlu olarak:

```text id="p4n8xm"
BoardController
MatchSystem
KingdomSystem
CombatSystem
MergeSystem
RunnerSystem
```

bulundurulmamalıdır.

Bunun yerine:

```text id="v8q3za"
GameplayModule
Gameplay Session
Gameplay Command
Gameplay State
Gameplay Result
Objective
```

gibi generic kavramlar kullanılmalıdır.

Gerçek gameplay içeriği projeye göre eklenir.

---

# 55. Örnek: Match Gameplay

Bir match-style oyun için olası yapı:

```text id="m6c2xr"
GameplayModule
├── BoardController
│   └── Board State
│
├── InputHandler
│
├── MatchSystem
│   └── Match Resolution
│
├── ObjectiveSystem
│   └── Objective State
│
└── MoveSystem
    └── Move State
```

Akış:

```text id="q7r4va"
Touch
 ↓
Input Controller
 ↓
Select / Swap Command
 ↓
BoardController
 ↓
MatchSystem
 ↓
Resolve
 ↓
ObjectiveSystem
 ↓
LevelSystem
```

Bu yalnızca örnektir.

Template'in match-3'e özel hale gelmesi anlamına gelmez.

---

# 56. Örnek: Combat Gameplay

Combat oyunu için:

```text id="x3p8mc"
GameplayModule
├── CharacterSystem
├── CombatSystem
├── EnemySystem
├── AbilitySystem
└── ObjectiveSystem
```

Akış:

```text id="n5v2za"
Input
 ↓
Attack Command
 ↓
CombatSystem
 ↓
Damage
 ↓
EnemySystem
 ↓
Enemy Defeated
 ↓
ObjectiveSystem
 ↓
LevelSystem
```

UI bu sistemlerin state'lerini gösterir.

---

# 57. Örnek: Merge Gameplay

Merge oyunu için:

```text id="r8m4xc"
GameplayModule
├── GridSystem
├── MergeSystem
├── SpawnSystem
└── ObjectiveSystem
```

Akış:

```text id="k6q2vp"
Drag / Drop
 ↓
Gameplay Command
 ↓
GridSystem
 ↓
MergeSystem
 ↓
Merged Result
 ↓
ObjectiveSystem
```

Yine aynı LevelSystem/GameFlow altyapısı kullanılabilir.

---

# 58. Definition of Done

Bir `GameplayModule` tamamlanmış sayılmadan önce:

* [ ] Gameplay responsibility açıkça tanımlanmış mı?
* [ ] Level lifecycle LevelSystem'de kalıyor mu?
* [ ] Global state GameFlow'da kalıyor mu?
* [ ] Gameplay runtime state'in owner'ı belli mi?
* [ ] Configuration ile runtime state ayrılmış mı?
* [ ] ScriptableObject üzerinde mutable session state tutulmuyor mu?
* [ ] Input ile gameplay logic ayrılmış mı?
* [ ] Gameplay command modeli net mi?
* [ ] Command validation gameplay layer'da mı?
* [ ] UI gameplay rules hesaplamıyor mu?
* [ ] Win/lose logic doğru owner'da mı?
* [ ] Objective state tek bir owner tarafından mı yönetiliyor?
* [ ] Economy doğrudan gameplay module içine gömülmemiş mi?
* [ ] Progression doğrudan gameplay module içine gömülmemiş mi?
* [ ] Save data doğrudan mutate edilmiyor mu?
* [ ] Analytics SDK doğrudan çağrılmıyor mu?
* [ ] Audio/Haptic platform implementation gameplay'e gömülmemiş mi?
* [ ] Event kullanımı gerçekten gerekli mi?
* [ ] Direct reference kullanılabilecek yerde gereksiz EventBus kullanılmıyor mu?
* [ ] Runtime object lifecycle açık mı?
* [ ] Pool kullanılıyorsa reset eksiksiz mi?
* [ ] Hot path allocation kontrol edilmiş mi?
* [ ] Gereksiz `Find`/`GetComponent`/Instantiate/Destroy kullanılmıyor mu?
* [ ] Async/timing operation lifecycle güvenli mi?
* [ ] Restart ve cleanup davranışı tanımlı mı?
* [ ] Gameplay logic mümkün olduğunca test edilebilir mi?
* [ ] Serialization güvenliği korunuyor mu?
* [ ] Generic template sınırları korunuyor mu?
* [ ] Gereksiz abstraction veya manager eklenmemiş mi?
