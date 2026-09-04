# BOOTSTRAP.md

Bu doküman, uygulamanın başlatılması sırasında core sistemlerin nasıl oluşturulduğunu, initialize edildiğini ve global game flow'a nasıl geçildiğini tanımlar.

Bootstrap'in amacı gameplay başlatmak değil, oyunun çalışması için gerekli temel altyapıyı deterministik ve kontrollü şekilde hazırlamaktır.

---

# 1. Bootstrap Responsibilities

Bootstrap yalnızca başlangıç orchestration'ından sorumludur.

Bootstrap:

* Core sistemlerin hazır olmasını sağlar.
* Gerekli configuration'ların yüklenmesini sağlar.
* Persistent player data'nın yüklenmesini başlatır veya bekler.
* Gerekli servislerin initialization sırasını yönetir.
* Global game flow'un başlayabileceği noktayı belirler.

Bootstrap:

* Gameplay mechanic yönetmez.
* Level logic yönetmez.
* UI presentation yönetmez.
* Economy business logic içermez.
* Reward calculation yapmaz.
* Tutorial logic içermez.

---

# 2. Startup Flow

Temel startup flow:

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

Gerçek projede bazı adımlar paralel veya farklı sırada çalışabilir.

Ancak dependency gerektiren sistemler hazır olmadan onlara bağımlı sistemlerin çalışmasına izin verilmemelidir.

---

# 3. Initialization Layers

Başlangıç süreci genel olarak üç katmana ayrılır.

## Layer 1: Core Infrastructure

Oyunun diğer sistemleri için temel altyapıyı oluşturur.

Örnek:

```text
EventSystem
Pooling
AudioSystem
Configuration Access
```

## Layer 2: Persistent / Runtime Systems

Oyuncu ve oyun state'ini yöneten sistemler hazırlanır.

Örnek:

```text
SaveSystem
EconomySystem
ProgressionSystem
MonetizationSystem
AnalyticsSystem
```

## Layer 3: Game Flow

Core sistemler hazır olduktan sonra global game flow başlatılır.

```text
GameFlow
    ↓
MainMenu
```

---

# 4. Initialization Order

Sistemler arasındaki dependency'ler initialization sırasını belirler.

Örnek:

```text
EventSystem
    ↓
Configuration
    ↓
SaveSystem
    ↓
Runtime Systems
    ↓
GameFlow
```

Bu sıra sabit bir zorunluluk değildir.

Bir sistemin başka bir sisteme ihtiyacı yoksa gereksiz şekilde dependency oluşturma.

Örneğin:

```text
AudioSystem
```

EconomySystem'in initialize olmasını beklememelidir.

---

# 5. Unity Lifecycle vs Bootstrap

Unity lifecycle ile application initialization aynı şey değildir.

Örneğin:

```text
Awake()
→ Local reference caching

Start()
→ Component initialization

Bootstrap
→ Application-level initialization
```

Bir sistemin `Awake()` içinde olması gereken initialization'ı Bootstrap'e taşıma.

Aynı şekilde application-level dependency'leri yalnızca Unity lifecycle sırasına güvenerek yönetme.

---

# 6. Persistent Objects

Global olarak application lifetime boyunca yaşaması gereken sistemler kontrollü şekilde persistent olabilir.

Örnek:

```text
GameFlow
SaveSystem
AudioSystem
AnalyticsSystem
```

Ancak her system için otomatik olarak:

```csharp
DontDestroyOnLoad(gameObject);
```

kullanma.

Persistent lifetime gerçekten gerekli olmalıdır.

Persistent object'lerin:

* Duplicate oluşmaması
* Lifecycle'ının belirli olması
* Scene değişimlerinde davranışının tanımlı olması

gerekir.

---

# 7. Bootstrap Scene

Projede dedicated bootstrap scene kullanılıyorsa bu scene yalnızca gerekli initialization infrastructure'ını içermelidir.

Örnek:

```text
Bootstrap Scene
├── Bootstrap
├── GameFlow
├── Core Systems
└── Persistent Systems
```

Gameplay scene'leri bootstrap scene'in implementation detaylarına bağımlı olmamalıdır.

Bootstrap scene'in görevi:

```text
Create / Initialize
        ↓
Prepare
        ↓
Enter Game Flow
```

olmalıdır.

---

# 8. Configuration Initialization

Designer-tunable configuration'lar runtime başlamadan önce erişilebilir durumda olmalıdır.

Örnek:

```text
EconomyConfig
LevelConfig
ProgressionConfig
AudioConfig
GameplayConfig
```

Configuration:

```text
ScriptableObject
```

üzerinden sağlanabilir.

Configuration'ın runtime sırasında değiştirilmesi gerekiyorsa bunun runtime state'e kopyalanması veya uygun bir runtime model kullanılması değerlendirilmelidir.

ScriptableObject üzerinde session state tutma.

---

# 9. Save Data Initialization

Persistent player data startup sırasında yüklenir.

Genel flow:

```text
SaveSystem
    ↓
Load PlayerSaveData
    ↓
Validate / Migrate
    ↓
Runtime Systems
    ↓
Game Ready
```

Save data hazır olmadan player-dependent runtime systems çalıştırılmamalıdır.

Örneğin:

```text
EconomySystem
ProgressionSystem
```

player save data'ya ihtiyaç duyuyorsa initialization sırasında bunu açıkça belirtmelidir.

---

# 10. First Launch

Oyuncunun ilk kez çalıştırdığı durumda save data bulunmayabilir.

Genel flow:

```text
Load Save
    ↓
Save Exists?
   / \
 No   Yes
 ↓     ↓
Create  Load
Default Data
   \   /
    ↓
Validate
    ↓
Initialize Runtime
```

Default data oluşturma SaveSystem'in persistence responsibility'si içerisinde kalmalıdır.

Gameplay system'leri "ilk kez oynuyor" kontrolünü kendi içerisinde tekrar tekrar yapmamalıdır.

---

# 11. Save Migration

Save data formatı değiştiğinde eski save'ler yeni modele migrate edilebilir.

Genel flow:

```text
Load Raw Save
      ↓
Version Check
      ↓
Migration
      ↓
Current Save Model
      ↓
Runtime Initialization
```

Migration logic'i gameplay system'lerine dağıtma.

Migration gerektiğinde `SaveSystem` veya ilgili save migration layer'ı sorumlu olmalıdır.

---

# 12. Async Initialization

Initialization async ise her operation'ın ownership ve lifecycle'ı belirli olmalıdır.

Örneğin:

```text
Load Save
    ↓
Await
    ↓
Initialize Systems
    ↓
Start Game Flow
```

Bootstrap:

* Unmanaged async operation başlatmamalıdır.
* Sonsuza kadar bekleyen operation oluşturmamalıdır.
* Cancellation gerektiğinde cancellation lifecycle sağlamalıdır.
* Aynı initialization'ın birden fazla kez çalışmasını engellemelidir.

Projede kullanılan async yaklaşımı tutarlı şekilde kullanılmalıdır.

---

# 13. Initialization Failure

Bir core system initialize edilemezse Bootstrap bunu sessizce geçmemelidir.

Örneğin:

```text
SaveSystem failed
      ↓
Game Ready ❌
```

Bu durumda sistemin failure policy'si uygulanmalıdır.

Olası davranışlar:

```text
Retry
Fallback
Safe Default
Error State
Block Startup
```

Failure behavior sistemin önemine göre belirlenmelidir.

Örneğin analytics initialization başarısız olduğunda gameplay'i tamamen durdurmak çoğu projede gerekli olmayabilir.

Ancak kritik player data yüklenemiyorsa oyunu başlatmak güvenli olmayabilir.

---

# 14. Initialization Idempotency

Initialization mümkün olduğunda kontrollü ve tekrar çağrıldığında beklenmeyen duplicate state oluşturmayan bir yapıda olmalıdır.

Özellikle:

```text
Event subscriptions
Persistent objects
SDK initialization
Save loading
Pool creation
```

gibi işlemlerde duplicate initialization engellenmelidir.

---

# 15. Scene Transition

Bootstrap tamamlandıktan sonra game flow ilgili state'e geçer.

Örnek:

```text
Bootstrap
    ↓
MainMenu State
    ↓
MainMenu Scene / UI
```

Gameplay scene yüklenmesi GameFlow veya ilgili scene/level orchestration sistemi tarafından yönetilebilir.

Bootstrap her scene transition'da tekrar çalışmamalıdır.

---

# 16. Bootstrap and GameFlow Relationship

Bootstrap ile GameFlow farklı sorumluluklara sahiptir.

```text
Bootstrap
→ "Oyun çalışmaya hazır mı?"

GameFlow
→ "Oyun şu anda hangi state'te?"
```

Örneğin:

```text
Bootstrap
    ↓
All required systems ready
    ↓
GameFlow.Start()
    ↓
MainMenu
```

GameFlow initialization tamamlanmadan gameplay state'e geçilmemelidir.

Detaylar:

`SYSTEMS/GameFlow.md`

---

# 17. Bootstrap and UI

Bootstrap UI presentation'ın sahibi değildir.

Bootstrap:

```text
Initialize
    ↓
Game Ready
```

UI:

```text
GameFlow State
    ↓
Show Appropriate UI
```

Splash/loading UI gösteriliyorsa bu UI presentation responsibility'sidir.

Gerçek initialization orchestration yine Bootstrap tarafından yönetilmelidir.

Detaylar:

`UI/SplashAndLoading.md`

---

# 18. Bootstrap and Gameplay

Gameplay sistemleri bootstrap tarafından doğrudan yönetilmemelidir.

Tercih edilen flow:

```text
Bootstrap
    ↓
Core Systems Ready
    ↓
GameFlow
    ↓
Gameplay State
    ↓
LevelSystem
    ↓
GameplayModule
```

Bu sayede bootstrap gameplay implementation detaylarını bilmez.

---

# 19. Duplicate Initialization

Aşağıdaki patternlerden kaçın:

```text
Bootstrap initializes EconomySystem
+
EconomySystem initializes itself again

Bootstrap loads Save
+
EconomySystem loads Save again

Bootstrap creates GameFlow
+
Scene creates another GameFlow
```

Bir responsibility için tek initialization owner belirle.

---

# 20. Bootstrap Checklist

Bootstrap tamamlanmış kabul edilmeden önce:

* [ ] Core systems initialize edildi.
* [ ] Required configuration hazır.
* [ ] Save data yüklendi veya güvenli default oluşturuldu.
* [ ] Save migration gerekiyorsa tamamlandı.
* [ ] Runtime systems gerekli dependency'lerini aldı.
* [ ] Persistent systems duplicate olmuyor.
* [ ] Async initialization lifecycle kontrollü.
* [ ] Initialization failure behavior tanımlı.
* [ ] GameFlow başlatılabilir durumda.
* [ ] Gameplay bootstrap'e gereksiz şekilde bağımlı değil.
* [ ] UI initialization ile bootstrap responsibility birbirine karışmıyor.
* [ ] Bootstrap birden fazla kez çalışmıyor.

---

# Final Principle

Bootstrap'in görevi oyunu oynamak değildir.

Bootstrap'in görevi:

```text
Prepare
   ↓
Initialize
   ↓
Validate
   ↓
Ready
   ↓
Hand Over to GameFlow
```

noktasında bitmelidir.

**Bootstrap ne kadar az gameplay bilgisine sahip olursa, template o kadar reusable olur.**
