# User Profile Popup

## 1. Amaç

`UserProfilePopup`, oyuncuya ait profil bilgilerinin ve oyuncu ilerlemesinin sunulduğu UI modülüdür.

Bu dokümanın amacı:

* Profil bilgilerinin UI'da nasıl gösterileceğini tanımlamak
* Profil state'inin hangi sistem tarafından sahiplenildiğini belirlemek
* Avatar ve diğer profil asset'lerinin nasıl seçileceğini tanımlamak
* Progression verisinin UI tarafından değiştirilmesini engellemek
* Save, Economy ve Progression sistemleriyle sınırları netleştirmek
* Profil ekranının farklı oyun içerikleriyle yeniden kullanılabilir olmasını sağlamak

`UserProfilePopup` bir **presentation layer** bileşenidir.

Oyuncu verisinin sahibi değildir.

---

# 2. Temel Mimari

Profil popup'ı aşağıdaki akışla çalışmalıdır:

```text
Player / Profile Configuration
            ↓
     Profile System
            ↓
      Runtime Profile
            ↓
           Event
            ↓
   UserProfilePopup
            ↓
       Profile View
```

Progression ayrı bir sorumluluktur:

```text
ProgressionConfig
        ↓
 ProgressionSystem
        ↓
 Progression State
        ↓
      Event
        ↓
 UserProfilePopup
```

UI bu verileri yalnızca görüntüler.

---

# 3. Sorumluluklar

`UserProfilePopup` sorumlulukları:

* Profil popup'ını açmak
* Profil popup'ını kapatmak
* Oyuncu adını göstermek
* Avatarı göstermek
* Profil seviyesini veya uygun progression bilgisini göstermek
* Gerekli görsel durumları güncellemek
* Kullanıcı etkileşimlerini ilgili sisteme veya UI controller'a iletmek
* Profil state değişikliklerini UI'a yansıtmak

`UserProfilePopup` sorumlulukları dışındaki konular:

* Oyuncu verisini kalıcı olarak saklamak
* Save data'yı doğrudan değiştirmek
* Level hesaplamak
* XP hesaplamak
* Ödül hesaplamak
* Currency değiştirmek
* Avatar unlock koşullarını hesaplamak
* IAP veya reklam SDK'sına erişmek
* Analytics SDK'sını doğrudan çağırmak

---

# 4. Source of Truth

Profil ile ilgili runtime state'in tek bir sahibi olmalıdır.

Örneğin:

```text
ProfileSystem
    ↓
PlayerName
AvatarId
Profile Metadata
```

Progression için:

```text
ProgressionSystem
    ↓
CurrentProgress
Level
XP
Unlock State
```

Economy için:

```text
EconomySystem
    ↓
Currency
```

SaveSystem bu state'lerin kalıcı hale getirilmesinden sorumludur.

UI hiçbir durumda aynı state'i kendi içinde ikinci kez sahiplenmemelidir.

Yanlış:

```text
ProfileSystem.PlayerName
        ↓
UserProfilePopup.playerName
        ↓
SaveSystem
```

Burada `UserProfilePopup.playerName` ikinci bir runtime source of truth haline gelebilir.

Doğru:

```text
ProfileSystem
      ↓
Authoritative Profile State
      ↓
UserProfilePopup
      ↓
Visual Representation
```

UI'daki text yalnızca görsel representation'dır.

---

# 5. Profil Verisi

Profil sistemi oyunun ihtiyacına göre aşağıdaki gibi bir runtime state sağlayabilir:

```csharp
public sealed class PlayerProfileState
{
    public string PlayerName;
    public int AvatarId;
}
```

Bu yapı örnektir.

Her projede ihtiyaç olmayan alanlar eklenmemelidir.

Örneğin:

* `PlayerName`
* `AvatarId`
* `TitleId`
* `FrameId`

gibi alanlar yalnızca gerçekten profil konseptinin parçasıysa kullanılmalıdır.

---

# 6. Configuration ve Runtime State

Profil configuration ile runtime state birbirinden ayrılmalıdır.

Örneğin avatarlar için:

```text
AvatarConfig
    ↓
Available Avatar Definitions
    ↓
AvatarId
```

Runtime state:

```text
ProfileSystem
    ↓
Selected AvatarId
```

UI:

```text
Selected AvatarId
    ↓
AvatarConfig
    ↓
Avatar Asset
    ↓
UserProfilePopup
```

ScriptableObject static/configuration verisini temsil eder.

Oyuncunun seçtiği avatar gibi session veya player-specific state ScriptableObject üzerinde mutable runtime state olarak tutulmamalıdır.

---

# 7. Avatar Asset Akışı

Avatar asset'lerinin nereden geldiği ve nasıl yüklendiği `SYSTEMS/AssetManagement.md` tarafından belirlenir.

Profil sistemi avatar seçimini tanımlayabilir:

```text
ProfileState.AvatarId
        ↓
AvatarConfig
        ↓
Avatar Definition
        ↓
Avatar Asset
        ↓
UserProfilePopup
```

Örnek:

```text
Art/UI/Profile/Avatars/Avatar_01.png
        ↓
AvatarConfig
        ↓
AvatarId = 1
        ↓
ProfileSystem
        ↓
UserProfilePopup
```

Asset path'in doğrudan UI koduna yazılması tercih edilmez.

Yanlış:

```csharp
Resources.Load<Sprite>("Avatars/Avatar_01");
```

Doğru yaklaşım:

```text
Profile State
    ↓
Avatar Definition
    ↓
Configured Asset Reference
    ↓
UI
```

Asset'in direct reference, Addressables veya başka bir yöntemle yüklenmesi Asset Management kurallarına göre belirlenir.

---

# 8. Avatar Seçimi

Eğer oyuncu avatar değiştirebiliyorsa, UI yalnızca kullanıcı niyetini iletmelidir.

Örneğin:

```text
UserProfilePopup
        ↓
Select Avatar Intent
        ↓
ProfileSystem
        ↓
Validate Selection
        ↓
Update Profile State
        ↓
SaveSystem
```

UI doğrudan:

```csharp
playerSaveData.AvatarId = selectedAvatarId;
```

yapmamalıdır.

Aynı şekilde UI avatar'ın unlock durumunu kendi başına hesaplamamalıdır.

Örneğin:

```csharp
if (playerLevel >= 10)
{
    avatar.Unlock();
}
```

gibi gameplay/progression kuralları UI içinde bulunmamalıdır.

---

# 9. Profil Adı

Oyuncu adı değiştirilebiliyorsa:

```text
UserProfilePopup
        ↓
Name Change Intent
        ↓
ProfileSystem
        ↓
Validation
        ↓
Runtime Profile State
        ↓
SaveSystem
```

UI:

* Input'u toplar
* Kullanıcıya feedback verir
* Değişiklik isteğini iletir

ProfileSystem:

* İsim validasyonu
* State değişimi
* Gerekirse normalizasyon
* Persistence isteği

gibi sorumlulukları yönetir.

UI kendi validation kurallarını profile sisteminin yerine geçecek şekilde tanımlamamalıdır.

Basit presentation validation yapılabilir.

Örneğin:

```text
Boş input
    ↓
UI validation
```

Ancak oyun kuralı olan validation ProfileSystem tarafından yapılmalıdır.

---

# 10. Progression Bilgisi

Profil ekranında progression gösterilebilir.

Örneğin:

```text
Level
XP
Progress Bar
Rank
Title
```

Ancak progression'ın sahibi `UserProfilePopup` değildir.

Doğru:

```text
ProgressionSystem
        ↓
Progression State
        ↓
UserProfilePopup
```

Örneğin:

```csharp
profileLevelText.text = progression.Level.ToString();
profileXpBar.SetValue(progression.Progress01);
```

Buradaki değerler `ProgressionSystem` tarafından sağlanmalıdır.

UI:

```csharp
currentXp += 10;
```

gibi progression state değişiklikleri yapmamalıdır.

---

# 11. Progress Bar

Profil progression bar'ı yalnızca görsel bir representation'dır.

Örneğin:

```text
Current XP
      ↓
ProgressionSystem
      ↓
Normalized Progress
      ↓
Profile XP Bar
```

UI progression matematiğini tekrar hesaplamamalıdır.

Yanlış:

```csharp
float progress = currentXp / (float)requiredXp;
```

Eğer progression kuralları sistem tarafından zaten hesaplanıyorsa UI'a hesaplanmış değer verilmesi tercih edilir:

```csharp
float progress01 = progression.Progress01;
```

Böylece progression kuralları tek yerde kalır.

---

# 12. Profil ve Economy Ayrımı

Profil ekranında currency gösterilebilir.

Ancak currency sahibi:

```text
EconomySystem
```

olmaya devam eder.

Örneğin:

```text
EconomySystem
      ↓
Currency State
      ↓
Event
      ↓
Currency Widget
```

UserProfilePopup yalnızca currency gösterecekse mevcut state'i okuyabilir veya ilgili presentation event'lerini dinleyebilir.

Currency değiştirme işlemi UI tarafından yapılmamalıdır.

---

# 13. Profil ve SaveSystem Ayrımı

SaveSystem profil verisinin persistence katmanıdır.

Önerilen yapı:

```text
ProfileSystem
    ↓
Runtime Profile State
    ↓
SaveSystem
    ↓
PlayerSaveData
```

UI:

```text
UserProfilePopup
    ↓
ProfileSystem
```

şeklinde çalışmalıdır.

UI'ın doğrudan SaveSystem'e bağlanması tercih edilmez.

Yanlış:

```text
UserProfilePopup
    ↓
SaveSystem
    ↓
PlayerSaveData
```

Doğru:

```text
UserProfilePopup
    ↓
ProfileSystem
    ↓
Runtime State
    ↓
SaveSystem
```

Bu ayrım profil state'i ile persistence sorumluluğunu birbirinden ayırır.

---

# 14. Popup Lifecycle

Unity lifecycle kuralları uygulanmalıdır.

Önerilen yapı:

```text
Awake
    ↓
Cache References

OnEnable
    ↓
Subscribe Events
    ↓
Sync Current State

OnDisable
    ↓
Unsubscribe Events
```

Popup açıldığında yalnızca event'leri beklemek yeterli değildir.

Mevcut state de UI'a uygulanmalıdır.

Örneğin:

```text
OnEnable
    ↓
Subscribe
    ↓
RefreshCurrentProfile
```

Ardından profil değişiklikleri event üzerinden güncellenebilir.

---

# 15. Event Kullanımı

Profil state'i başka sistemler tarafından değişebiliyorsa event kullanılabilir.

Örneğin:

```text
ProfileSystem
    ↓
ProfileChanged
    ↓
UserProfilePopup
```

UI event'i aldığında:

```text
ProfileChanged
    ↓
Refresh Profile Presentation
```

yapmalıdır.

Event yalnızca gerçekten decoupled communication gerektiğinde kullanılmalıdır.

Aynı GameObject veya açık ownership ilişkisi bulunan iki component arasında sırf mimari "event-driven" olduğu için EventBus kullanılmamalıdır.

---

# 16. Event Subscription Güvenliği

Subscription:

```text
OnEnable
```

içinde yapılmalı.

Unsubscription:

```text
OnDisable
```

içinde yapılmalıdır.

Örneğin:

```csharp
private void OnEnable()
{
    _profileSystem.ProfileChanged += HandleProfileChanged;
}

private void OnDisable()
{
    _profileSystem.ProfileChanged -= HandleProfileChanged;
}
```

Popup pooled veya tekrar tekrar enable/disable olabiliyorsa bu özellikle önemlidir.

Aksi halde:

* Duplicate callback
* Memory leak
* Birden fazla UI refresh
* Destroy edilmiş object'e callback

gibi problemler oluşabilir.

---

# 17. Popup Open

Popup açılırken mevcut profile state yeniden senkronize edilmelidir.

Önerilen akış:

```text
Open
 ↓
Read Current State
 ↓
Refresh UI
 ↓
Show
```

Eğer popup zaten event subscription'ı nedeniyle güncel kalıyorsa bile açık bir `Refresh` noktası bulunması UI state'inin deterministik olmasını kolaylaştırır.

---

# 18. Popup Close

Popup kapanırken profile state değiştirilmemelidir.

Kapanma:

```text
UserProfilePopup
      ↓
Hide
```

ile sınırlı olabilir.

Deferred edit modeli kullanılıyorsa:

```text
Edit
 ↓
Temporary UI State
 ↓
Confirm
 ↓
ProfileSystem
```

şeklinde ilerlenebilir.

Cancel:

```text
Edit
 ↓
Temporary UI State
 ↓
Cancel
 ↓
Discard Temporary State
```

şeklinde çalışabilir.

Immediate apply modelinde ise Cancel gerekmeyebilir.

---

# 19. Profil Düzenleme Modeli

Profil ekranı yalnızca read-only olabilir.

Bu durumda:

```text
ProfileSystem
      ↓
UserProfilePopup
```

yeterlidir.

Düzenleme varsa:

```text
User Input
    ↓
Edit Intent
    ↓
ProfileSystem
    ↓
Validation
    ↓
State Change
    ↓
Persistence
```

kullanılmalıdır.

UI'ın doğrudan runtime state üzerinde mutation yapması tercih edilmez.

---

# 20. Analytics

Profil UI interaction'ları analytics gerektiriyorsa event UI içinde doğrudan analytics SDK'sına gönderilmemelidir.

Yanlış:

```csharp
AnalyticsSDK.LogEvent("profile_open");
```

Doğru yaklaşım:

```text
User Intent
    ↓
Profile / Relevant System
    ↓
AnalyticsSystem
    ↓
Analytics SDK
```

Analytics event'inin hangi katmandan üretileceği olayın anlamına göre belirlenmelidir.

Örneğin "profile opened" presentation event'i olabilir.

"avatar changed" ise profile state değişikliğidir.

Analytics'in sahibi:

```text
AnalyticsSystem
```

olmaya devam eder.

---

# 21. UI Hierarchy

Örnek UI yapısı:

```text
UserProfilePopup
├── Background
├── Header
│   ├── Title
│   └── CloseButton
│
├── ProfileSection
│   ├── Avatar
│   ├── PlayerName
│   └── OptionalTitle
│
├── ProgressionSection
│   ├── LevelLabel
│   ├── ProgressBar
│   └── ProgressText
│
└── Actions
    ├── EditProfileButton
    └── OptionalAction
```

Bu hierarchy örnektir.

Her oyun tüm alanları kullanmak zorunda değildir.

---

# 22. UI Component Sınırları

`UserProfilePopup` tek bir dev component haline getirilmemelidir.

Gerekirse presentation component'lerine ayrılabilir:

```text
UserProfilePopup
├── ProfileHeaderView
├── AvatarView
├── ProgressionView
└── ProfileActionView
```

Ancak bu ayrım yalnızca gerçek karmaşıklık varsa yapılmalıdır.

Sırf component sayısını artırmak için abstraction eklenmemelidir.

---

# 23. Navigation

Profil popup'ının açılması UI navigation sorumluluğudur.

Örneğin:

```text
TopBarHeader
      ↓
Profile Button
      ↓
UserProfilePopup
```

Burada TopBarHeader profil state'inin sahibi değildir.

Benzer şekilde:

```text
BottomNavigationBar
      ↓
Profile Intent
      ↓
Navigation Controller
      ↓
UserProfilePopup
```

kullanılabilir.

Profil popup'ının açılması global `GameFlow` state değişimi anlamına gelmiyorsa GameFlow'a bağlanmamalıdır.

---

# 24. Asset Management

Profil asset'leri için temel ayrım:

```text
Asset
  ↓
Configuration
  ↓
Profile System / Feature
  ↓
UserProfilePopup
```

Asset'in yüklenme ve release stratejisi:

```text
AssetManagement.md
```

tarafından belirlenir.

Profil dokümanı:

* Hangi asset'in gerektiğini
* Asset'in ne amaçla kullanıldığını
* Asset'in hangi configuration üzerinden seçildiğini

tanımlayabilir.

Ancak global asset loading mimarisini yeniden tanımlamamalıdır.

---

# 25. Performance

Profil popup'ı genellikle hot path değildir.

Buna rağmen:

* Açılışta gereksiz allocation yapılmamalı
* Gereksiz `GetComponent` çağrıları tekrarlanmamalı
* Her frame state polling yapılmamalı
* `Update()` yalnızca gerçek ihtiyaç varsa kullanılmalı
* Profil değişiklikleri event-driven olmalı
* Avatar asset'leri gereksiz yere tekrar yüklenmemeli

Özellikle:

```csharp
Update()
{
    RefreshProfile();
}
```

gibi sürekli polling yapılmamalıdır.

---

# 26. Animation

Popup animasyonları presentation layer'a aittir.

Örneğin:

```text
Open
 ↓
Show Animation
 ↓
Visible
```

ve:

```text
Close
 ↓
Hide Animation
 ↓
Disabled
```

Animasyon sistemi ile gameplay state birbirine bağlanmamalıdır.

Profil state değişikliğinin başarılı olması yalnızca bir tween'in tamamlanmasına bağlı olmamalıdır.

Tween kullanılıyorsa popup disable/destroy/pool durumunda ilgili tween'ler temizlenmeli veya kill edilmelidir.

---

# 27. Generic Template Rule

`UserProfilePopup` belirli bir oyunun özel profil modeline bağımlı olmamalıdır.

Örneğin template core içinde:

```text
KingdomLevel
StarCount
BuildingCount
DecorationCount
```

gibi belirli bir oyuna ait kavramlar bulunmamalıdır.

Bunun yerine:

```text
PlayerName
Avatar
Level
Progress
Title
```

gibi genel profil kavramları kullanılabilir.

Oyuna özel bilgiler gerekiyorsa feature layer'a eklenmelidir.

---

# 28. Definition of Done

`UserProfilePopup` tamamlanmış sayılmadan önce:

* [ ] UI yalnızca presentation sorumluluğunda mı?
* [ ] Profil runtime state'inin tek sahibi belli mi?
* [ ] Progression state'inin sahibi `ProgressionSystem` mı?
* [ ] Economy state UI dışında mı?
* [ ] UI doğrudan `PlayerSaveData` değiştirmiyor mu?
* [ ] UI doğrudan SDK çağırmıyor mu?
* [ ] Avatar seçimi configuration üzerinden mi yapılıyor?
* [ ] Asset loading/lifetime `AssetManagement.md` ile uyumlu mu?
* [ ] Event subscription `OnEnable` içinde mi?
* [ ] Event unsubscription `OnDisable` içinde mi?
* [ ] Popup açılırken mevcut state sync ediliyor mu?
* [ ] Gereksiz `Update()` polling kullanılmıyor mu?
* [ ] Profil değişikliği system üzerinden mi ilerliyor?
* [ ] UI progression hesabı yapmıyor mu?
* [ ] Tween lifecycle güvenli mi?
* [ ] Gereksiz abstraction eklenmemiş mi?
* [ ] Generic template sınırları korunuyor mu?
* [ ] Serialization açısından mevcut veriler korunuyor mu?

---

# 29. Source of Truth

Profil mimarisinde temel sorumluluk dağılımı:

```text
ProfileConfig
    ↓
Static Profile Configuration

ProfileSystem
    ↓
Runtime Profile State

ProgressionSystem
    ↓
Progression Runtime State

EconomySystem
    ↓
Currency Runtime State

SaveSystem
    ↓
Persistent Player Data

AssetManagement
    ↓
Asset Loading / Lifetime

UserProfilePopup
    ↓
Profile Presentation
```

Temel kural:

> `UserProfilePopup` oyuncunun profilini **sahiplenmez**, oyuncunun profilini **gösterir**.
