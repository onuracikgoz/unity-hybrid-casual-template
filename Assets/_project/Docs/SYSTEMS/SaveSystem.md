# Save System

## 1. Amaç

`SaveSystem`, oyuncuya ait persistence gerektiren runtime state'in güvenli şekilde kaydedilmesinden ve yüklenmesinden sorumludur.

Amaç:

* Player progress'in kalıcı olarak saklanması
* Runtime state ile persistence state'in ayrılması
* Save formatının versioning ile yönetilmesi
* Eski save formatlarının migrate edilebilmesi
* Save corruption durumlarının kontrollü ele alınması
* Local storage lifecycle'ının merkezi olarak yönetilmesi
* Gameplay/system kodunun fiziksel dosya formatına bağımlı olmamasının sağlanması
* İleride cloud save eklenebilecek temiz bir sınır oluşturulması

Temel local persistence yaklaşımı:

```text id="f7m2xa"
Runtime State
    ↓
SaveSystem
    ↓
Local Save Storage
    ↓
JSON File
    ↓
Application.persistentDataPath
```

---

# 2. Temel Prensip

SaveSystem'in temel görevi:

> Runtime state'i persistence modeline dönüştürmek ve persistence modelini tekrar runtime state'e uygulamaktır.

SaveSystem:

* Gameplay logic'in sahibi değildir
* Economy logic'in sahibi değildir
* Progression logic'in sahibi değildir
* UI state'in sahibi değildir
* ScriptableObject configuration'ın sahibi değildir

SaveSystem persistence orchestration katmanıdır.

---

# 3. Configuration, Runtime State ve Save Data

Üç kavram birbirinden ayrılmalıdır.

```text id="m8q4vc"
Configuration
    ↓
ScriptableObject
    ↓
Static / Designer Data
```

```text id="r5x2za"
Runtime State
    ↓
Memory
    ↓
Current Session
```

```text id="k7n3mp"
Save Data
    ↓
Persistent Model
    ↓
Disk / Cloud
```

Örneğin:

```text id="v4c8qa"
EconomyConfig
    → Currency definitions

EconomyRuntimeState
    → Current currency amount

PlayerSaveData
    → Currency amount that must persist
```

---

# 4. SaveSystem Source of Truth Değildir

SaveSystem state'in sahibi değildir.

Örneğin:

```text id="p6m2xa"
EconomySystem
    → Currency State
```

SaveSystem:

```text id="q8r4vc"
EconomySystem
    ↓
Save Data
```

alır.

Benzer şekilde:

```text id="x3n7za"
ProgressionSystem
    → Progression State

SaveSystem
    → Persists Progression
```

SaveSystem'in kendi içinde ikinci bir mutable currency/progression state'i bulunmamalıdır.

---

# 5. Local Storage

İlk template implementation için local storage:

```text id="c9m4xa"
Application.persistentDataPath
```

üzerinde JSON dosyası kullanılabilir.

Örneğin:

```text id="r7q2vc"
persistentDataPath/
└── player_save.json
```

Exact filename project configuration ile belirlenebilir.

Kodun farklı yerlerine path string'i dağıtılmamalıdır.

---

# 6. Neden JSON?

JSON local save için şu avantajları sağlar:

* Human-readable
* Debug edilmesi kolay
* Basit serialization
* Versioning için uygun
* Migration yapılabilir
* Küçük/orta büyüklükte player save için yeterli
* Database dependency'si gerektirmez

Özellikle hybrid-casual template'in ilk versiyonunda SQLite gibi bir database kullanmak zorunlu değildir.

---

# 7. JSON Database Değildir

JSON persistence:

```text id="m3v8qa"
Player Save
```

için kullanılmaktadır.

Şunlar için database olarak düşünülmemelidir:

```text id="q5r9xc"
Large Content Catalog
Complex Queries
Large History
Relational Data
```

Bu tür ihtiyaçlar ortaya çıkarsa ayrı bir local database çözümü değerlendirilebilir.

---

# 8. PlayerPrefs

`PlayerPrefs`, ana player save sistemi değildir.

Basit preference değerleri için kullanılabilir:

```text id="v7c3ma"
Music Enabled
SFX Enabled
Some Local Preference
```

Ancak:

```text id="k8n4xp"
Currency
Progression
Inventory
Level State
Unlocked Content
```

gibi büyüyen player data'sı PlayerPrefs'e dağıtılmamalıdır.

Template'te ana persistence modeli JSON olarak tutulmalıdır.

---

# 9. Save Data Model

Save data persistence için özel bir model kullanmalıdır.

Örneğin:

```csharp id="p4m8za"
[Serializable]
public class PlayerSaveData
{
    public int version;
    public EconomySaveData economy;
    public ProgressionSaveData progression;
    public SettingsSaveData settings;
    public ProfileSaveData profile;
}
```

Bu model runtime object'lerinin birebir kopyası olmak zorunda değildir.

---

# 10. Save Data ve Runtime State Aynı Şey Değildir

Örneğin runtime:

```text id="x6q3vc"
EconomyRuntimeState
├── CurrentAmount
├── PendingTransaction
└── TemporaryModifiers
```

Save:

```text id="r9m4za"
EconomySaveData
└── CurrentAmount
```

Temporary runtime state save edilmeyebilir.

Save model yalnızca persistence gerektiren state'i taşımalıdır.

---

# 11. Save Data ve Configuration Aynı Şey Değildir

Örneğin:

```text id="m7c2xa"
EconomyConfig
    → Starting Currency = 500
```

ve:

```text id="q4r8vp"
PlayerSaveData
    → Currency = 1250
```

aynı data değildir.

Configuration designer tarafından tanımlanır.

Save data oyuncuya aittir.

---

# 12. Save Data Ownership

Persistent data'nın hangi system'e ait olduğu açık olmalıdır.

Örneğin:

```text id="v5n3mc"
EconomySystem
    → EconomySaveData

ProgressionSystem
    → ProgressionSaveData

SettingsSystem
    → SettingsSaveData

ProfileSystem
    → ProfileSaveData
```

SaveSystem bunları tek persistence modelinde birleştirebilir.

---

# 13. Save Flow

Genel save akışı:

```text id="k8m4za"
Runtime State
    ↓
State Owner
    ↓
Create / Update Save Model
    ↓
SaveSystem
    ↓
Serialize
    ↓
Write
```

Örneğin:

```text id="p3r7vc"
EconomySystem
    ↓
EconomySaveData
    ↓
PlayerSaveData
    ↓
SaveSystem
    ↓
JSON
```

---

# 14. Load Flow

Load akışı:

```text id="x5q2ma"
Application Start
    ↓
Bootstrap
    ↓
SaveSystem
    ↓
Read JSON
    ↓
Deserialize
    ↓
Validate
    ↓
Migrate if necessary
    ↓
Apply to Runtime Systems
```

Runtime system'ler load sonucunu kendi state'lerine uygular.

---

# 15. Bootstrap ile İlişki

SaveSystem initialization sırasında load edilebilir.

Önerilen akış:

```text id="r8m3xa"
Bootstrap
    ↓
Initialize Core Systems
    ↓
Load Player Save
    ↓
Validate / Migrate
    ↓
Apply Persistent State
    ↓
Runtime Systems Ready
    ↓
GameFlow
```

Save data hazır olmadan progression/economy gibi persistent state'e bağımlı sistemlerin çalışmaya başlamasına izin verilmemelidir.

---

# 16. First Launch

İlk launch'ta save file bulunmayabilir.

Akış:

```text id="q7c4vp"
Load
 ↓
File Exists?
 ├── Yes → Deserialize
 └── No  → Create Default Save
```

Default save:

```text id="m9r2xa"
Default Configuration
        ↓
Initial PlayerSaveData
```

ile oluşturulabilir.

Default değerler mümkün olduğunca configuration'dan gelmelidir.

---

# 17. Default Save

Örneğin:

```text id="v4n8mc"
First Launch
    ↓
Create Default PlayerSaveData
    ↓
Currency = Config.StartingCurrency
    ↓
Current Level = Config.StartingLevel
    ↓
Default Settings
    ↓
Save
```

İlk save creation logic tek bir yerde yönetilmelidir.

---

# 18. Save Version

Save formatında version bulunmalıdır.

Örneğin:

```json
{
    "version": 3
}
```

Version:

```text id="k5m7za"
Save Format Version
```

anlamına gelir.

Game version veya app version ile birebir aynı olmak zorunda değildir.

---

# 19. Version Neden Gerekli?

Save model zaman içinde değişebilir.

Örneğin:

```text id="p8r3vc"
Version 1
    ↓
currency

Version 2
    ↓
softCurrency
premiumCurrency

Version 3
    ↓
currency objects
```

Migration olmadan eski save doğrudan yeni modele uygulanırsa data kaybı veya invalid state oluşabilir.

---

# 20. Migration

Eski save yeni format'a dönüştürülmelidir.

Örneğin:

```text id="x4q8ma"
v1 Save
   ↓
Migration v1 → v2
   ↓
Migration v2 → v3
   ↓
Current Save
```

Migration zinciri kontrollü ve deterministic olmalıdır.

---

# 21. Migration Ownership

Migration logic SaveSystem veya SaveMigration katmanında tutulabilir.

Örneğin:

```text id="m7c3xp"
SaveSystem
    ↓
SaveMigration
    ├── V1ToV2
    └── V2ToV3
```

Gameplay system'leri eski save formatlarını bilmemelidir.

---

# 22. Migration Rules

Migration:

* Deterministic olmalı
* Mümkün olduğunca idempotent olmalı
* Data kaybını önlemeli
* Eski version'dan current version'a ulaşabilmeli
* Test edilebilir olmalı

Migration sırasında runtime gameplay logic çalıştırılmamalıdır.

---

# 23. Migration Testing

Her migration için test oluşturmak faydalıdır.

Örneğin:

```text id="q6r4va"
Given V1 Save
    ↓
Migrate
    ↓
Expected V3 Save
```

Özellikle:

* Currency
* Progression
* Inventory
* Unlocks
* Settings

gibi kritik alanlar test edilmelidir.

---

# 24. Save Validation

Deserialize edilmiş data doğrudan runtime'a uygulanmamalıdır.

Önce validation yapılabilir.

Örneğin:

```text id="v8m2xc"
Save Loaded
    ↓
Schema Validation
    ↓
Value Validation
    ↓
Reference Validation
    ↓
Apply
```

Validation örnekleri:

```text id="r5q7za"
Currency >= 0
Level >= MinimumLevel
Enum values valid
Required collections not null
IDs valid
```

Validation rules system-specific olabilir.

---

# 25. Invalid Save

Save invalid ise sistem kontrollü davranmalıdır.

Örneğin:

```text id="k4m8vc"
Invalid Save
    ↓
Try Recovery
    ↓
Backup / Previous Save
    ↓
Fallback Default Save
```

Oyuncunun progress'ini doğrudan sessizce sıfırlamak son seçenek olmalıdır.

---

# 26. Corrupted Save

JSON parse edilemiyorsa:

```text id="p7c3mx"
Read File
    ↓
Deserialize Failed
```

durumu corruption olarak ele alınabilir.

Önerilen yaklaşım:

```text id="x8r4qa"
player_save.json
        ↓
Backup / Temp / Previous
        ↓
Attempt Recovery
        ↓
If Impossible
        ↓
Fallback
```

Exact recovery strategy platform ve project requirements'a göre belirlenebilir.

---

# 27. Atomic Write

Save yazarken mevcut save'in yarıda bozulması engellenmelidir.

Basit yaklaşım:

```text id="m5q9vc"
Serialize
    ↓
Write Temporary File
    ↓
Validate / Flush
    ↓
Replace Existing Save
```

Örneğin:

```text
player_save.tmp
```

üzerinden yazılıp tamamlandıktan sonra ana save'e geçirilmesi tercih edilebilir.

Platform-specific file semantics dikkate alınmalıdır.

---

# 28. Save Backup

Kritik save'lerde backup tutulabilir.

Örneğin:

```text id="v3n8xa"
player_save.json
player_save.backup.json
```

Backup strategy:

```text id="q6m2rc"
Current Save
    ↓
Backup Previous
    ↓
Write New Save
```

şeklinde olabilir.

Backup sayısı gereksiz büyütülmemelidir.

---

# 29. Save Frequency

Save her frame'de yapılmamalıdır.

Yanlış:

```text id="x7c4ma"
Update
 ↓
Save
```

Save trigger'ları anlamlı state transition'lara bağlanabilir.

Örneğin:

```text id="m8r2vp"
Level Completed
Purchase Completed
Progression Changed
Settings Changed
Application Pause
Application Quit
Important State Change
```

---

# 30. Debounced Save

Birden fazla state kısa süre içinde değişiyorsa save operation birleştirilebilir.

Örneğin:

```text id="p5q8xa"
Currency Changed
Progression Changed
Settings Changed
      ↓
Pending Save
      ↓
Single Write
```

Bu yaklaşım özellikle aynı frame veya kısa zaman aralığında çok sayıda state update olduğunda disk I/O'yu azaltabilir.

---

# 31. Immediate Save

Bazı kritik işlemler immediate save gerektirebilir.

Örneğin:

```text id="v4m7qc"
Purchase Completed
```

gibi değerli state değişikliklerinde persistence timing daha sıkı tutulabilir.

Ancak her event için synchronous disk write yapmak performansı olumsuz etkileyebilir.

Karar state'in önemine göre verilmelidir.

---

# 32. Application Pause

Mobile'da application background'a girebilir.

SaveSystem gerekirse:

```text id="k9r3xa"
OnApplicationPause(true)
    ↓
Flush Pending Save
```

yapabilir.

Ancak pause callback'inin her platformda sonsuz çalışma süresi garanti etmediği dikkate alınmalıdır.

Kritik state'leri yalnızca pause'a güvenerek saklamak doğru değildir.

---

# 33. Application Quit

`OnApplicationQuit` ek bir save fırsatı olabilir.

Ancak mobile lifecycle'da process'in kill edilmesi nedeniyle buna güvenilmemelidir.

Bu nedenle önemli state transition'larda save daha önce tetiklenmelidir.

---

# 34. Save Queue

Birden fazla save request geldiğinde duplicate write'lar azaltılabilir.

Örneğin:

```text id="m6q2vc"
Request Save
Request Save
Request Save
      ↓
Pending Save
      ↓
Single Write
```

SaveSystem pending state tutabilir.

Ancak queue mekanizması gereksiz complexity yaratıyorsa daha basit dirty flag yaklaşımı yeterlidir.

---

# 35. Dirty State

System'ler persistence gerektiren state değiştiğinde SaveSystem'e dirty signal verebilir.

Örneğin:

```text id="x8r4ma"
Economy Changed
    ↓
Save Dirty
```

Sonrasında:

```text id="p7m3vc"
Save Trigger
    ↓
Write
```

uygulanabilir.

Dirty state SaveSystem'in source of truth olması anlamına gelmez.

---

# 36. Save Threading

Dosya I/O'nun main thread üzerindeki etkisi dikkate alınmalıdır.

Save implementation gerektiğinde background operation kullanabilir.

Ancak:

```text id="v5q8xa"
Background Thread
    ↓
Unity Object
```

doğrudan erişmemelidir.

Save model serialization öncesinde gerekli Unity-dependent data main thread'de hazırlanmalıdır.

---

# 37. Async Save

Async save kullanılabilir.

Örneğin:

```text id="m4c7za"
Prepare Save Data
    ↓
Serialize
    ↓
Async File Write
```

Ancak pending write lifecycle açık olmalıdır.

Bir scene veya application lifecycle kapanırken save operation'ın ne olacağı tanımlanmalıdır.

---

# 38. Concurrent Save

Aynı anda birden fazla write çalıştırılmamalıdır.

Risk:

```text id="q8r3mc"
Save A ────────┐
               ├── Same File
Save B ────────┘
```

race condition oluşturabilir.

SaveSystem write serialization veya single-writer model kullanmalıdır.

---

# 39. Save File Path

Save path tek bir merkezi noktadan oluşturulmalıdır.

Örneğin konsept olarak:

```csharp id="n7m4xp"
var path = Path.Combine(
    Application.persistentDataPath,
    "player_save.json");
```

Ancak path logic farklı system'lere kopyalanmamalıdır.

---

# 40. File Format

JSON formatı:

* Stable
* Versioned
* Serializable
* Debuggable

olmalıdır.

Runtime Unity component'lerinin doğrudan serialize edilmesi tercih edilmez.

Örneğin:

```text id="x5q8va"
GameObject
MonoBehaviour
Transform
```

gibi Unity object reference'ları save modelinin parçası olmamalıdır.

---

# 41. IDs

Persistent content reference gerekiyorsa stable ID kullanılmalıdır.

Örneğin:

```text id="m3r7xc"
avatar_03
level_025
product_100
```

gibi ID'ler save data'da tutulabilir.

Save data:

```text id="q8v4za"
"avatar_03"
```

saklar.

Asset reference'ın kendisini JSON'a serialize etmez.

---

# 42. Save ve ScriptableObject Reference

ScriptableObject reference doğrudan save modeline konulmamalıdır.

Yanlış:

```text id="p5m9xc"
PlayerSaveData
    ↓
AvatarConfig ScriptableObject Reference
```

Doğru:

```text id="v7q3za"
PlayerSaveData
    ↓
AvatarId = "avatar_03"
    ↓
Configuration
    ↓
Resolve Avatar
```

---

# 43. Save ve Addressables

Addressable asset reference gerekiyorsa save data stable asset/content ID saklayabilir.

Örneğin:

```text id="x4m8vc"
Save
    ↓
ContentId
    ↓
Configuration / Catalog
    ↓
Addressable Asset
```

Save data doğrudan runtime Addressable handle saklamamalıdır.

---

# 44. Save Data Size

Player save mümkün olduğunca küçük tutulmalıdır.

Save'e şunları koymak yerine:

```text id="k7r2xa"
Entire Config
Entire Level Data
Entire Prefab State
Entire Asset Data
```

yalnızca player-specific persistence state tutulmalıdır.

Örneğin:

```text id="m5q8vc"
Current Level
Currency
Unlocks
Settings
Profile
```

gibi.

---

# 45. Inventory

Inventory save edilecekse yalnızca gerekli persistent state saklanmalıdır.

Örneğin:

```text id="v8c3ma"
item_001 : 5
item_014 : 2
```

gibi ID + quantity modeli kullanılabilir.

Inventory object'lerinin tamamını serialize etmek gereksiz olabilir.

---

# 46. Time-Based State

Time-dependent state için timestamp kullanılabilir.

Örneğin:

```text id="q4m7xc"
LastRewardClaimUtc
```

gibi bir değer save edilebilir.

Local device clock manipülasyonu gibi konular ayrı bir anti-cheat/server-authority problemidir ve SaveSystem'in sorumluluğu değildir.

---

# 47. Save Security

JSON okunabilir olduğu için local save güvenli bir storage değildir.

Oyuncu cihazdaki dosyaya erişebiliyorsa data değiştirilebilir.

Bu nedenle:

```text id="x6r2va"
Currency = 999999
```

gibi değerlerin local save'de bulunması güvenlik açısından tek başına korunmuş değildir.

Competitive veya server-authoritative oyunlarda kritik state backend tarafından doğrulanmalıdır.

---

# 48. Encryption

Save encryption ancak gerçek bir requirement varsa kullanılmalıdır.

Encryption:

* Cheating'i tamamen engellemez
* Local tampering'i mutlak olarak önlemez
* Debugging'i zorlaştırabilir
* Key management gerektirir

Bu nedenle template'te encryption varsayılan olarak zorunlu değildir.

---

# 49. Checksums / Integrity

Save integrity gerekiyorsa checksum/signature kullanılabilir.

Örneğin:

```text id="p8n4mc"
Payload
    ↓
Checksum / Signature
    ↓
Save
```

Load sırasında:

```text id="m7q3xa"
Read
 ↓
Verify
 ↓
Deserialize
```

yapılabilir.

Bu da gerçek requirement varsa eklenmelidir.

---

# 50. Cloud Save

Cloud save core local persistence'in yerine geçmez.

İleri aşamada:

```text id="v4r8za"
SaveSystem
      ↓
Persistence Layer
      ├── Local
      └── Cloud
```

gibi bir yapı kurulabilir.

Ancak ilk implementation:

```text id="q6m3xc"
SaveSystem
      ↓
Local JSON
```

olabilir.

Cloud backend gerekmiyorsa template'e eklenmemelidir.

---

# 51. Cloud Conflict

Cloud save eklendiğinde conflict resolution ayrıca tasarlanmalıdır.

Örneğin:

```text id="x8r2va"
Local Save
      +
Cloud Save
      ↓
Conflict
```

olası stratejiler:

```text id="m5c7qa"
Latest Timestamp
Server Authority
Progress Merge
Player Choice
```

olabilir.

Bu problem local SaveSystem'in basit JSON implementation'ına şimdiden gömülmemelidir.

---

# 52. Repository Boundary

İleride local/cloud ayrımı gerektiğinde persistence boundary açık tutulabilir.

Konsept:

```text id="v7q4mc"
SaveSystem
    ↓
Persistence
    ├── Local Save
    └── Cloud Save
```

Ancak sırf gelecekte cloud gelebilir diye bugünden çok sayıda interface eklemek zorunlu değildir.

Gerçek ihtiyaç ortaya çıktığında abstraction eklenebilir.

---

# 53. SaveSystem ve Economy

EconomySystem currency state'in owner'ıdır.

Örneğin:

```text id="k8m2za"
EconomySystem
    ↓
Currency Changed
    ↓
Save Dirty
    ↓
SaveSystem
```

SaveSystem currency hesaplamaz.

---

# 54. SaveSystem ve Progression

ProgressionSystem progression state'in owner'ıdır.

```text id="p4r7xc"
ProgressionSystem
    ↓
Progression Changed
    ↓
SaveSystem
```

SaveSystem:

```text id="x9m3va"
"Player hangi levelda?"
```

sorusunun domain owner'ı değildir.

Sadece persistence sağlar.

---

# 55. SaveSystem ve Settings

Settings persistence:

```text id="m6q8za"
SettingsSystem
    ↓
Settings Changed
    ↓
SaveSystem
```

şeklinde olabilir.

UI doğrudan JSON'a yazmamalıdır.

---

# 56. SaveSystem ve Profile

ProfileSystem:

```text id="v3c7mx"
Profile State
```

owner'ıdır.

SaveSystem:

```text id="q5r8za"
ProfileSaveData
```

persist eder.

Profile UI doğrudan save file'a erişmemelidir.

---

# 57. SaveSystem ve UI

UI SaveSystem'in implementation'ını bilmemelidir.

Yanlış:

```text id="k7m4vc"
SettingsPopup
    ↓
Write JSON
```

Doğru:

```text id="p8r2xa"
SettingsPopup
    ↓
SettingsSystem
    ↓
SaveSystem
```

---

# 58. Save Triggers

Save trigger'ları aşağıdaki gibi olabilir:

```text id="x4q9mc"
Important State Change
Level Completion
Purchase Completion
Reward Granted
Progression Change
Settings Change
Application Pause
Explicit Save Request
```

Her frame save yapılmamalıdır.

---

# 59. Save Atomicity

Bir logical save operation mümkün olduğunca tek coherent snapshot olmalıdır.

Örneğin:

```text id="m8r3xa"
Economy = 1000
Progression = Level 10
Profile = Avatar 03
```

aynı save snapshot içinde bulunmalıdır.

Bir system yazıp diğerini eski state'te bırakacak partial save modellerinden kaçınılmalıdır.

---

# 60. Snapshot

Save operation sırasında persistence modeli snapshot olarak hazırlanabilir.

Örneğin:

```text id="q6c4vp"
Runtime Systems
      ↓
Build PlayerSaveData
      ↓
Immutable Snapshot
      ↓
Serialize / Write
```

Böylece async write sırasında runtime state'in değişmesiyle save data'nın tutarsızlaşması önlenebilir.

---

# 61. Save Failure

Save write başarısız olursa:

```text id="v7m2xc"
Write Failed
    ↓
Log / Report
    ↓
Retry if appropriate
    ↓
Keep Dirty State
```

gibi bir davranış uygulanabilir.

Başarısız save sonrası dirty state temizlenmemelidir.

---

# 62. Retry

Retry idempotent olmalıdır.

Örneğin:

```text id="x3q8za"
Save Failed
    ↓
Retry
    ↓
Same Logical Snapshot
```

mümkün olduğunca tekrar güvenli şekilde yazılabilmelidir.

Retry infinite loop oluşturmamalıdır.

---

# 63. Save Logging

Development/debug ortamında:

```text id="m5r7vc"
Save Started
Save Completed
Save Failed
Load Started
Load Completed
Migration Started
Migration Completed
```

gibi loglar faydalı olabilir.

Ancak production'da yüksek frekanslı save logging gereksiz log spam oluşturmamalıdır.

---

# 64. Save Metrics

Gerektiğinde aşağıdaki metrics izlenebilir:

```text id="k8q4ma"
Save Count
Load Count
Save Duration
Save Size
Load Duration
Migration Count
Migration Duration
Failure Count
```

Özellikle mobile performance debugging için save duration faydalıdır.

---

# 65. Testing

SaveSystem için:

### Serialization

```text id="p7m3xc"
Runtime Data
    ↓
Save Data
    ↓
JSON
    ↓
Save Data
```

round-trip test edilebilir.

### Migration

```text id="v4q8za"
V1
 ↓
V2
 ↓
V3
```

test edilmelidir.

### Corruption

Invalid JSON ve invalid values test edilmelidir.

### Recovery

Backup/fallback davranışı test edilmelidir.

### Atomic Write

Temporary file → final file flow doğrulanmalıdır.

---

# 66. EditMode Tests

Scene gerektirmeyen save logic için `EditMode` testleri tercih edilmelidir.

Özellikle:

* Serialization
* Deserialization
* Migration
* Validation
* Default save creation
* Save model conversion
* ID resolution

test edilebilir.

---

# 67. PlayMode Tests

Unity lifecycle gerektiren durumlar için `PlayMode` testleri kullanılabilir.

Örneğin:

* Bootstrap + Load
* Application lifecycle
* Runtime state application
* Save trigger integration
* System initialization ordering

---

# 68. Serialization Safety

Save model serialization contract'tır.

Şunlar değiştirilirken dikkat edilmelidir:

* Field names
* Field types
* Enum values
* Nested structures
* Collection types
* Default values

Eski save'lerin desteklenmesi gerekiyorsa migration veya compatibility strategy kullanılmalıdır.

---

# 69. Enum Safety

Enum değerleri save'e numeric olarak serialize ediliyorsa enum sırası değiştirilirken dikkat edilmelidir.

Örneğin:

```text id="m6r2xc"
Old:
0 = Soft
1 = Premium
```

değerlerini:

```text id="q8v4za"
New:
0 = Premium
1 = Soft
```

şeklinde değiştirmek eski save'leri bozabilir.

Enum persistence için stable IDs/string representation gerektiğinde tercih edilebilir.

---

# 70. Collection Safety

Save data collection'ları:

* Null
* Empty
* Duplicate
* Invalid ID

durumlarına karşı validate edilmelidir.

Örneğin unlocked content listesinde duplicate ID bulunması runtime state'in beklenmedik şekilde oluşmasına neden olabilir.

---

# 71. Save and Content Updates

Yeni content eklenmesi save migration gerektirmeyebilir.

Örneğin:

```text id="v5m8qa"
New Level 101
```

eklenmesi yalnızca configuration update olabilir.

Ancak eski player state'in anlamı değişiyorsa migration gerekir.

Migration kararı:

> Content değişti mi?

yerine:

> Existing persisted data'nın schema veya semantics'i değişti mi?

sorusuyla verilmelidir.

---

# 72. Save and Deleted Content

Bir content ID artık configuration'da yoksa save:

```text id="q4r7mc"
Unknown Content ID
```

durumunu kontrollü ele almalıdır.

Seçenekler:

```text id="m8c3xa"
Ignore
Migrate
Replace
Mark Invalid
```

game design'e göre belirlenebilir.

Save load sırasında unknown ID yüzünden bütün save'in çöpe gitmesi engellenmelidir.

---

# 73. Save and Version Compatibility

Current build yalnızca desteklediği save version'ları yüklemelidir.

Örneğin:

```text id="x7m2va"
Save Version
    ↓
Current Version?
 ├── Older → Migrate
 ├── Same → Load
 └── Newer → Handle Unsupported Version
```

Daha yeni bir save version'ı eski client tarafından okunamıyorsa save'in üzerine yazılmamalıdır.

---

# 74. Unsupported Future Save

Örneğin:

```text id="p5r8xc"
Installed App → supports V3
Save → V4
```

durumunda:

```text id="v9m3qa"
Do Not Overwrite
```

önemlidir.

Eski client'in bilmediği save'i default save ile ezmesi ciddi data loss yaratabilir.

---

# 75. Backup Before Migration

Migration sırasında mümkünse eski save'in backup'ı alınabilir.

Örneğin:

```text id="k4q7mc"
V1 Save
    ↓
Backup V1
    ↓
Migrate
    ↓
V3 Save
```

Migration bug'larının recover edilmesini kolaylaştırır.

---

# 76. Save Reset

Oyuncunun local save'i resetlenmesi gerektiğinde işlem SaveSystem üzerinden yapılmalıdır.

Örneğin:

```text id="x8m2va"
Reset Player Data
    ↓
Clear Save
    ↓
Create Default Save
```

UI:

```text id="q6r3vc"
"Delete Data" Intent
```

gönderir.

UI doğrudan file delete yapmaz.

---

# 77. Debug Save Tools

Development build'de faydalı araçlar olabilir:

```text id="m7c4xa"
Export Save
Import Save
Reset Save
Print Save
Validate Save
Force Migration
```

Bu araçlar production UI'ın parçası olmak zorunda değildir.

---

# 78. Save and Cheat Prevention

Local save güvenilir bir authority değildir.

Eğer oyun:

```text id="v3q8mc"
Competitive
Online
Leaderboards
Real-Money Economy
```

gibi kritik güvenlik gerektiriyorsa server authority düşünülmelidir.

SaveSystem local convenience/persistence sağlar.

Anti-cheat system değildir.

---

# 79. Local Save as Cache

Local save bazı sistemler için cache gibi de davranabilir.

Örneğin:

```text id="p8m2za"
Remote Data
    ↓
Local Cache
```

Ancak cache ile authoritative player save birbirine karıştırılmamalıdır.

Cache:

```text id="x4r7vc"
Re-downloadable
```

olabilir.

Player progress:

```text id="m5q8xa"
Persistent / Important
```

olabilir.

---

# 80. Database Decision

Template'in başlangıç persistence çözümü:

```text id="v7c3ma"
JSON
+
Application.persistentDataPath
```

olarak kabul edilir.

SQLite veya başka bir local database ancak aşağıdaki ihtiyaçlardan biri ortaya çıkarsa değerlendirilmelidir:

```text id="q8m4xc"
Large Local Dataset
Complex Queries
Relational Data
Large History
Offline Database
```

Player save'in sırf "database daha profesyonel" göründüğü için SQLite'a taşınması gerekmez.

---

# 81. Recommended Data Flow

Template için önerilen genel yapı:

```text id="r5x8qa"
              CONFIGURATION
                   │
                   ↓
              Runtime Systems
                   │
             Runtime State
                   │
                   ↓
              Save Trigger
                   │
                   ↓
               SaveSystem
                   │
              PlayerSaveData
                   │
                   ↓
             Serialization
                   │
                   ↓
                JSON
                   │
                   ↓
       Application.persistentDataPath
```

Load:

```text id="k7m3vc"
JSON
 ↓
Deserialize
 ↓
Version Check
 ↓
Migration
 ↓
Validation
 ↓
PlayerSaveData
 ↓
Runtime Systems
```

---

# 82. Responsibility Matrix

| Sorumluluk          | Owner                          |
| ------------------- | ------------------------------ |
| Currency state      | EconomySystem                  |
| Progression state   | ProgressionSystem              |
| Settings state      | SettingsSystem                 |
| Profile state       | ProfileSystem                  |
| Gameplay state      | Gameplay System                |
| Persistent model    | SaveSystem                     |
| JSON serialization  | SaveSystem / serializer        |
| File path           | Local Save Storage             |
| Migration           | Save Migration                 |
| Asset configuration | ScriptableObject               |
| UI presentation     | UI                             |
| Cloud persistence   | Future Cloud Persistence Layer |

---

# 83. Definition of Done

SaveSystem implementation tamamlanmış sayılmadan önce:

* [ ] Runtime state ve save data ayrılmış mı?
* [ ] Configuration ve save data ayrılmış mı?
* [ ] State owner'ları açık mı?
* [ ] Local persistence JSON olarak tanımlı mı?
* [ ] `Application.persistentDataPath` kullanılıyor mu?
* [ ] File path merkezi mi?
* [ ] PlayerPrefs ana save sistemi olarak kullanılmıyor mu?
* [ ] Save version bulunuyor mu?
* [ ] Migration strategy var mı?
* [ ] Migration testleri var mı?
* [ ] Save validation var mı?
* [ ] Corruption handling var mı?
* [ ] Backup/recovery strategy tanımlı mı?
* [ ] Atomic write uygulanıyor mu?
* [ ] Concurrent write engelleniyor mu?
* [ ] Save frequency kontrol altında mı?
* [ ] Dirty/pending save yaklaşımı tanımlı mı?
* [ ] Application pause dikkate alınıyor mu?
* [ ] Quit lifecycle'a tek başına güvenilmiyor mu?
* [ ] Async write lifecycle güvenli mi?
* [ ] Snapshot consistency korunuyor mu?
* [ ] Save failure sonrası dirty state korunuyor mu?
* [ ] Unsupported future save overwrite edilmiyor mu?
* [ ] Unknown content ID kontrollü ele alınıyor mu?
* [ ] Enum/serialization değişiklikleri güvenli mi?
* [ ] Runtime Unity object reference'ları save'e yazılmıyor mu?
* [ ] Stable content IDs kullanılıyor mu?
* [ ] UI doğrudan storage'a erişmiyor mu?
* [ ] Gameplay system'leri JSON'a doğrudan erişmiyor mu?
* [ ] Cloud save local save'den ayrılabilecek şekilde sınırlandırılmış mı?
* [ ] Gereksiz database dependency'si eklenmemiş mi?
* [ ] Mobile lifecycle dikkate alınmış mı?
* [ ] EditMode/PlayMode testleri uygun şekilde eklenmiş mi?
