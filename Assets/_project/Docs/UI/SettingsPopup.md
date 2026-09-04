# Settings Popup UI

## 1. Amaç

Bu doküman, oyun içerisindeki Settings Popup UI yapısını tanımlar.

Settings Popup, oyuncunun oyun ayarlarını görüntülemesini ve değiştirmesini sağlayan presentation katmanıdır.

Örnek ayarlar:

```text id="x7m2p4"
Settings
├── Music
├── Sound Effects
├── Notifications
├── Vibration
├── Language
└── Privacy
```

Gerçek ayarlar oyuna göre değişebilir.

Template, belirli bir oyunun ayarlarını zorunlu hale getirmemelidir.

---

# 2. Sorumluluk

SettingsPopup'ın sorumlulukları:

* Mevcut ayarları göstermek
* Kullanıcı input'unu almak
* Ayar değişikliği isteğini ilgili owner'a iletmek
* Visual state'i güncellemek
* Popup lifecycle'ını yönetmek

SettingsPopup'ın sorumluluğu olmayan şeyler:

* Settings state'inin authoritative owner'ı olmak
* Save data'yı doğrudan yönetmek
* Audio mixer state'ini doğrudan yönetmek
* Analytics SDK çağırmak
* Localization system'i yönetmek
* Player profile state'ini yönetmek
* GameFlow state'i değiştirmek

Temel prensip:

```text id="q9n4v6"
Settings UI
    ↓
User Intent
    ↓
Settings Owner
    ↓
Runtime State
    ↓
Persistence / Relevant System
```

---

# 3. Settings State Ownership

Settings'in runtime state'i UI içerisinde tutulmamalıdır.

Örneğin:

```text id="p5r8m2"
SettingsSystem
├── MusicEnabled
├── SfxEnabled
├── VibrationEnabled
└── NotificationsEnabled
```

SettingsPopup:

```text id="c7x3n9"
SettingsSystem
    ↓
Current State
    ↓
SettingsPopup
    ↓
Presentation
```

UI yalnızca state'in görsel temsilini yapar.

---

# 4. Settings Configuration ve Runtime State

Configuration ile runtime state birbirinden ayrılmalıdır.

Örneğin:

```text id="m4q7k1"
SettingsConfig
    ↓
Static Configuration

SettingsSystem
    ↓
Runtime State

SettingsPopup
    ↓
Presentation
```

`SettingsConfig` içerisinde designer tarafından belirlenen değerler bulunabilir.

Örneğin:

```csharp id="r8n2v5"
[SerializeField] private bool vibrationOptionVisible;
```

Ancak oyuncunun mevcut tercihi:

```csharp id="k3p9m6"
private bool vibrationEnabled;
```

gibi runtime state olarak `SettingsSystem` tarafından yönetilmelidir.

ScriptableObject runtime session state'i tutmamalıdır.

---

# 5. Save Data İlişkisi

Settings persistence gerekiyorsa SaveSystem kullanılabilir.

Önerilen akış:

```text id="w6m1q8"
SettingsPopup
    ↓
Settings Change Request
    ↓
SettingsSystem
    ↓
Runtime Settings State
    ↓
SaveSystem
    ↓
Persistent Save Data
```

SettingsPopup doğrudan `PlayerSaveData` üzerinde değişiklik yapmamalıdır.

Yanlış:

```text id="z4p7n2"
SettingsPopup
    ↓
PlayerSaveData.MusicEnabled = false
```

Doğru:

```text id="s8q3m5"
SettingsPopup
    ↓
SettingsSystem.SetMusicEnabled(false)
    ↓
SaveSystem
```

SaveSystem'in detayları:

```text id="d5x9k1"
SYSTEMS/SaveSystem.md
```

içerisinde tanımlanır.

---

# 6. Audio Settings

Music veya SFX ayarı varsa SettingsPopup audio sistemini doğrudan yönetmemelidir.

Önerilen:

```text id="n2v6r8"
SettingsPopup
    ↓
SettingsSystem
    ↓
AudioSystem
```

Örneğin:

```text id="q4m7p1"
Music Toggle
    ↓
SettingsSystem.SetMusicEnabled(...)
    ↓
AudioSystem
```

AudioSystem'in görevi:

```text id="x8k3n5"
Music
SFX
Volume
Audio Playback
```

gibi audio davranışlarını yönetmektir.

Detay:

```text id="j6r2m9"
SYSTEMS/Audio.md
```

---

# 7. Toggle Lifecycle

Bir toggle mevcut runtime state ile initialize edilmelidir.

Örneğin:

```text id="p9m4x7"
SettingsPopup.Show()
        ↓
Read Current Settings
        ↓
MusicToggle = Current Music State
        ↓
SFXToggle = Current SFX State
```

UI kendi default değerini source of truth olarak kullanmamalıdır.

Yanlış:

```text id="v3q8n1"
MusicToggle = true
```

Doğru:

```text id="b7m2k5"
MusicToggle = SettingsSystem.MusicEnabled
```

---

# 8. User Change

Kullanıcı toggle değiştirdiğinde:

```text id="c5n9r2"
User Input
    ↓
Toggle Changed
    ↓
SettingsSystem.Set(...)
```

UI:

```text id="m8q4x6"
"Music is now disabled."
```

gibi presentation güncellemelerini yapabilir.

Ancak state'in gerçek owner'ı `SettingsSystem` olmaya devam eder.

---

# 9. Event-Based Synchronization

Birden fazla sistem aynı settings state'ini kullanıyorsa event-based synchronization tercih edilebilir.

Örneğin:

```text id="r6k2p8"
SettingsSystem
    ↓
SettingsChanged
    ├── AudioSystem
    ├── HapticSystem
    └── SettingsPopup
```

Bu sayede sistemler birbirlerinin implementation detaylarına doğrudan bağlanmak zorunda kalmaz.

Ancak tek bir local relationship için EventBus kullanılmamalıdır.

---

# 10. Audio Example

Music kapatıldığında:

```text id="h4n8q2"
Player
    ↓
Music Toggle
    ↓
SettingsSystem
    ↓
MusicEnabled = false
    ↓
SettingsChanged
    ├── AudioSystem
    └── SettingsPopup
```

AudioSystem:

```text id="w7m3p9"
MusicEnabled = false
    ↓
Stop / Mute / Adjust Music
```

uygulamasını kendi sorumluluğunda gerçekleştirir.

SettingsPopup AudioSystem implementation detaylarını bilmemelidir.

---

# 11. Vibration Settings

Vibration veya haptic ayarı varsa:

```text id="q3r8m1"
SettingsPopup
    ↓
SettingsSystem
    ↓
HapticSystem
```

akışı kullanılabilir.

Gameplay veya UI code'u:

```text id="n5x7k4"
if (Settings...)
```

ile her yerde kendi vibration kararını vermemelidir.

İlgili haptic system veya merkezi settings state bu kararı sağlayabilir.

---

# 12. Notification Settings

Notification ayarı varsa:

```text id="m9k2v6"
SettingsPopup
    ↓
SettingsSystem
    ↓
NotificationSystem
```

şeklinde bağlanabilir.

Notification SDK çağrıları SettingsPopup içerisinde yapılmamalıdır.

---

# 13. Language Settings

Language seçimi varsa:

```text id="p4r7n3"
SettingsPopup
    ↓
SettingsSystem
    ↓
LocalizationSystem
```

akışı kullanılabilir.

SettingsPopup:

```text id="x8m1q5"
Selected Language
```

değerini gösterir.

Localization'ın uygulanması `LocalizationSystem` sorumluluğudur.

---

# 14. Immediate vs Deferred Apply

Ayar değişiklikleri iki şekilde uygulanabilir.

## Immediate Apply

Kullanıcı değiştirdiği anda uygulanır:

```text id="z6q3m8"
Toggle
    ↓
SettingsSystem
    ↓
Apply Immediately
```

Örneğin:

* Music
* SFX
* Vibration

---

## Deferred Apply

Değişiklikler geçici olarak UI'da tutulur ve `Apply` ile uygulanır:

```text id="c2n7p5"
User Changes
    ↓
Pending Settings
    ↓
Apply
    ↓
SettingsSystem
```

Bu model özellikle birden fazla ayarın birlikte uygulanması gerektiğinde kullanılabilir.

Hangi modelin kullanılacağı feature requirement'a göre belirlenmelidir.

---

# 15. Cancel

Deferred settings kullanılıyorsa Cancel davranışı:

```text id="v8m4r2"
Current Settings
        ↓
Temporary UI State
        ↓
Cancel
        ↓
Discard Temporary State
        ↓
Restore UI
```

şeklinde çalışabilir.

Cancel runtime settings'i değiştirmemelidir.

---

# 16. Reset Settings

Reset butonu varsa:

```text id="k5q9m3"
Reset
    ↓
SettingsSystem
    ↓
Default Settings
```

kullanılabilir.

Default değerlerin sahibi açık olmalıdır.

Örneğin:

```text id="p7x2n6"
SettingsConfig
    ↓
Default Values
```

ve:

```text id="m3r8q4"
SettingsSystem
    ↓
Runtime Values
```

---

# 17. Default Values

Default settings configuration'dan gelebilir.

Örneğin:

```csharp id="q6n1v9"
[CreateAssetMenu]
public class SettingsConfig : ScriptableObject
{
    [SerializeField] private bool musicEnabled = true;
    [SerializeField] private bool sfxEnabled = true;
    [SerializeField] private bool vibrationEnabled = true;
}
```

Bunlar başlangıç/default değerleridir.

Oyuncunun mevcut tercihleri:

```text id="x4m7p2"
SettingsSystem
```

tarafından yönetilir.

---

# 18. First Launch

İlk açılışta:

```text id="r8k3n5"
Bootstrap
    ↓
SaveSystem
    ↓
No Existing Settings
    ↓
Settings Defaults
    ↓
SettingsSystem
    ↓
Runtime Settings Ready
```

ardından SettingsPopup mevcut state'i gösterir.

SettingsPopup first-launch initialization yapmamalıdır.

---

# 19. Save Migration

Settings save data formatı değişirse migration `SaveSystem` sorumluluğunda olmalıdır.

Örneğin:

```text id="n6q2m8"
Old Save
    ↓
Migration
    ↓
New Settings Data
    ↓
SettingsSystem
```

SettingsPopup eski ve yeni save formatlarını bilmemelidir.

---

# 20. Popup Lifecycle

Önerilen lifecycle:

```text id="p3r7x9"
Open Request
    ↓
Show
    ↓
Sync Current State
    ↓
User Interaction
    ↓
Hide
    ↓
Cleanup
```

`Show()` sırasında mevcut state UI'ya synchronize edilmelidir.

---

# 21. Event Subscription

SettingsPopup event dinliyorsa:

```text id="m5k8q1"
OnEnable
    ↓
Subscribe

OnDisable
    ↓
Unsubscribe
```

uygulanmalıdır.

Popup kapatıldığında event subscription açık bırakılmamalıdır.

Özellikle popup tekrar tekrar açılıp kapanıyorsa duplicate subscription oluşmamalıdır.

---

# 22. Animation

Popup açılış/kapanış animasyonları presentation sorumluluğundadır.

Örneğin:

```text id="x2n6v8"
Open
    ↓
Scale / Fade
    ↓
Visible
```

Kapanış:

```text id="q9m3r5"
Hide Request
    ↓
Animation
    ↓
Disable
```

Tween lifecycle mevcut proje convention'ına uygun şekilde temizlenmelidir.

Popup pooling kullanıyorsa animation state tamamen resetlenmelidir.

---

# 23. Close Button

Close button yalnızca popup'ın kapanmasını istemelidir.

Örneğin:

```text id="v4k7p2"
Close Button
    ↓
SettingsPopup.Hide()
```

Global GameFlow değiştirilmemelidir.

---

# 24. Asset Integration

Settings UI asset'leri configuration üzerinden bağlanabilir.

Örneğin:

```text id="j8m2q6"
Art/UI/Settings/
├── SettingsIcon.png
├── MusicIcon.png
├── SfxIcon.png
└── VibrationIcon.png
```

↓

```text id="c5r9n3"
SettingsConfig
├── SettingsIcon
├── MusicIcon
├── SfxIcon
└── VibrationIcon
```

↓

```text id="n7x4p1"
SettingsPopup
```

Asset loading strategy:

```text id="m3q8k5"
SYSTEMS/AssetManagement.md
```

kurallarına göre uygulanır.

---

# 25. Accessibility / UX

Settings UI mümkün olduğunca açık ve anlaşılır olmalıdır.

Örneğin toggle state:

```text id="r6p2m9"
Music
[ ON ]
```

veya:

```text id="k8n4q3"
Music
[ OFF ]
```

şeklinde açıkça gösterilebilir.

Kullanıcı yalnızca icon veya renk farkına bağımlı bırakılmamalıdır.

---

# 26. Platform-Specific Settings

Bazı ayarlar platforma göre farklı davranabilir.

Örneğin:

```text id="x5m7q2"
Haptic Feedback
```

mobilde anlamlı olabilirken başka platformlarda farklı davranabilir.

UI platform-specific implementation detaylarını sahiplenmemelidir.

Örneğin:

```text id="p9r3k6"
SettingsPopup
    ↓
SettingsSystem
    ↓
Platform-specific implementation
```

kullanılmalıdır.

---

# 27. Analytics

Settings değişiklikleri analytics'e gönderilecekse SettingsPopup doğrudan analytics SDK çağırmamalıdır.

Önerilen:

```text id="m4x8q1"
SettingsSystem
    ↓
Setting Changed
    ↓
AnalyticsSystem
```

Analytics SDK ownership:

```text id="j7n2p5"
SYSTEMS/Analytics.md
```

tarafındadır.

UI event'i doğrudan analytics SDK'ya bağlanmamalıdır.

---

# 28. Monetization

SettingsPopup monetization logic'i içermemelidir.

Örneğin:

```text id="q3m9v6"
Remove Ads
Restore Purchases
Premium Settings
```

gibi seçenekler varsa UI yalnızca ilgili flow'u başlatır.

Purchase logic:

```text id="x6k2r8"
SYSTEMS/Monetization.md
```

tarafından yönetilir.

---

# 29. Generic Template Rule

Settings sistemi generic olmalıdır.

Template core'unda aşağıdaki gibi oyun-specific settings bulunmamalıdır:

```text id="n8p4m2"
Kingdom Music
Star Effects
Building Animation
Decoration Mode
```

Bunun yerine generic settings kullanılmalıdır:

```text id="r5q7k3"
Music
SFX
Vibration
Notifications
Language
```

Oyunun ihtiyaçlarına göre yeni settings eklenebilir.

---

# 30. Dependency Direction

Tercih edilen dependency yönü:

```text id="v2m8q5"
SettingsPopup
    ↓
Settings API / Command
    ↓
SettingsSystem
    ↓
Relevant Systems
    ↓
SaveSystem
```

Kaçınılması gereken:

```text id="c9r3m7"
SettingsPopup
    ↕
AudioSystem
    ↕
SaveSystem
    ↕
AnalyticsSystem
```

UI bütün sistemlerin coordinator'ı haline gelmemelidir.

---

# 31. Source of Truth

Settings mimarisinde:

```text id="p7x4n2"
Default Values
    → SettingsConfig

Runtime Settings
    → SettingsSystem

Persistence
    → SaveSystem

Audio Behavior
    → AudioSystem

Haptic Behavior
    → Haptic System

Localization
    → LocalizationSystem

Analytics
    → AnalyticsSystem

Visual Presentation
    → SettingsPopup
```

Her responsibility'nin authoritative owner'ı açık olmalıdır.

---

# 32. Definition of Done

SettingsPopup tamamlanmadan önce:

* UI yalnızca presentation ve user intent sorumluluğunda mı?
* Runtime settings state'in owner'ı belli mi?
* Default values configuration'dan mı geliyor?
* ScriptableObject runtime state tutuyor mu?
* Save data UI tarafından doğrudan değiştiriliyor mu?
* Audio logic UI'ya girmiş mi?
* Haptic logic UI'ya girmiş mi?
* Localization logic UI'ya girmiş mi?
* Analytics SDK doğrudan UI'dan çağrılıyor mu?
* Monetization logic UI'ya girmiş mi?
* Event subscription lifecycle-safe mi?
* Popup tekrar açıldığında duplicate subscription oluşuyor mu?
* Initial state synchronize ediliyor mu?
* Animation state cleanup ediliyor mu?
* Asset'ler configuration üzerinden mi bağlanıyor?
* Hard-coded asset path var mı?
* Generic template prensibi korunuyor mu?
* Serialization güvenli mi?

Bu soruların cevabı net olmalıdır.
