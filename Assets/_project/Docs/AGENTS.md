# AGENTS.md

Bu dosya, projede çalışan AI coding agent'ları için temel geliştirme kurallarını tanımlar.

Bu proje, farklı türlerdeki **Unity hybrid-casual oyunlar** için kullanılabilecek genel ve yeniden kullanılabilir bir template mimarisidir.

Bu dosyadaki kurallar, proje içerisindeki diğer teknik dokümanlarla birlikte uygulanmalıdır.

---

# 1. Documentation Rules

Projedeki Markdown dokümanları mimarinin bir parçasıdır.

Kod yazmadan veya mevcut kodu değiştirmeden önce, yapılan iş ile ilgili dokümanları bul ve incele.

## Documentation Hierarchy

```text
AGENTS.md
    ↓
Genel geliştirme ve mimari kurallar

ARCHITECTURE.md
    ↓
Projenin genel mimarisi ve sistemler arası ilişkiler

SYSTEMS/*.md
    ↓
Core / infrastructure sistemlerinin sorumlulukları ve kontratları

GAMEPLAY/*.md
    ↓
Gameplay sistemlerinin ve modüllerinin kuralları

UI/*.md
    ↓
Reusable UI modüllerinin davranışları ve bağımlılıkları
```

## Documentation Selection

Her task için tüm dokümanları okuma.

Yalnızca task ile ilgili dokümanları bul ve incele.

Örnek:

```text
Shop task
→ AGENTS.md
→ ARCHITECTURE.md
→ SYSTEMS/Economy.md
→ SYSTEMS/Monetization.md
→ UI/ShopView.md
```

```text
Level task
→ AGENTS.md
→ ARCHITECTURE.md
→ GAMEPLAY/LevelSystem.md
→ GAMEPLAY/GameplayModule.md
```

```text
Settings UI task
→ AGENTS.md
→ UI/SettingsPopup.md
```

## Documentation vs Code

Dokümantasyon ile mevcut kod arasında farklılık varsa:

1. Farkı tespit et.
2. Mevcut implementasyonu incele.
3. Task'ın gerçek ihtiyacını değerlendir.
4. Mevcut mimariyi neden değiştirdiğini açıkça gerekçelendir.
5. Gerekiyorsa ilgili dokümantasyonu güncelle.

Dokümantasyonu görmezden gelerek yeni bir mimari oluşturma.

Dokümantasyon eskiyse, yalnızca gerekli kısmı güncelle.

---

# 2. General Principles

## Correctness First

Öncelik sırası:

```text
Correctness
→ Existing Architecture
→ Clear Responsibilities
→ Serialization Safety
→ Data-Driven Design
→ Decoupled Communication
→ Performance
→ Simplicity
```

Kod yalnızca çalışıyor olmakla yeterli değildir. Doğru sistem içerisinde, doğru sorumlulukta çalışmalıdır.

## Keep Systems Decoupled

Sistemler gereksiz şekilde birbirine bağlanmamalıdır.

Bir sistem başka bir sistemin internal implementation detaylarına bağımlı olmamalıdır.

Decoupling gerektiğinde:

* Event system
* Public API
* Explicit dependency
* Command / data flow

kullanılabilir.

Ancak her iletişimi EventBus üzerinden geçirmek zorunlu değildir.

Local ownership ve lifecycle ilişkisi açık olan durumlarda direct reference kullanılabilir.

## Single Responsibility

Bir class veya system mümkün olduğunca tek bir ana sorumluluğa sahip olmalıdır.

Örneğin:

```text
GameManager
→ Global game flow

LevelManager
→ Level state / level lifecycle

EconomySystem
→ Currency state and transactions

ScoreSystem
→ Score state

ShopView
→ Shop presentation
```

Bir sistem başka sistemlerin sorumluluklarını üstlenmemelidir.

## Avoid Over-Engineering

Gereksiz abstraction oluşturma.

İhtiyaç yoksa:

* Interface
* Factory
* Service locator
* Dependency container
* Wrapper
* Manager
* Event
* Generic framework

ekleme.

Mevcut probleme en küçük ve en anlaşılır çözümü uygula.

---

# 3. Architecture Rules

## Global Game Flow

Global oyun akışı bir FSM/state-based yapı üzerinden yönetilmelidir.

Temel state'ler:

```text
Boot
MainMenu
Gameplay
Pause
Win
Lose
```

Gerektiğinde yeni state eklenebilir.

`GameManager` yalnızca global flow'u yönetmelidir.

`GameManager`:

* Gameplay mechanics yönetmez.
* UI state yönetmez.
* Economy yönetmez.
* Level rules yönetmez.
* Reward calculation yapmaz.

## Runtime State Ownership

Her runtime state'in tek bir authoritative owner'ı olmalıdır.

Örnek:

```text
Score
→ ScoreSystem

Currency
→ EconomySystem

Current Level
→ LevelManager

Board State
→ BoardController
```

Aynı runtime state birden fazla sistem tarafından bağımsız şekilde tutulmamalıdır.

UI runtime state'in sahibi değildir.

## Data-Driven Design

Designer tarafından değiştirilebilecek değerler mümkün olduğunca ScriptableObject veya uygun serialized configuration üzerinden tanımlanmalıdır.

Örneğin:

```text
LevelConfig
ShopConfig
EconomyConfig
ProgressionConfig
AudioConfig
GameplayConfig
```

Hard-coded designer values oluşturma.

## ScriptableObject Runtime State Değildir

ScriptableObject'ler temel olarak configuration/static data için kullanılmalıdır.

Session sırasında değişen runtime state'i ScriptableObject üzerinde tutma.

Aşağıdaki ayrımı koru:

```text
Configuration
→ ScriptableObject

Runtime State
→ Runtime class / system / controller

Persistent Save Data
→ Save data model
```

## Event System

EventBus / EventManager sistemler arası decoupled communication için kullanılabilir.

Event kullanırken:

* Event'in sahibi belli olmalı.
* Event payload gereksiz veri taşımamalı.
* Subscriber lifecycle doğru yönetilmeli.
* `OnEnable` içinde subscribe ediliyorsa `OnDisable` içinde unsubscribe edilmelidir.

EventBus, direct reference'ın yerine otomatik olarak kullanılmamalıdır.

## Gameplay / Presentation Separation

Gameplay sistemi `WHAT` olduğunu belirler.

Presentation sistemi `HOW` gösterileceğini belirler.

Örneğin:

```text
Gameplay
→ Player won

UI
→ Win popup göster

Audio
→ Win sound çal

VFX
→ Win effect göster
```

UI gameplay sonucunu hesaplamamalıdır.

UI yalnızca gameplay/system state'i sunmalıdır.

## Input Architecture

Input ile gameplay logic birbirinden ayrılmalıdır.

Tercih edilen flow:

```text
Input
→ Input Controller
→ Gameplay Command
→ Gameplay System
```

Aynı gameplay logic mümkün olduğunca:

* Touch
* Mouse
* Keyboard
* Gamepad

gibi farklı input kaynaklarından kullanılabilir olmalıdır.

## Dependency Management

Dependency'leri mümkün olduğunca explicit tut.

Bir dependency için scene-wide search kullanma.

Özellikle gameplay veya initialization için:

```csharp
GameObject.Find(...)
FindObjectOfType(...)
FindFirstObjectByType(...)
FindAnyObjectByType(...)
```

kullanma.

Dependency'leri:

* Serialized reference
* Constructor / initialization parameter
* Explicit registration
* Controlled dependency injection

gibi uygun yöntemlerle yönet.

---

# 4. Ownership

## System Ownership

Her sistem kendi domain'inin sahibi olmalıdır.

Örneğin:

```text
EconomySystem
→ Currency
→ Transactions

ProgressionSystem
→ Player progression

MonetizationSystem
→ Ads / IAP integration

AnalyticsSystem
→ Analytics SDK communication

AudioSystem
→ Audio state / playback coordination

SaveSystem
→ Persistence
```

Bir sistem başka bir sistemin state'ini doğrudan değiştirmemelidir.

Mümkün olduğunda public API veya command üzerinden işlem yapılmalıdır.

## UI Ownership

UI:

* Gameplay state'in sahibi değildir.
* Economy state'in sahibi değildir.
* Reward calculation yapmaz.
* Price calculation yapmaz.
* Save data değiştirmez.

UI gerekli bilgiyi ilgili sistemlerden alır ve presentation yapar.

## Economy Ownership

Currency, transaction ve reward state'leri EconomySystem tarafından yönetilmelidir.

UI veya gameplay component'leri currency değerini kendi içinde kopyalayarak authoritative state oluşturmamalıdır.

## Monetization Ownership

Ad ve IAP SDK entegrasyonları MonetizationSystem içerisinde izole edilmelidir.

UI yalnızca monetization işlemini başlatabilir veya sonucunu gösterebilir.

SDK çağrılarını UI component'lerine dağıtma.

## Analytics Ownership

Analytics SDK çağrıları AnalyticsSystem üzerinden yapılmalıdır.

Gameplay sistemlerinin doğrudan analytics SDK implementation detaylarına bağımlı olmasını engelle.

## Save Ownership

Persistent player data SaveSystem tarafından yönetilmelidir.

Save model ile runtime state'i birbirine karıştırma.

Önerilen ayrım:

```text
PlayerConfig
→ Static configuration

PlayerRuntimeState
→ Current session state

PlayerSaveData
→ Persistent player data
```

---

# 5. Performance

Bu proje mobile platformları hedefleyebilen hybrid-casual oyunlar için tasarlanmıştır.

Performans kritik kodlarda allocation ve gereksiz Unity API çağrılarından kaçın.

## Hot Path

Özellikle aşağıdaki alanlarda allocation oluşturma:

```text
Update
FixedUpdate
LateUpdate
Input
Collision
Board logic
AI
Spawn
Frequent UI updates
```

Kaçınılması gerekenler:

* LINQ
* Closure
* Boxing
* Temporary collections
* Temporary arrays
* Repeated delegate allocation
* String concatenation
* Reflection
* Per-frame object creation

Performans kritik alanlarda allocation gerekiyorsa gerekçesini değerlendir.

## Object Pooling

Sık oluşturulup yok edilen runtime objeleri için pooling kullan.

Örnek:

```text
VFX
Particles
Projectiles
Floating Numbers
Tiles
Enemies
Collectibles
Temporary UI
```

Gameplay hot path içerisinde gereksiz:

```csharp
Instantiate(...)
Destroy(...)
```

kullanma.

One-time initialization sırasında Instantiate kullanımı gerektiğinde kabul edilebilir.

## Pool Reset

Pool'dan çıkan object tamamen temiz bir runtime state ile başlamalıdır.

Reset edilmesi gerekenler ihtiyaca göre:

```text
Position
Rotation
Scale
Animation
Physics
Timers
Flags
Runtime Data
Visual State
Event Subscriptions
```

Pool'a dönen object eski session state'ini taşımamalıdır.

## Component Access

Sık kullanılan component reference'larını cache et.

Hot path içerisinde tekrar tekrar:

```csharp
GetComponent<T>()
```

çağırma.

---

# 6. Unity Rules

## Unity Lifecycle

Genel lifecycle yaklaşımı:

```text
Awake
→ Local initialization / reference caching

OnEnable
→ Subscribe

Start
→ Initialization requiring other Awake calls

OnDisable
→ Unsubscribe

OnDestroy
→ Final cleanup when required
```

Unity lifecycle bağımlılıklarını açıkça yönet.

## Coroutine / Async

Projede kullanılan async yaklaşımını tutarlı şekilde kullan.

Gereksiz şekilde:

```text
Coroutine
Task
async/await
UniTask
```

arasında geçiş yapma.

Uzun yaşayan async operasyonların ownership ve cancellation lifecycle'ı olmalıdır.

Unmanaged infinite coroutine oluşturma.

## Tween / Animation

Projede belirlenmiş tween sistemini kullan.

Tween'ler:

* Disable sırasında temizlenmeli.
* Destroy sırasında temizlenmeli.
* Pool'a dönerken resetlenmeli.

Gameplay state yalnızca animation/tween completion'a bağımlı olmamalıdır.

Animation presentation'dır.

## Serialization Safety

Unity serialization sistemine dikkat et.

Özellikle değişikliklerde:

* Serialized field name
* Serialized field type
* Enum values
* ScriptableObject structure
* Prefab references
* Scene references

dikkatle korunmalıdır.

Serialized field rename gerekiyorsa uygun durumlarda:

```csharp
[FormerlySerializedAs("oldName")]
```

kullan.

## ScriptableObjects

ScriptableObject'leri configuration/static data için kullan.

Runtime session state'i ScriptableObject içine koyma.

## Addressables / Resources

Projede mevcut asset loading yaklaşımını koru.

Yeni bir loading sistemi oluşturmak için mevcut sistemi gereksiz yere değiştirme.

`Resources` kullanımını yalnızca mevcut architecture bunu gerektiriyorsa kullan.

Addressables kullanılan bir projede asset lifecycle ve release kurallarına uy.

## Audio

Audio logic'i gameplay logic'ten ayır.

Gameplay:

```text
"Player won"
```

Audio:

```text
"Win sound play"
```

AudioSystem ses asset'leri, playback ve audio state'in sorumluluğunu taşımalıdır.

## VFX

VFX presentation layer'dır.

VFX gameplay state'in sahibi olmamalıdır.

Sık kullanılan VFX'ler mümkün olduğunda pool edilmelidir.

---

# 7. Coding Rules

## Naming

Mevcut proje naming convention'larını koru.

Unity/C# standart naming:

```text
Class
→ PascalCase

Method
→ PascalCase

Public Property
→ PascalCase

Private Field
→ camelCase veya mevcut project convention

Constant
→ PascalCase veya mevcut project convention
```

Yeni naming convention oluşturma.

## Null Handling

Null durumlarını bilinçli şekilde ele al.

Null check eklemek yalnızca warning'i susturmak için yapılmamalıdır.

Bir reference'ın neden null olabileceğini anlamaya çalış.

Required dependency için fail-fast davranış uygun olabilir.

Optional dependency için kontrollü fallback kullanılabilir.

## Error Handling

Hataları sessizce yutma.

```csharp
catch
{
}
```

gibi boş error handling kullanma.

Error handling sistemin gerçek failure behavior'ına uygun olmalıdır.

## Testing

Pure gameplay logic mümkün olduğunca scene'den bağımsız test edilebilir olmalıdır.

Tercih:

```text
EditMode
→ Deterministic / pure logic

PlayMode
→ Unity integration / lifecycle / scene behavior
```

Her küçük değişiklik için gereksiz test framework abstraction'ı oluşturma.

## Code Quality

Yeni kod:

* Açık
* Küçük
* Test edilebilir
* Maintainable
* Mevcut architecture ile uyumlu

olmalıdır.

Kod miktarını artırmak başarı kriteri değildir.

---

# 8. Modification Safety

Mevcut projede değişiklik yaparken mevcut sistemi mümkün olduğunca koru.

## Inspect Before Modify

Kod yazmadan önce:

1. İlgili scriptleri bul.
2. Benzer implementasyonları incele.
3. Dependency'leri kontrol et.
4. Runtime state ownership'i belirle.
5. İlgili documentation'ı oku.
6. En küçük değişiklik alanını belirle.

## Do Not Rewrite Unnecessarily

Çalışan bir sistemi sırf daha güzel göründüğü için yeniden yazma.

Task için gerekli olmayan refactor yapma.

Unrelated code'a dokunma.

## Preserve Existing Contracts

Aşağıdaki contract'ları gereksiz yere bozma:

```text
Public APIs
Serialized fields
Prefab references
Scene references
ScriptableObject data
Save data
Event contracts
Existing system responsibilities
```

Bir contract değişikliği gerekiyorsa etkisini değerlendir.

## Serialization Changes

Unity serialization değişikliklerinde backward compatibility'yi düşün.

Özellikle mevcut prefab, scene veya ScriptableObject asset'lerini bozabilecek değişikliklerden kaçın.

## Smallest Surface Area

Bir feature için mümkün olan en küçük kod ve asset değişikliğini yap.

Büyük değişiklik gerekiyorsa nedenini açıkça ortaya koy.

---

# 9. Definition of Done

Bir task tamamlanmadan önce aşağıdaki maddeleri kontrol et:

* [ ] Doğru system / class üzerinde değişiklik yapıldı.
* [ ] Existing architecture korundu.
* [ ] İlgili documentation incelendi.
* [ ] Gerekliyse documentation güncellendi.
* [ ] Runtime state'in authoritative owner'ı net.
* [ ] UI gameplay state'in sahibi değil.
* [ ] Designer-tunable values data-driven.
* [ ] ScriptableObject runtime state olarak kullanılmadı.
* [ ] Event subscription lifecycle doğru.
* [ ] Pool kullanılan object'ler tamamen resetleniyor.
* [ ] Hot path içerisinde gereksiz allocation yok.
* [ ] Hot path içerisinde gereksiz Instantiate / Destroy yok.
* [ ] Gereksiz Find / GetComponent çağrıları yok.
* [ ] Serialization güvenliği kontrol edildi.
* [ ] Async / Coroutine lifecycle kontrol edildi.
* [ ] Tween lifecycle kontrol edildi.
* [ ] Mobile performance dikkate alındı.
* [ ] Uygun yerlerde test edildi.
* [ ] Unrelated code değiştirilmedi.
* [ ] Gereksiz abstraction eklenmedi.

---

# Final Principle

Kod yazarken şu soruları sırayla sor:

```text
1. Bu kodun sorumluluğu ne?
2. Bu state'in sahibi kim?
3. Bu iş için mevcut bir system veya module var mı?
4. İlgili documentation ne diyor?
5. Bu dependency gerçekten gerekli mi?
6. Decoupling gerekli mi?
7. Bu kod hot path'te çalışıyor mu?
8. Allocation / Instantiate / Destroy oluşturuyor mu?
9. Serialization veya mevcut asset'leri etkiler mi?
10. Daha basit bir çözüm var mı?
```

**Mevcut mimariyi anlamadan yeni mimari oluşturma.**

**Mevcut bir sorumluluğu başka bir system'e taşıma.**

**Gereksiz abstraction ekleme.**

**Gameplay state ile presentation state'i birbirine karıştırma.**

**Önce doğru ownership'i belirle, sonra kodu yaz.**
