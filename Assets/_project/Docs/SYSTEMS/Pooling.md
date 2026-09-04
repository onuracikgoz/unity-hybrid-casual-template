# Pooling System

## 1. Amaç

`PoolingSystem`, sık oluşturulup yok edilen runtime object'lerinin tekrar kullanılmasını sağlar.

Amaç:

* Runtime allocation'ı azaltmak
* `Instantiate` / `Destroy` maliyetini azaltmak
* Garbage Collection baskısını azaltmak
* Gameplay hot path'lerini daha stabil hale getirmek
* Runtime object lifecycle'ını standartlaştırmak
* Pooled object'lerin temiz ve tekrar kullanılabilir olmasını garanti etmek

Pooling bir zorunluluk değildir.

Bir object'in kullanım pattern'i pooling gerektirmiyorsa normal lifecycle kullanılabilir.

---

# 2. Pooling Ne Çözer?

Normal lifecycle:

```text
Spawn
  ↓
Instantiate
  ↓
Use
  ↓
Destroy
```

Pooling:

```text
Create Pool
    ↓
Get
    ↓
Use
    ↓
Release
    ↓
Reset
    ↓
Available
    ↓
Get Again
```

Temel amaç object'i her seferinde yeniden yaratmak yerine tekrar kullanmaktır.

---

# 3. Hangi Object'ler Pool İçin Uygundur?

Sık oluşturulan ve kısa ömürlü object'ler pooling için güçlü adaylardır.

Örneğin:

```text
VFX
Particles
Projectile
Floating Damage Text
Collectible
Enemy
Temporary Gameplay Object
Tile
Board Object
Temporary UI
```

Ancak yalnızca:

```text
"Bu object runtime'da oluşturuluyor."
```

diye pooling zorunlu değildir.

---

# 4. Pooling İçin Uygun Olmayan Object'ler

Aşağıdaki object'lerde pooling gereksiz olabilir:

```text
Bootstrap Object
Main Camera
Persistent System
Static Scene Object
One-time Initialization Object
Rarely Created Object
```

Örneğin splash screen yalnızca bir kez oluşturuluyorsa pool yapmak gereksizdir.

---

# 5. Pool Ownership

Pool'un sahibi açık olmalıdır.

Örneğin:

```text id="r4m8qa"
EnemySystem
    ↓
EnemyPool
```

veya:

```text id="v7c2mx"
VFXSystem
    ↓
VFXPool
```

olabilir.

Global pool manager'a her şeyi koymak zorunlu değildir.

---

# 6. Pool Manager

Projede ortak bir `PoolManager` bulunabilir.

Örneğin:

```text id="k8n3za"
PoolManager
├── Enemy Pool
├── Projectile Pool
├── VFX Pool
└── FloatingText Pool
```

Bu yapı yalnızca gerçekten ortak pooling orchestration ihtiyacı varsa kullanılmalıdır.

Her pool için global manager dependency'si oluşturmak zorunlu değildir.

---

# 7. Pool ve Asset Ayrımı

Pooling asset loading sisteminin yerine geçmez.

Asset Management:

```text id="m5q9xc"
Asset
    ↓
Load
    ↓
Prefab Reference
```

Pooling:

```text id="p7r2va"
Prefab
    ↓
Pool
    ↓
Runtime Instances
```

Yani:

```text
AssetManagement
    → Asset nereden geliyor?

Pooling
    → Runtime instance nasıl tekrar kullanılıyor?
```

---

# 8. Pool ve Addressables

Addressables ile yüklenen prefab'lar pool'a alınabilir.

Örneğin:

```text id="x3m8qa"
Addressables
    ↓
Load Prefab
    ↓
Create Pool
    ↓
Get / Release To Pool
```

Ancak Addressables reference lifetime ile pool lifetime birbirine karıştırılmamalıdır.

Pool'da yaşayan object'in bağlı olduğu asset, pool kullanılmaya devam ettiği sürece erişilebilir olmalıdır.

---

# 9. Pool Lifetime

Pool'un lifecycle'ı açık olmalıdır.

Örneğin:

```text id="q6r4mc"
Bootstrap
    ↓
Create Pool
    ↓
Gameplay
    ↓
Reuse
    ↓
Gameplay End
    ↓
Clear Pool
```

veya pool global persistent ise:

```text id="n8v2za"
Bootstrap
    ↓
Create Pool
    ↓
Persistent
```

kullanılabilir.

Hangi lifecycle'ın kullanılacağı object'in ownership'ine bağlıdır.

---

# 10. Prewarm

Prewarm, pool'un başlangıçta belirli sayıda object oluşturmasıdır.

Örneğin:

```text id="m4c7xp"
Enemy Pool
    ↓
Prewarm 20
```

Bu sayede ilk gameplay sırasında Instantiate spike azaltılabilir.

Prewarm değeri configuration'dan gelebilir.

---

# 11. Prewarm Her Zaman Gerekli Değildir

Küçük veya nadiren kullanılan pool'lar için prewarm gereksiz olabilir.

Örneğin:

```text id="v5q8ma"
Rare Boss VFX
```

için 50 object prewarm etmek gereksiz memory tüketebilir.

Prewarm:

```text
Expected Peak Usage
```

ve:

```text
Memory Budget
```

ile birlikte değerlendirilmelidir.

---

# 12. Dynamic Pool Growth

Pool ihtiyaç olduğunda büyüyebilir.

Örneğin:

```text id="r7n3vc"
Pool Size = 20

Request #21
    ↓
Create Additional Instance
```

Bu durumda growth strategy açık olmalıdır.

Örneğin:

```text
Fixed Capacity
Dynamic Growth
Dynamic Growth + Maximum Capacity
```

seçilebilir.

---

# 13. Maximum Capacity

Pool'un maksimum kapasitesi gerekiyorsa tanımlanabilir.

Örneğin:

```text id="k4m9xa"
Pool Capacity
    = 50
```

50 object aktifken yeni request geldiğinde sistemin davranışı belirli olmalıdır.

Olası davranışlar:

```text
Reject
Queue
Fallback Instantiate
Steal Oldest
```

Gameplay'e göre karar verilmelidir.

---

# 14. Pool Request Failure

Pool'da object bulunamadığında sessizce null dönmek her durumda doğru değildir.

Örneğin:

```text id="p8x3mv"
Get Projectile
    ↓
Pool Empty
```

davranışı açık olmalıdır.

Kritik gameplay object'lerinde failure kontrollü şekilde ele alınmalıdır.

Development build'de açık warning/error faydalı olabilir.

---

# 15. Get / Release Contract

Temel API mümkün olduğunca basit tutulmalıdır.

Örneğin:

```csharp id="c5r8za"
var enemy = enemyPool.Get();
enemyPool.Release(enemy);
```

veya:

```csharp id="m7q2vx"
var enemy = pool.Get<Enemy>();
pool.Release(enemy);
```

gibi bir API yeterli olabilir.

Pool API gereksiz generic abstraction'larla büyütülmemelidir.

---

# 16. Pooled Object Lifecycle

Pooled object için lifecycle:

```text id="x4n9mc"
Get
 ↓
Reset
 ↓
Initialize
 ↓
Active
 ↓
Release
 ↓
Cleanup
 ↓
Inactive
```

şeklinde düşünülebilir.

Reset ve initialization sorumlulukları net olmalıdır.

---

# 17. Reset Contract

Pool'dan çıkan object kullanılmadan önce geçerli bir başlangıç state'ine sahip olmalıdır.

Örneğin enemy:

```text id="q8m3va"
Health
Target
Velocity
Status Effects
Animation State
Timers
Flags
```

resetlenmelidir.

---

# 18. Reset Eksikliği

En tehlikeli pooling bug'larından biri state leakage'dir.

Örneğin:

```text id="r6c2xp"
Enemy A
Health = 10
IsBurning = true
Target = Player
```

pool'a döner.

Daha sonra:

```text id="v9m4za"
Enemy B
```

olarak kullanılır.

Eğer reset yapılmadıysa:

```text
IsBurning = true
Target = Player
```

gibi eski state yeni instance'a sızabilir.

---

# 19. Reset Edilmesi Gereken State

Pooled object'in sahip olduğu runtime state gözden geçirilmelidir.

Kontrol listesi:

```text id="k7p3xc"
Transform
Position
Rotation
Scale

Velocity
Acceleration

Health
Damage
Target

Timers
Cooldowns

Animation
Animator Parameters

Particles
Trail Renderer

Visual State
Material State

Flags
Booleans

Temporary Data
References

Event Subscriptions
Callbacks
```

Her object için gereken alanlar farklıdır.

---

# 20. Transform Reset

Pool'a dönen object'in transform state'i kontrol edilmelidir.

Özellikle:

```text
Position
Rotation
Scale
Parent
```

değerleri tekrar kullanımdan önce doğru hale getirilmelidir.

Yanlış parent altında kalmış pooled object UI veya gameplay hierarchy'sinde beklenmedik sonuçlar oluşturabilir.

---

# 21. Parent Management

Pooled object'ler gerektiğinde belirlenmiş pool root altında tutulabilir.

Örneğin:

```text id="m4x8qa"
GameplayRoot
├── Active Enemies
└── EnemyPool
```

Release:

```text
Enemy
    ↓
EnemyPool Root
```

Get:

```text
EnemyPool
    ↓
Gameplay Parent
```

şeklinde çalışabilir.

Parent lifecycle açık olmalıdır.

---

# 22. Active State

Release edilen object:

```csharp id="n6r2vc"
gameObject.SetActive(false);
```

ile inactive hale getirilebilir.

Get sırasında:

```csharp id="p8m4za"
gameObject.SetActive(true);
```

yapılabilir.

Ancak `OnEnable` / `OnDisable` lifecycle'larının pool davranışıyla uyumlu olması gerekir.

---

# 23. OnEnable / OnDisable

Pooled object'ler için:

```text id="q5c7xm"
OnEnable
    ↓
Subscribe / Activate

OnDisable
    ↓
Unsubscribe / Deactivate
```

pattern'i kullanılabilir.

Ancak `OnEnable` her pool spawn'ında çalışabileceği için pahalı initialization burada yapılmamalıdır.

---

# 24. Awake

`Awake` genellikle:

```text id="v8r3qa"
Cache Component References
Create Immutable Local Data
Initial Setup
```

için kullanılabilir.

Örneğin:

```csharp id="m4n7xc"
private void Awake()
{
    _renderer = GetComponent<Renderer>();
    _animator = GetComponent<Animator>();
}
```

Her pool spawn'ında `GetComponent` çağırmak yerine bir kez cache etmek tercih edilir.

---

# 25. Initialize

Runtime-specific data `Initialize` sırasında verilebilir.

Örneğin:

```csharp id="x7q2ma"
enemy.Initialize(enemyData, target);
```

Burada:

```text
enemyData
target
spawnPosition
runtime parameters
```

gibi session-specific data sağlanabilir.

---

# 26. Initialize ve Reset Ayrımı

`Reset`:

```text
Eski state'i temizle
```

`Initialize`:

```text
Yeni kullanım için state'i kur
```

anlamına gelmelidir.

Örneğin:

```text id="c9m4vp"
Release
 ↓
Reset
 ↓
Pool
 ↓
Get
 ↓
Initialize
```

---

# 27. Release

Release işlemi object'in pool'a geri dönmesini sağlar.

Örneğin:

```text id="r3x8qa"
Release
 ↓
Stop Active Behavior
 ↓
Unsubscribe
 ↓
Reset
 ↓
Deactivate
 ↓
Return To Pool
```

Object release edildikten sonra gameplay logic'in onu kullanmaya devam etmemesi gerekir.

---

# 28. Double Release

Aynı object'in pool'a iki kez release edilmesi engellenmelidir.

Risk:

```text id="m6q2xc"
Release
 ↓
Pool

Release Again
 ↓
Same Object Already In Pool
```

Bu durum:

* Duplicate pool entries
* Invalid state
* Same instance'in iki consumer'a verilmesi

gibi ciddi bug'lara yol açabilir.

Development build'de detection faydalıdır.

---

# 29. Use After Release

Release edilmiş object'e gameplay logic'in referans üzerinden erişmeye devam etmesi tehlikelidir.

Örneğin:

```text id="v4n8za"
enemyPool.Release(enemy);

enemy.TakeDamage(10);
```

gibi bir kullanım invalid lifecycle oluşturur.

Release sonrasında owner artık object'i kullanmamalıdır.

---

# 30. Pool ve References

Başka system'ler pooled object reference'larını tutuyorsa release lifecycle dikkate alınmalıdır.

Örneğin:

```text id="q7m3xc"
TargetSystem
    ↓
Enemy Reference
```

Enemy pool'a döndüğünde `TargetSystem` bu reference'ı temizlemelidir.

Aksi halde inactive veya başka entity olarak yeniden kullanılan object'e referans kalabilir.

---

# 31. Event Subscription

Pooled object EventBus event'lerini dinliyorsa:

```text id="n8c4va"
Get
 ↓
Subscribe
 ↓
Use
 ↓
Release
 ↓
Unsubscribe
```

lifecycle'ı garanti edilmelidir.

En güvenli pattern:

```csharp id="p5r7xm"
private void OnEnable()
{
    Subscribe();
}

private void OnDisable()
{
    Unsubscribe();
}
```

Ancak event source static/global ise özellikle dikkat edilmelidir.

---

# 32. Coroutine

Pooled object üzerinde çalışan coroutine'ler release sırasında durdurulmalıdır.

Örneğin:

```text id="x3m8qc"
Enemy
 ↓
Coroutine
 ↓
Release
```

coroutine yaşamaya devam ederse object pool'a döndükten sonra state değiştirebilir.

Release sırasında ilgili async/timing operation lifecycle'ı güvenli şekilde kapatılmalıdır.

---

# 33. Async Operation

Pooled object'in async operation'ı varsa aynı kural geçerlidir.

Örneğin:

```text id="r9q4va"
Load / Delay / Tween / Async
        ↓
Object Released
```

operation object'in artık aktif olmadığını dikkate almalıdır.

Operation sonucu object'e uygulanmadan önce lifecycle kontrolü yapılmalıdır.

---

# 34. Tween Cleanup

Pooled object üzerinde tween çalışıyorsa release sırasında tween temizlenmelidir.

Örneğin:

```text id="m7c2xp"
Release
 ↓
Kill / Complete as appropriate
 ↓
Reset Transform
 ↓
Pool
```

Yeni kullanımın eski tween tarafından etkilenmesine izin verilmemelidir.

---

# 35. Particle System

Particle object'leri pool için yaygın adaydır.

Release sırasında:

```text id="k5n8za"
Stop
 ↓
Clear
 ↓
Reset
 ↓
Pool
```

uygulanmalıdır.

Yeni kullanımda eski particle state'i kalmamalıdır.

---

# 36. Trail Renderer

Trail kullanan pooled object'lerde trail state temizlenmelidir.

Örneğin:

```text id="v3q7mc"
Release
 ↓
Clear Trail
 ↓
Deactivate
```

Aksi halde object yeniden spawn olduğunda eski hareket izi tekrar görünebilir.

---

# 37. Animator

Animator state pool reuse sırasında resetlenmelidir.

Örneğin:

```text id="p8m4xa"
Reset Animation State
Reset Parameters
```

gerekiyorsa uygulanmalıdır.

Bir enemy'nin önceki kullanımındaki:

```text
Death
```

state'i ile tekrar spawn olması engellenmelidir.

---

# 38. Physics State

Rigidbody kullanan pooled object'lerde:

```text id="q6r2vc"
Velocity
Angular Velocity
Physics State
```

resetlenmelidir.

Yeni spawn position'ı ayarlandıktan sonra eski velocity'nin object'i hareket ettirmesine izin verilmemelidir.

---

# 39. Pool ve Gameplay State

Pool yalnızca runtime object lifecycle'ını yönetir.

Gameplay state'in owner'ı değildir.

Örneğin:

```text id="m4x8qa"
EnemyPool
    → Enemy Instances

CombatSystem
    → Combat State
```

Pool:

```text
"Bu enemy instance hazır mı?"
```

sorusunu yönetir.

CombatSystem:

```text
"Enemy ne durumda?"
```

sorusunun sahibidir.

---

# 40. Pool ve SaveSystem

Pooled object'ler doğrudan save data'ya yazılmamalıdır.

Örneğin:

```text id="r7c3xm"
Enemy Pool
    ↓
SaveSystem
```

doğrudan dependency olmamalıdır.

Gameplay'de persistence gerekiyorsa gameplay state → SaveSystem üzerinden modellenmelidir.

---

# 41. Pool ve Scene Transition

Scene-specific pool'lar scene ile birlikte temizlenmelidir.

Örneğin:

```text id="x8m2qa"
Gameplay Scene
    ↓
Gameplay Pool
    ↓
Scene Exit
    ↓
Clear Pool
```

Persistent pool'lar ise yalnızca gerçekten shared lifecycle ihtiyacı varsa scene'ler arasında yaşayabilir.

---

# 42. Pool ve Addressable Release

Addressable prefab pool tarafından kullanılıyorsa:

```text id="q4n9vc"
Load Asset
 ↓
Create Pool
 ↓
Pool Active
 ↓
Clear Pool
 ↓
Release Asset
```

sırası korunmalıdır.

Pool object'leri hala asset'e referans verirken Addressable handle release edilmemelidir.

---

# 43. Pool Prewarm Configuration

Prewarm ve capacity değerleri designer-tunable ise configuration kullanılabilir.

Örneğin:

```text id="m6r3xa"
PoolConfig
├── InitialSize
├── MaxSize
└── AllowGrowth
```

Bu değerler ScriptableObject olabilir.

Runtime pool state ScriptableObject üzerinde tutulmamalıdır.

---

# 44. Pool Metrics

Development/debug için pool metrics faydalı olabilir.

Örneğin:

```text id="v8c4mp"
Total Created
Active Count
Inactive Count
Peak Active
Get Count
Release Count
Failed Get Count
```

Bu bilgiler production gameplay logic'in source of truth'u değildir.

---

# 45. Performance

Pooling şu problemleri azaltabilir:

* Instantiate cost
* Destroy cost
* GC pressure
* Runtime allocation spikes

Ancak pooling her zaman performansı otomatik olarak artırmaz.

Aşırı büyük pool:

```text
Memory Usage ↑
```

oluşturabilir.

Bu nedenle:

```text
CPU
GC
Memory
Peak Usage
```

birlikte değerlendirilmelidir.

---

# 46. Pooling ve Mobile

Mobile cihazlarda:

* Memory budget
* Frame consistency
* GC spike
* Loading spike

özellikle önemlidir.

Pooling özellikle gameplay sırasında sık spawn/despawn edilen object'lerde frame-time stabilitesine yardımcı olabilir.

Ancak gereksiz prewarm ile başlangıç memory footprint'i artırılmamalıdır.

---

# 47. Hot Path

Gameplay hot path'inde:

```text id="p7m2xc"
Get
Initialize
Use
Release
```

işlemleri mümkün olduğunca allocation-free olmalıdır.

Pool implementation'ın kendisi:

* LINQ
* Temporary collections
* Reflection
* String allocation

gibi gereksiz maliyetler oluşturmamalıdır.

---

# 48. Pool API ve Genericity

Pool API mümkün olduğunca küçük olmalıdır.

Örneğin:

```text id="x5r8qa"
Get()
Release()
Clear()
Prewarm()
```

çoğu kullanım için yeterlidir.

Şunlar ancak gerçek ihtiyaç varsa eklenmelidir:

```text
GetAsync()
GetWithFactory()
GetWithDependencyGraph()
GetWithResolver()
```

Template'in pooling sistemi abstraction yarışına dönüşmemelidir.

---

# 49. Prefab Validation

Pool'a verilen prefab development sırasında validate edilebilir.

Örneğin:

```text id="m3q7vc"
Required Component
Missing Component
Invalid Configuration
```

durumları erken fark edilmelidir.

Runtime'da sürekli validation yapmak yerine initialization aşamasında kontrol etmek tercih edilir.

---

# 50. Pool Collection Type

Pool internal collection implementation'ı kullanım pattern'ine göre seçilmelidir.

Örneğin:

```text id="r8n4xa"
Stack
Queue
List
Custom Structure
```

kullanılabilir.

Her pool için aynı collection zorunlu değildir.

Önemli olan:

* Allocation
* Access pattern
* Removal cost
* Memory usage

olmalıdır.

---

# 51. Pool Sorting / Priority

Bazı gameplay object'lerinde release sırasında priority gerekebilir.

Örneğin projectile pool'da:

```text id="q6m2vc"
Oldest Active Object
```

geri alınabilir.

Ancak bu davranış gameplay semantics'ini etkiliyorsa pool bunu kendiliğinden belirlememelidir.

Object stealing gameplay owner tarafından tanımlanmalıdır.

---

# 52. Pool Failure ve Gameplay

Pool failure gameplay sonucunu etkiliyorsa karar gameplay layer'da verilmelidir.

Örneğin:

```text id="v4r8qa"
Projectile Pool Full
```

durumunda:

```text
Reject Shot
```

kararını CombatSystem verebilir.

Pool yalnızca:

```text
Request could not be fulfilled
```

bilgisini sağlar.

---

# 53. Testing

Pooling için aşağıdaki testler faydalıdır:

* Get returns valid object
* Release returns object to pool
* Released object becomes inactive
* Object can be reused
* State is reset
* Double release is handled
* Pool capacity behaves correctly
* Growth behaves correctly
* Clear removes active/inactive references
* Event subscriptions are cleaned
* Coroutine/tween state is cleaned
* Physics state is reset

Özellikle reset testleri önemlidir.

---

# 54. Pool Leak

Pool leak şu durumlarda oluşabilir:

```text id="k8m3xa"
Get
 ↓
Never Release
```

veya:

```text id="p5q7vc"
Release
 ↓
External Reference Keeps Object Alive
```

Gameplay lifecycle ile pool lifecycle'ın birlikte tasarlanması gerekir.

Development metrics peak active count takibinde faydalı olabilir.

---

# 55. Pool ve Object Destruction

Pooled object normal gameplay sırasında `Destroy` edilmemelidir.

Örneğin:

```text id="x7m4za"
Enemy Dies
    ↓
Release To Pool
```

kullanılmalıdır.

Ancak pool shutdown sırasında:

```text id="r3c8mp"
Clear Pool
    ↓
Destroy Instances
```

normal olabilir.

---

# 56. Pool Shutdown

Pool kapatılırken:

```text id="q9n2va"
Stop Active Behavior
 ↓
Unsubscribe
 ↓
Cleanup
 ↓
Destroy Instances
 ↓
Clear Collection
```

gibi bir shutdown lifecycle uygulanmalıdır.

Addressable kullanılıyorsa asset release de doğru sırada yapılmalıdır.

---

# 57. Recommended Pool Lifecycle

Genel önerilen lifecycle:

```text id="m6x3qa"
Pool Creation
      ↓
Optional Prewarm
      ↓
Get
      ↓
Reset
      ↓
Initialize
      ↓
Activate
      ↓
Use
      ↓
Release
      ↓
Deactivate
      ↓
Cleanup
      ↓
Available
```

Exact sıra object implementation'ına göre değişebilir.

---

# 58. Pooling Decision Tree

Yeni bir runtime object için:

```text id="v8r4mc"
Frequently Created?
        │
       No
        ↓
Normal Lifecycle
        │
       Yes
        ↓
Short-Lived?
        │
       No
        ↓
Evaluate Case
        │
       Yes
        ↓
Instantiate/Destroy Hot Path?
        │
       No
        ↓
Normal Lifecycle
        │
       Yes
        ↓
Use Pool
```

Sonrasında memory budget ve peak usage kontrol edilmelidir.

---

# 59. Pooling Checklist

Yeni pooled object eklenirken:

* [ ] Pool gerçekten gerekli mi?
* [ ] Pool owner belli mi?
* [ ] Object lifetime açık mı?
* [ ] Get/Release contract belli mi?
* [ ] Initial size belirlendi mi?
* [ ] Maximum size gerekiyorsa belirlendi mi?
* [ ] Growth behavior belli mi?
* [ ] Reset contract tanımlı mı?
* [ ] Transform resetleniyor mu?
* [ ] Runtime state temizleniyor mu?
* [ ] Event subscriptions temizleniyor mu?
* [ ] Coroutine/async operation temizleniyor mu?
* [ ] Tween temizleniyor mu?
* [ ] Animator state resetleniyor mu?
* [ ] Particle/trail state temizleniyor mu?
* [ ] Physics state resetleniyor mu?
* [ ] Double release kontrol edildi mi?
* [ ] Use-after-release riski kontrol edildi mi?
* [ ] External references temizleniyor mu?
* [ ] Scene transition lifecycle tanımlı mı?
* [ ] Addressable lifetime doğru mu?
* [ ] Hot path allocation kontrol edildi mi?
* [ ] Memory budget kontrol edildi mi?
* [ ] Pool failure davranışı belli mi?
* [ ] Testler mevcut mu?

---

# 60. Definition of Done

Pooling implementation tamamlanmış sayılmadan önce:

* [ ] Pool kullanım gerekçesi açık mı?
* [ ] Ownership belli mi?
* [ ] Lifecycle açık mı?
* [ ] Get/Release güvenli mi?
* [ ] Reset eksiksiz mi?
* [ ] Double release engelleniyor mu?
* [ ] Use-after-release riski kontrol edilmiş mi?
* [ ] Subscription cleanup garanti edilmiş mi?
* [ ] Coroutine/async lifecycle güvenli mi?
* [ ] Tween lifecycle güvenli mi?
* [ ] Physics/Animator/Particle state resetleniyor mu?
* [ ] Pool capacity davranışı belirli mi?
* [ ] Prewarm memory açısından uygun mu?
* [ ] Dynamic growth gerekiyorsa kontrollü mü?
* [ ] Scene lifecycle doğru mu?
* [ ] Addressable lifetime doğru yönetiliyor mu?
* [ ] Runtime `Instantiate` / `Destroy` hot path'ten kaldırılmış mı?
* [ ] Hot path allocation kontrol edilmiş mi?
* [ ] Pool gameplay state'in owner'ı haline gelmemiş mi?
* [ ] SaveSystem'e gereksiz dependency yok mu?
* [ ] Gereksiz global manager/abstraction eklenmemiş mi?
* [ ] Testler mevcut mu?
* [ ] Mobile memory/performance değerlendirilmiş mi?
