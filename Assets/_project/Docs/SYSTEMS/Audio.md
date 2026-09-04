# Audio System

## 1. Amaç

`AudioSystem`, oyundaki müzik, ses efektleri, ambience ve gerektiğinde voice/audio feedback davranışlarının merkezi runtime yönetiminden sorumludur.

Temel amaç:

* Audio playback'i merkezi ve kontrollü hale getirmek
* UI, gameplay ve sistemlerin doğrudan `AudioSource` yönetmesini engellemek
* Music ve SFX yaşam döngüsünü yönetmek
* Audio settings ile playback arasında bağlantı kurmak
* Audio event'lerini dinleyerek uygun sesleri oynatabilmek
* Scene geçişlerinde gerekli audio lifecycle'ını korumak
* Pooling veya reusable audio source kullanımını gerektiğinde yönetmek
* AssetManagement ile audio asset loading/lifetime ayrımını korumak

Temel prensip:

> Gameplay veya UI "hangi sesin nasıl çalacağını" doğrudan yönetmez. İlgili sistem bir audio intent/event üretir, AudioSystem bunu uygun şekilde yorumlar.

---

# 2. Temel Mimari

Genel yapı:

```text
Gameplay / UI / Systems
          ↓
     Audio Intent
          ↓
      AudioSystem
          ↓
     Audio Definition
          ↓
      Audio Playback
          ↓
      AudioSource
```

Örneğin:

```text
LevelCompleted
      ↓
AudioSystem
      ↓
LevelComplete SFX
```

ve:

```text
Button Click
      ↓
UI Event / Audio Intent
      ↓
AudioSystem
      ↓
Button Click SFX
```

Music için:

```text
GameFlow
   ↓
Music Intent
   ↓
AudioSystem
   ↓
Music Track
```

---

# 3. AudioSystem Sorumlulukları

`AudioSystem`:

* SFX playback
* Music playback
* Audio volume yönetimi
* Mute state
* Audio category kontrolü
* Audio lifecycle
* Audio event mapping
* Audio source reuse
* Gerekirse music transition
* Gerekirse audio asset loading/release koordinasyonu

konularından sorumludur.

`AudioSystem`:

* Gameplay state'in sahibi değildir.
* UI state'in sahibi değildir.
* Economy state'in sahibi değildir.
* SaveSystem değildir.
* AnalyticsSystem değildir.
* Asset database değildir.

---

# 4. Audio Kategorileri

Template başlangıcında aşağıdaki kategoriler yeterlidir:

```text
Master
Music
SFX
Voice
```

İhtiyaca göre:

```text
Ambient
UI
```

gibi ek kategoriler oluşturulabilir.

Gereksiz yere onlarca mixer group oluşturulmamalıdır.

---

# 5. Audio Configuration

Audio configuration ScriptableObject üzerinden tanımlanabilir.

Örneğin:

```text
AudioConfig
├── Default Volumes
├── Music Settings
├── SFX Settings
├── Voice Settings
└── Audio Definitions
```

Audio asset referansları da configuration üzerinden tutulabilir.

Örneğin:

```text
AudioDefinition
├── Id
├── Clip
├── Category
├── Volume
├── Pitch
└── Loop
```

Buradaki değerler static/designer configuration'dır.

Runtime volume state configuration asset'inin üzerine yazılmamalıdır.

---

# 6. Runtime Audio State

Configuration:

```text
AudioConfig
```

statik ayarları sağlar.

Runtime:

```text
AudioSystem
├── Current Music
├── Current Volume
├── Mute State
└── Active Playback
```

gibi bilgileri yönetir.

Save edilecek kullanıcı ayarları:

```text
SettingsSystem
```

tarafından yönetilir.

Örneğin:

```text
SettingsSystem
    ↓
Music Volume = 0.7
    ↓
AudioSystem
```

---

# 7. SettingsSystem ile İlişki

Audio ayarlarının sahibi `SettingsSystem` olabilir.

Örneğin:

```text
SettingsPopup
      ↓
SettingsSystem
      ↓
Music Volume
SFX Volume
Mute
      ↓
AudioSystem
```

AudioSystem kendi playback davranışını bu runtime settings'e göre uygular.

SettingsPopup doğrudan:

```text
AudioSource.volume = ...
```

yapmamalıdır.

---

# 8. SaveSystem ile İlişki

Audio ayarlarının persistence'ı:

```text
SettingsSystem
      ↓
SaveSystem
```

üzerinden yapılır.

AudioSystem JSON'a doğrudan yazmaz.

Yanlış:

```text
AudioSystem
    ↓
PlayerSaveData
```

Doğru:

```text
SettingsSystem
    ↓
SaveSystem
```

AudioSystem yalnızca runtime playback'i etkiler.

---

# 9. SFX Playback

SFX için basit API yeterlidir.

Örneğin:

```csharp
PlaySfx(audioId);
```

veya context gerekiyorsa:

```csharp
PlaySfx(audioId, position);
```

AudioSystem:

* AudioDefinition'ı çözer.
* Uygun source'u bulur.
* Volume/category ayarlarını uygular.
* Clip'i oynatır.

Gameplay sistemi `AudioSource` ile uğraşmaz.

---

# 10. Positional SFX

3D veya positional audio gerçekten gerekiyorsa:

```text
Gameplay Event
      ↓
AudioSystem
      ↓
Position
      ↓
AudioSource
```

kullanılabilir.

Örneğin:

```text
Explosion
Position = world position
```

AudioSystem uygun source'u o pozisyonda oynatabilir.

UI SFX için positional audio genellikle gerekli değildir.

---

# 11. UI Audio

UI butonları veya popup feedback'leri için:

```text
ButtonClicked
      ↓
AudioSystem
      ↓
UISfx
```

kullanılabilir.

Ancak her UI prefab'ına bağımsız audio orchestration koymak gereksiz olabilir.

Basit projede:

```text
CommonButton
    ↓
Audio Intent
    ↓
AudioSystem
```

yeterlidir.

---

# 12. Gameplay Audio

Gameplay audio domain event'lerinden türetilebilir.

Örneğin:

```text
LevelCompleted
      ↓
AudioSystem
      ↓
LevelCompleteSfx
```

ve:

```text
GameplayActionCompleted
      ↓
AudioSystem
      ↓
ActionSfx
```

GameplaySystem'in:

```csharp
audioSource.PlayOneShot(...)
```

çağırması tercih edilmez.

---

# 13. Event-Based Audio

AudioSystem event consumer olarak çalışabilir.

Örneğin:

```text
LevelCompleted
CurrencyChanged
PurchaseCompleted
TutorialCompleted
```

gibi event'ler audio feedback üretebilir.

Ancak her event'in otomatik olarak sese dönüştürülmesi zorunlu değildir.

Audio mapping configuration üzerinden seçilebilir.

---

# 14. Direct Audio Intent

Bazı sesler event yerine doğrudan intent ile çağrılabilir.

Örneğin:

```text
UI
 ↓
AudioSystem.PlaySfx("button_click")
```

Bu, event'in gereksiz abstraction olduğu basit durumlarda kabul edilebilir.

Karar:

```text
Local explicit relationship
    → Direct call

Decoupled domain reaction
    → Event
```

olmalıdır.

Her audio request EventBus üzerinden gönderilmemelidir.

---

# 15. Music Lifecycle

Music için lifecycle:

```text
No Music
    ↓
Play Track
    ↓
Playing
    ↓
Change Track
    ↓
Fade Out
    ↓
Fade In
```

şeklinde yönetilebilir.

GameFlow state değişimleri music değişikliğini tetikleyebilir.

Örneğin:

```text
MainMenu
    ↓
Gameplay
    ↓
Win
```

her state için farklı music configuration bulunabilir.

---

# 16. GameFlow ile İlişki

GameFlow global state'in sahibidir.

AudioSystem global game state'i yönetmez.

Örneğin:

```text
GameFlow → Gameplay
       ↓
AudioSystem → Gameplay Music
```

GameFlow:

```text
"Şu an Gameplay state'indeyiz."
```

der.

AudioSystem:

```text
"Gameplay state'i için tanımlı music'i oynat."
```

şeklinde davranır.

---

# 17. Music Ownership

Aynı anda birden fazla sistemin music kontrol etmesi engellenmelidir.

Yanlış:

```text
MainMenuController → PlayMusic()
GameplayController → PlayMusic()
ShopView → PlayMusic()
SettingsPopup → PlayMusic()
```

Bu yapı zamanla kontrol edilemez hale gelir.

Music playback'in merkezi owner'ı:

```text
AudioSystem
```

olmalıdır.

---

# 18. Music Transition

Music transition için tween veya coroutine kullanılabilir.

Örneğin:

```text
Track A
   ↓
Fade Out
   ↓
Stop
   ↓
Track B
   ↓
Fade In
```

Transition sırasında:

* Scene değişimi
* Object disable
* AudioSystem destroy
* Application pause

gibi lifecycle durumları dikkate alınmalıdır.

---

# 19. Tween Kullanımı

Projede bir tween sistemi kullanılıyorsa audio fade için de aynı sistem kullanılabilir.

Örneğin:

```text
Music Fade
Volume Fade
```

işlemleri tween üzerinden yürütülebilir.

Pooled veya geçici audio object'lerde tween:

```text
OnDisable
    ↓
Kill / Cleanup
```

edilmelidir.

Audio playback gameplay state'in tek source of truth'u olmamalıdır.

---

# 20. AudioSource Yönetimi

`AudioSource` sayısı kontrol altında tutulmalıdır.

Her gameplay event'i için:

```text
Instantiate Audio GameObject
```

yapmak doğru değildir.

Özellikle sık tekrarlanan SFX için reusable source/pool kullanılabilir.

Örneğin:

```text
AudioSource Pool
├── Source 1
├── Source 2
├── Source 3
└── ...
```

---

# 21. Audio Pooling

Audio pooling özellikle:

* Çok sık SFX
* Positional SFX
* Temporary audio sources

için kullanılabilir.

Lifecycle:

```text
Get Source
   ↓
Reset
   ↓
Configure
   ↓
Play
   ↓
Playback Complete
   ↓
Reset
   ↓
Release
```

Pooling yalnızca gerçekten allocation veya source lifecycle problemi varsa kullanılmalıdır.

Basit bir projede tek veya birkaç persistent AudioSource yeterliyse ayrıca pool oluşturmak gereksizdir.

---

# 22. Pooled Audio Reset

Pooled AudioSource tekrar kullanılmadan önce:

```text
clip
volume
pitch
loop
spatialBlend
position
rotation
output mixer
mute
playOnAwake
```

gibi state'ler doğru şekilde resetlenmelidir.

Ayrıca:

```text
Playback state
Tween
Coroutine
Async operation
```

varsa temizlenmelidir.

---

# 23. One-Shot SFX

Kısa SFX için mümkün olduğunda:

```text
PlayOneShot
```

benzeri mekanizmalar tercih edilebilir.

Bu sayede aynı source üzerinde kısa seslerin birbirini gereksiz yere kesmesi engellenebilir.

Ancak aynı anda çok sayıda ses çalınması durumunda voice count kontrol edilmelidir.

---

# 24. Voice Limiting

Mobil platformlarda aynı anda aşırı sayıda SFX çalmak performans ve ses kalitesi açısından problem yaratabilir.

Gerekirse:

```text
Max Concurrent SFX
```

veya kategori bazlı limitler kullanılabilir.

Örneğin:

```text
UI = 2
Gameplay = 8
Voice = 1
```

gibi değerler tamamen oyun gereksinimine göre belirlenmelidir.

İlk sürümde gereksiz karmaşıklık oluşturulmamalıdır.

---

# 25. Duplicate / Spam SFX

Çok hızlı tekrarlanan gameplay event'leri aynı SFX'i aşırı sık oynatabilir.

Örneğin:

```text
Hit
Hit
Hit
Hit
Hit
Hit
```

gereksiz ses duvarına dönüşebilir.

Gerekirse:

```text
Cooldown
Maximum frequency
Priority
Voice limit
```

uygulanabilir.

Bu policy configuration üzerinden kontrol edilebilir.

---

# 26. Audio Priority

Gerekirse AudioDefinition üzerinde priority bulunabilir.

Örneğin:

```text
Critical UI Feedback
    priority = high

Minor Gameplay SFX
    priority = low
```

Voice limit dolduğunda düşük priority sesler atlanabilir.

Bu özellik gerçek ihtiyaç oluştuğunda eklenmelidir.

---

# 27. Audio Asset Management

AudioSystem asset loading sisteminin kendisi değildir.

AssetManagement:

```text
Asset
 ↓
Load
 ↓
Lifetime
 ↓
Release
```

sorumluluğunu taşır.

AudioSystem:

```text
AudioDefinition
 ↓
Request Playback
 ↓
Use Loaded Clip
```

sorumluluğunu taşır.

Bu ayrım korunmalıdır.

---

# 28. Direct Reference vs Addressables

Küçük ve sürekli kullanılan audio asset'leri için direct reference yeterli olabilir.

Örneğin:

```text
AudioDefinition
    ↓
AudioClip
```

Büyük veya dinamik content için Addressables kullanılabilir.

Örneğin:

```text
AudioDefinition
    ↓
Addressable Reference
    ↓
Asset Management
    ↓
AudioClip
```

Seçim asset boyutu, kullanım sıklığı ve lifecycle'a göre yapılmalıdır.

---

# 29. Audio Asset Lifetime

Addressables veya başka async loading sistemi kullanılıyorsa:

```text
Load
 ↓
Play
 ↓
Playback Complete
 ↓
Release
```

lifecycle'ı açık olmalıdır.

Bir clip'in playback'i bitmeden asset release edilmemelidir.

Music gibi uzun süre yaşayan asset'lerde ownership açıkça belirlenmelidir.

---

# 30. Boot Audio

Boot/splash sırasında gerekli müzik veya ses kullanılacaksa boot-critical asset'lerin yükleme stratejisi dikkatli seçilmelidir.

Splash'ta:

```text
SplashConfig
    ↓
Direct Audio Reference
```

gibi basit bir yapı tercih edilebilir.

Boot'un kendisini audio Addressables yüklemesine bağımlı hale getirmek gereksiz bir chicken-and-egg problemi yaratabilir.

---

# 31. Audio ve Splash

SplashView yalnızca presentation'dır.

Yanlış:

```text
SplashView
 ↓
Initialize Audio SDK
```

Doğru:

```text
Bootstrap
 ↓
AudioSystem Initialize
 ↓
SplashView
```

SplashView gerekiyorsa:

```text
AudioSystem.Play(...)
```

intent'i gönderebilir.

---

# 32. Audio ve Settings Popup

SettingsPopup:

```text
User Changes Volume
      ↓
SettingsSystem
      ↓
AudioSystem
```

akışını kullanır.

SettingsPopup doğrudan AudioMixer veya AudioSource değiştirmemelidir.

---

# 33. Audio ve Monetization

Satın alma veya rewarded ad sonucu audio feedback gerekiyorsa:

```text
MonetizationSystem
      ↓
PurchaseCompleted / RewardEarned
      ↓
AudioSystem
      ↓
SFX
```

kullanılabilir.

MonetizationSystem:

```text
AudioSource.Play()
```

yapmamalıdır.

---

# 34. Audio ve Economy

Currency değişimi ses üretmek için event oluşturabilir:

```text
EconomySystem
      ↓
CurrencyChanged
      ↓
AudioSystem
```

Ancak AudioSystem currency state'ini okumak zorunda değildir.

Örneğin:

```text
CurrencyChanged
```

event'i yalnızca gerekli presentation context'i taşıyabilir.

---

# 35. Audio ve UI

UI event'leri audio feedback oluşturabilir.

Örneğin:

```text
Button Click
Popup Open
Popup Close
Tab Selected
Purchase Success
```

Ancak UI'nin her birinde bağımsız audio manager oluşturulmamalıdır.

---

# 36. Audio Event Mapping

Audio event mapping configuration ile yapılabilir.

Örneğin:

```text
LevelCompleted
    ↓
level_complete_sfx
```

```text
PurchaseCompleted
    ↓
purchase_success_sfx
```

```text
TutorialCompleted
    ↓
tutorial_complete_sfx
```

Bu mapping değiştirilebilir olduğunda gameplay code'una dokunmadan ses değiştirilebilir.

---

# 37. Randomized SFX

Aynı olayda farklı clip'lerden biri seçilebilir.

Örneğin:

```text
Hit
 ↓
Hit_01
Hit_02
Hit_03
```

AudioDefinition:

```text
AudioSet
├── Clip 01
├── Clip 02
└── Clip 03
```

şeklinde tutulabilir.

Random seçim gerekiyorsa deterministic test ihtiyacı ayrıca değerlendirilmelidir.

---

# 38. Pitch Variation

Tekrarlanan SFX için küçük pitch variation kullanılabilir.

Örneğin:

```text
Base Pitch
+
Random Variation
```

Bu yalnızca audio presentation katmanında yapılmalıdır.

Gameplay sonucu pitch'e bağlanmamalıdır.

---

# 39. Audio Pause

GameFlow `Pause` state'ine geçtiğinde audio davranışı configuration'a göre belirlenebilir.

Örneğin:

```text
Pause
 ↓
Music Duck
SFX Stop
Voice Stop
```

veya:

```text
Pause
 ↓
Music Continue
```

gibi davranışlar desteklenebilir.

Pause logic gameplay state'in sahibi olmaya devam eder.

---

# 40. Audio Ducking

Voice veya önemli UI feedback sırasında music volume geçici olarak düşürülebilir.

Örneğin:

```text
Voice Starts
      ↓
Music Volume ↓
      ↓
Voice Ends
      ↓
Music Volume ↑
```

Bu özellik gerekiyorsa AudioSystem içinde tutulmalıdır.

Gameplay sistemleri music volume'u elle değiştirmemelidir.

---

# 41. Application Pause

Mobil application background olduğunda audio playback davranışı açıkça tanımlanmalıdır.

Örneğin:

```text
Application Pause
      ↓
AudioSystem
      ↓
Pause Audio
```

Foreground:

```text
Application Resume
      ↓
AudioSystem
      ↓
Resume / Restore
```

Platform davranışı ve proje gereksinimi dikkate alınmalıdır.

---

# 42. Scene Transition

AudioSystem persistent ise:

```text
Bootstrap
 ↓
AudioSystem
 ↓
DontDestroyOnLoad
```

gibi bir lifetime kullanılabilir.

Ancak `DontDestroyOnLoad` otomatik olarak her sistem için doğru çözüm değildir.

Music'in scene boyunca devam etmesi gerekiyorsa persistent lifetime anlamlıdır.

Scene-specific audio:

```text
Scene Load
 ↓
Scene Audio
 ↓
Scene Unload
 ↓
Cleanup
```

şeklinde yönetilebilir.

---

# 43. Duplicate AudioSystem

Bootstrap/persistent object lifecycle nedeniyle birden fazla AudioSystem instance oluşması engellenmelidir.

Örneğin:

```text
Bootstrap
 ↓
AudioSystem A
```

ve sonra:

```text
MainMenu
 ↓
AudioSystem B
```

oluşmamalıdır.

Persistent system lifetime açıkça tanımlanmalıdır.

---

# 44. Initialization

AudioSystem initialization:

```text
Initialize
 ↓
Load / Resolve Configuration
 ↓
Create / Resolve Sources
 ↓
Apply Settings
 ↓
Ready
```

şeklinde ilerleyebilir.

Initialization tekrar çağrıldığında duplicate AudioSource veya duplicate subscription oluşturulmamalıdır.

---

# 45. Error Handling

Audio asset bulunamazsa:

```text
Missing AudioDefinition
Missing Clip
Failed Load
```

gibi durumlar güvenli şekilde ele alınmalıdır.

Bir SFX'in bulunamaması gameplay'i crash ettirmemelidir.

Örneğin:

```text
Missing SFX
 ↓
Log Warning
 ↓
Continue Gameplay
```

uygun olabilir.

Production build'de log seviyesi ayrıca kontrol edilebilir.

---

# 46. Audio Logging

Analytics ile AudioSystem birbirine karıştırılmamalıdır.

AudioSystem:

```text
Audio playback
```

yönetir.

Analytics:

```text
PurchaseCompleted
LevelCompleted
```

gibi business/gameplay event'lerini yönetir.

Her `PlaySfx()` çağrısının analytics event'i olması gerekmemektedir.

---

# 47. Performance

AudioSystem için:

* Sık kullanılan asset'leri gereksiz yere tekrar load etmemek
* Gereksiz `Instantiate/Destroy` yapmamak
* AudioSource'ları gerektiğinde reuse etmek
* Her frame volume güncellememek
* Gereksiz event allocation oluşturmamak
* Asset lifetime'ı doğru yönetmek

önemlidir.

Gameplay hot path'inde:

```text
new AudioSource
Instantiate
Destroy
```

gibi işlemlerden kaçınılmalıdır.

---

# 48. Memory

Audio asset'leri memory açısından pahalı olabilir.

Özellikle:

* Uzun music track'leri
* Voice paketleri
* Büyük ambience dosyaları

için import/load stratejisi dikkatli seçilmelidir.

Tüm audio asset'lerini startup'ta memory'ye almak yalnızca gerçekten gerekli olduğunda yapılmalıdır.

---

# 49. Compression

Audio import settings gameplay kodunun sorumluluğu değildir.

Audio asset'leri için:

* Compression
* Load Type
* Sample Rate
* Streaming

gibi ayarlar asset pipeline/import configuration üzerinden yönetilmelidir.

AssetManagement dokümanı ile uyumlu olmalıdır.

---

# 50. Testing

AudioSystem'in mümkün olduğunca provider-independent test edilebilir olması tercih edilir.

## EditMode

Test edilebilecekler:

* AudioDefinition resolution
* Volume calculation
* Category routing
* Mute behavior
* Audio mapping
* Priority
* Cooldown
* Duplicate playback policy

## PlayMode

Test edilebilecekler:

* AudioSource lifecycle
* Scene transition
* Persistent AudioSystem
* Application pause/resume
* Settings integration
* Asset loading
* Pool reuse

Sesin gerçekten duyulup duyulmadığını otomatik test etmek çoğu durumda unit test seviyesinde gerekli değildir.

---

# 51. AudioSystem API

API mümkün olduğunca küçük tutulmalıdır.

Örneğin:

```csharp
PlaySfx(string audioId);
PlaySfx(string audioId, Vector3 position);

PlayMusic(string musicId);
StopMusic();

SetMusicVolume(float volume);
SetSfxVolume(float volume);
SetMuted(bool muted);
```

Gereksiz:

```csharp
PlaySpecificAudioSource(...)
FindAudioSource(...)
CreateAudioObject(...)
DestroyAudioObject(...)
```

gibi internal implementation detaylarını public API'ye açmaktan kaçınılmalıdır.

---

# 52. Direct Reference Kararı

Her ses için EventBus kullanmak zorunlu değildir.

Örneğin:

```text
Simple Button
 ↓
AudioSystem.PlaySfx("button_click")
```

gayet yeterli olabilir.

Ancak:

```text
LevelCompleted
```

gibi birden fazla sistemin ilgilendiği domain event zaten varsa:

```text
LevelCompleted
 ↓
AudioSystem
```

daha uygun olabilir.

Karar:

```text
Explicit local relationship
    → Direct Reference

Decoupled domain reaction
    → Event
```

---

# 53. Audio ve EventSystem

AudioSystem EventSystem'in sahibi değildir.

EventSystem:

```text
Communication
```

sağlar.

AudioSystem:

```text
Audio playback
```

sağlar.

AudioSystem'i EventBus'ın içine gömmek veya tüm audio API'sini event'e dönüştürmek gereksizdir.

---

# 54. Anti-Pattern'ler

Aşağıdakilerden kaçınılmalıdır:

```text
GameplaySystem → AudioSource
```

```text
ShopView → AudioSource
```

```text
SettingsPopup → AudioMixer
```

```text
MonetizationSystem → AudioSource
```

```text
Every Event → EventBus → Audio
```

```text
Every SFX → Instantiate AudioObject
```

```text
AudioSystem → Save JSON
```

```text
AudioSystem → Analytics SDK
```

```text
Every Scene → New Persistent AudioSystem
```

```text
AudioConfig → Mutable Runtime State
```

---

# 55. Generic Template Rule

AudioSystem generic kalmalıdır.

Core template içerisinde:

```text
KingdomMusic
StarEarnedSound
BuildingCompleteSound
```

gibi oyun-spesifik kavramlar bulunmamalıdır.

Bunun yerine:

```text
Music
SFX
Voice
UI
Gameplay
Purchase
LevelComplete
TutorialComplete
```

gibi generic kategoriler kullanılmalıdır.

---

# 56. Extension Rules

Yeni bir audio özelliği eklerken:

1. Bu gerçekten AudioSystem'in sorumluluğu mu?
2. Event mi, direct intent mi daha uygun?
3. Yeni bir audio state gerekiyor mu?
4. Asset direct reference mı Addressables mı olmalı?
5. Pooling gerçekten gerekli mi?
6. Runtime state ile configuration ayrılmış mı?
7. SettingsSystem ile sınır korunuyor mu?
8. SaveSystem'e doğrudan erişiliyor mu?
9. Audio SDK/provided API projeye gereksiz şekilde sızıyor mu?
10. Scene lifecycle güvenli mi?
11. Mobile pause/resume düşünülmüş mü?
12. Yeni abstraction gerçekten gerekli mi?

---

# 57. Source of Truth

Audio mimarisinde:

```text
AudioConfig
    ↓
Static Audio Configuration
```

```text
SettingsSystem
    ↓
Runtime User Audio Settings
```

```text
AudioSystem
    ↓
Playback State
```

```text
SaveSystem
    ↓
Persistent User Settings
```

şeklinde ayrım yapılır.

AudioSource'ın mevcut state'i kalıcı gameplay state'i değildir.

---

# 58. Definition of Done

Bir audio özelliği tamamlanmadan önce:

* [ ] Audio playback'in sahibi AudioSystem mi?
* [ ] Gameplay/UI doğrudan AudioSource kullanmıyor mu?
* [ ] Configuration ile runtime state ayrılmış mı?
* [ ] SettingsSystem ile entegrasyon doğru mu?
* [ ] Audio ayarları SaveSystem üzerinden persist ediliyor mu?
* [ ] Event kullanımı gerçekten gerekli mi?
* [ ] Gereksiz EventBus kullanımı yok mu?
* [ ] Sık kullanılan audio source'lar gereksiz instantiate edilmiyor mu?
* [ ] Pooling gerekiyorsa reset contract mevcut mu?
* [ ] Music lifecycle açık mı?
* [ ] Scene transition davranışı açık mı?
* [ ] Persistent AudioSystem duplicate oluşmasını engelliyor mu?
* [ ] Application pause/resume düşünülmüş mü?
* [ ] Asset loading/lifetime AssetManagement ile uyumlu mu?
* [ ] Addressables kullanılıyorsa release ownership açık mı?
* [ ] Missing asset durumunda gameplay crash olmuyor mu?
* [ ] AudioSystem analytics SDK çağırmıyor mu?
* [ ] Monetization/UI/gameplay doğrudan audio implementation'a bağımlı değil mi?
* [ ] Mobil memory/performance dikkate alınmış mı?
* [ ] Gereksiz abstraction eklenmemiş mi?
* [ ] Generic template kuralları korunmuş mu?
* [ ] Test edilebilir lifecycle mevcut mu?

---

# 59. Sonuç

Audio mimarisinin temel ayrımı:

```text
Domain System
      ↓
Event / Intent
      ↓
AudioSystem
      ↓
Audio Definition
      ↓
Audio Playback
      ↓
AudioSource
```

Kullanıcı ayarları:

```text
SettingsPopup
      ↓
SettingsSystem
      ↓
AudioSystem
      ↓
Playback
```

Persistence:

```text
SettingsSystem
      ↓
SaveSystem
```

Asset lifecycle:

```text
AssetManagement
      ↓
Load / Release
```

AudioSystem ise bunların arasındaki **playback orchestration** katmanıdır.

En önemli prensip:

> Ses, gameplay state'in sahibi değildir. Gameplay bir olay üretir, AudioSystem bunun oyuncuya nasıl duyurulacağını belirler.
