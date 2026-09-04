# Top Bar Header UI

## 1. Amaç

Bu doküman, oyun içerisindeki üst bar/header UI yapısını tanımlar.

Top Bar Header; oyuncunun sık kullanılan global bilgilerini ve aksiyonlarını göstermesi için kullanılan presentation katmanıdır.

Örnek içerikler:

```text
┌─────────────────────────────────────┐
│ Profile   ❤️ 5   🪙 120   💎 35  ⚙ │
└─────────────────────────────────────┘
```

Gerçek içerik oyuna göre değişebilir.

TopBarHeader belirli bir oyunun ekonomi, can, yıldız veya ilerleme sistemine bağımlı olmamalıdır.

---

# 2. Sorumluluk

TopBarHeader'ın temel sorumlulukları:

* Global runtime bilgilerini göstermek
* İlgili system event'lerini dinlemek
* UI değerlerini güncellemek
* UI asset'lerini configuration üzerinden kullanmak
* Kullanıcı aksiyonlarını ilgili command veya system'e iletmek

TopBarHeader'ın sorumluluğu olmayan şeyler:

* Currency hesaplamak
* Reward hesaplamak
* Currency vermek
* Currency harcamak
* Player save data değiştirmek
* Gameplay state yönetmek
* Economy kurallarını uygulamak
* Progression kurallarını uygulamak

Temel prensip:

```text id="7r2w8m"
System
    ↓
Authoritative Runtime State

UI
    ↓
Presentation
```

---

# 3. Genel Mimari

Önerilen yapı:

```text id="r7c5yn"
EconomySystem
      ↓
Currency Changed Event
      ↓
TopBarHeader
      ↓
Update Currency Text
```

Asset tarafında:

```text id="z2m3kd"
CurrencyConfig
      ↓
Currency Icon
      ↓
TopBarHeader
```

Birlikte:

```text id="b4q1s8"
CurrencyConfig
      ↓
EconomySystem
      ↓
Currency State
      ↓
Event
      ↓
TopBarHeader
      ↓
Presentation
```

---

# 4. Configuration

TopBarHeader'da kullanılacak görsel asset'ler configuration üzerinden tanımlanabilir.

Örneğin:

```csharp id="0s6y3k"
[CreateAssetMenu(
    fileName = "CurrencyConfig",
    menuName = "Project/Config/Currency Config")]
public class CurrencyConfig : ScriptableObject
{
    [SerializeField] private Sprite icon;

    public Sprite Icon => icon;
}
```

Bu configuration'ın amacı:

```text id="w4f3k0"
"Hangi currency icon'u kullanılacak?"
```

sorusunu cevaplamaktır.

Currency'nin runtime miktarı burada tutulmaz.

Yanlış:

```csharp id="5e2n8a"
[SerializeField] private int amount;
```

Doğru:

```text id="1h7x4z"
CurrencyConfig
    → Static configuration

EconomySystem
    → Runtime currency amount
```

---

# 5. Currency Ownership

Currency miktarının authoritative owner'ı `EconomySystem` olmalıdır.

Örneğin:

```text id="m9w6k2"
EconomySystem
    ├── Coins
    ├── Gems
    └── Other Currency
```

TopBarHeader sadece görüntüler:

```text id="n8q4c6"
EconomySystem.Coins = 120
        ↓
TopBarHeader
        ↓
"120"
```

UI içerisinde:

```csharp id="f7j2p1"
coins += 10;
```

gibi runtime economy manipulation yapılmamalıdır.

---

# 6. Currency Update

Currency değiştiğinde UI yeni değeri göstermelidir.

Tercih edilen iletişim:

```text id="u6v9s2"
EconomySystem
      ↓
CurrencyChanged
      ↓
TopBarHeader
```

Örneğin:

```csharp id="q3r8t1"
private void OnCurrencyChanged(CurrencyChangedEvent data)
{
    UpdateCurrency(data.CurrencyType, data.Amount);
}
```

Event sistemi için:

```text id="j4m8x0"
SYSTEMS/EventSystem.md
```

kuralları uygulanır.

Event subscription:

```text id="e8p1v5"
OnEnable
    ↓
Subscribe

OnDisable
    ↓
Unsubscribe
```

---

# 7. Direct Reference vs Event

TopBarHeader her bilgi için EventBus kullanmak zorunda değildir.

Örneğin UI doğrudan sahibi olduğu bir local component ile iletişim kuruyorsa direct reference kullanılabilir.

Event tercih edilebilir:

```text id="p4x9s6"
EconomySystem
      ↓
Global Currency Change
      ↓
TopBarHeader
```

Direct reference tercih edilebilir:

```text id="c5d7m2"
TopBarHeader
      ↓
Owned Local Presenter
```

Karar:

```text id="h2k6q8"
Cross-system / decoupled
    → Event

Explicit local ownership
    → Direct Reference
```

Her iletişimi EventBus'a dönüştürmek gerekli değildir.

---

# 8. TopBarHeader Components

TopBarHeader aşağıdaki gibi parçalara ayrılabilir:

```text id="q1v6r9"
TopBarHeader
├── ProfileButton
├── CurrencyWidget
├── ProgressWidget
├── LivesWidget
├── SettingsButton
└── ...
```

Bunların tamamının kullanılması zorunlu değildir.

Oyun gereksinimine göre kullanılacak component'ler seçilir.

Template, kullanılmayan UI sistemlerini zorunlu hale getirmemelidir.

---

# 9. CurrencyWidget

CurrencyWidget yalnızca currency presentation'ından sorumludur.

Örnek:

```text id="z5w3n8"
CurrencyWidget
├── Icon
├── AmountText
└── Optional Animation
```

CurrencyWidget:

* Amount hesaplamaz
* Purchase gerçekleştirmez
* Reward vermez
* EconomySystem state'ini değiştirmez

Örneğin kullanıcı currency icon'una tıklarsa:

```text id="d8r4m1"
CurrencyWidget
      ↓
Open Shop / Currency Purchase Command
      ↓
Relevant System
```

UI kendi başına satın alma işlemi gerçekleştirmez.

---

# 10. Currency Animation

Currency değiştiğinde görsel feedback gösterilebilir.

Örneğin:

```text id="e5n9q3"
120
 ↓
+20
 ↓
140
```

Animation presentation katmanında yapılmalıdır.

Ancak animation tamamlanmadan economy state güncellenmiş kabul edilmemelidir.

Doğru:

```text id="s7k1p4"
EconomySystem
    ↓
Amount = 140
    ↓
UI Animation
    ↓
Visual = 140
```

Yanlış:

```text id="a2m8v6"
UI Animation
    ↓
Animation Completed
    ↓
EconomySystem.Amount = 140
```

Gameplay/system state animation'a bağımlı olmamalıdır.

---

# 11. Number Formatting

Currency miktarının nasıl gösterileceği presentation concern'dür.

Örneğin:

```text id="t6h2q9"
1200
1.2K
1,200
```

gibi formatlar UI tarafından veya ayrı bir presentation utility tarafından uygulanabilir.

Ancak gerçek currency amount değiştirilmemelidir.

Örneğin:

```text id="g9c4r2"
Runtime Amount = 1200
Displayed Amount = "1.2K"
```

---

# 12. Player Profile

Profile alanı varsa TopBarHeader oyuncu profilinin presentation'ını gösterir.

Örneğin:

```text id="n3x7p5"
PlayerProfile
├── Avatar
├── Name
└── Level
```

Player data'nın authoritative owner'ı ilgili Player/Profile system olmalıdır.

TopBarHeader profile state'i sahiplenmez.

---

# 13. Progress

Oyunun global progression bilgisi header'da gösterilebilir.

Örneğin:

```text id="w6r2k9"
Level 12
Progress 65%
```

Bu durumda:

```text id="j7m4s1"
ProgressionSystem
      ↓
Progress Updated
      ↓
TopBarHeader
```

TopBarHeader progress hesaplamamalıdır.

Örneğin aşağıdaki yaklaşım kullanılmamalıdır:

```csharp id="r8v2d5"
currentXp / requiredXp
```

eğer bu hesaplama progression domain'ine aitse.

Bu hesaplama `ProgressionSystem` veya ilgili domain modelinin sorumluluğunda olmalıdır.

UI yalnızca sonucu gösterir.

---

# 14. Settings Button

Settings button UI aksiyonunu ilgili UI flow'a iletir.

Örneğin:

```text id="q6n1x4"
SettingsButton
      ↓
Open Settings
      ↓
SettingsPopup
```

TopBarHeader, SettingsPopup'ın içindeki ayarların sahibi değildir.

Settings popup:

```text id="b8k3m7"
UI/SettingsPopup.md
```

dokümanında tanımlanır.

---

# 15. Shop Button

Currency veya shop button bulunuyorsa:

```text id="m2r7q5"
TopBarHeader
      ↓
Open Shop
      ↓
ShopView
```

kullanılabilir.

TopBarHeader:

* Product listesi oluşturmaz
* Product price hesaplamaz
* IAP çağrısı yapmaz
* Ad göstermez
* Purchase gerçekleştirmez

Shop UI:

```text id="x8p4c1"
UI/ShopView.md
```

Monetization:

```text id="v3k9m6"
SYSTEMS/Monetization.md
```

tarafından tanımlanır.

---

# 16. Asset Integration

TopBarHeader asset'leri feature configuration üzerinden bağlanmalıdır.

Örneğin:

```text id="c4m8y2"
Art/UI/Currency/CoinIcon.png
        ↓
CurrencyConfig.Icon
        ↓
CurrencyWidget
        ↓
TopBarHeader
```

UI prefab'ı doğrudan asset path bilmemelidir.

Örneğin:

```csharp id="r1z5k7"
Resources.Load<Sprite>("UI/CoinIcon");
```

kullanımı tercih edilmez.

---

# 17. Prefab Integration

Önerilen yapı:

```text id="d7q2m9"
Prefabs/
└── UI/
    └── TopBarHeader.prefab
```

Prefab içerisinde presentation component'leri bulunabilir:

```text id="k5w8n3"
TopBarHeader
├── CurrencyWidget
├── ProfileWidget
├── SettingsButton
└── ...
```

Configuration asset'leri Inspector üzerinden bağlanabilir.

Örneğin:

```text id="s4j9p2"
TopBarHeader
    ↓
CurrencyWidget
    ↓
CurrencyConfig
```

---

# 18. Scene Integration

TopBarHeader global UI ise ilgili persistent UI root içerisinde bulunabilir.

Örneğin:

```text id="y6m1q8"
Persistent UI
└── TopBarHeader
```

veya:

```text id="p3x7r4"
MainMenu Scene
└── TopBarHeader
```

hangi yapı kullanılacaksa proje architecture'ı belirlemelidir.

TopBarHeader'ın birden fazla scene'de duplicate edilmesine dikkat edilmelidir.

---

# 19. Lifecycle

TopBarHeader lifecycle'ı:

```text id="c8n2v5"
Awake
    ↓
Cache Local References

OnEnable
    ↓
Subscribe

Start
    ↓
Initial Presentation Sync

OnDisable
    ↓
Unsubscribe

OnDestroy
    ↓
Cleanup if Required
```

Event subscription `Awake` veya `Start` içerisinde rastgele yapılmamalıdır.

Projenin lifecycle convention'ı korunmalıdır.

---

# 20. Initial State

UI yalnızca event bekleyerek initial state'i öğrenmemelidir.

Örneğin:

```text id="g2r6m9"
TopBarHeader Enabled
        ↓
No CurrencyChanged Event
        ↓
UI = "0"
```

gibi bir durum oluşabilir.

Initial synchronization yapılmalıdır.

Örneğin:

```text id="q9v4k1"
TopBarHeader
      ↓
Read Current Economy State
      ↓
Display Current Value
```

Ardından event'ler gelecekteki değişiklikleri günceller.

---

# 21. Save System İlişkisi

TopBarHeader save data'yı doğrudan okumamalıdır.

Yanlış:

```text id="h7m3p9"
TopBarHeader
    ↓
PlayerSaveData
    ↓
Coins
```

Doğru:

```text id="x4n8q2"
SaveSystem
    ↓
Load Save Data
    ↓
EconomySystem
    ↓
Runtime Currency State
    ↓
TopBarHeader
```

Save data persistence layer'ıdır.

EconomySystem runtime authority'dir.

TopBarHeader presentation layer'ıdır.

---

# 22. Monetization İlişkisi

TopBarHeader ile MonetizationSystem arasında doğrudan purchase logic bulunmamalıdır.

Örneğin:

```text id="m8q5r3"
TopBarHeader
      ↓
Open Shop
      ↓
ShopView
      ↓
Purchase Request
      ↓
MonetizationSystem
      ↓
EconomySystem
```

UI purchase sonucunu gösterebilir ancak transaction'ın sahibi değildir.

---

# 23. Generic Template Rule

TopBarHeader aşağıdaki domain'lere doğrudan bağımlı hale getirilmemelidir:

```text id="z7m2x4"
Kingdom
Stars
Buildings
Decorations
Specific Puzzle Board
Specific Character
```

Bunların yerine generic concepts kullanılmalıdır:

```text id="q1k8n5"
Currency
Progression
Profile
Resource
Notification
Settings
```

Oyunun içeriği değiştiğinde TopBarHeader mimarisi mümkün olduğunca korunmalıdır.

---

# 24. Communication Example

Tam akış:

```text id="s9r3m6"
Player Action
    ↓
Gameplay / System
    ↓
RewardSystem
    ↓
EconomySystem
    ↓
Currency State Changed
    ↓
Event
    ↓
TopBarHeader
    ↓
CurrencyWidget
    ↓
Visual Update
```

Burada UI reward veya currency hesabına katılmaz.

---

# 25. Source of Truth

TopBarHeader için:

```text id="p6v2x8"
Currency amount
    → EconomySystem

Progress
    → ProgressionSystem

Player profile
    → Player/Profile System

Purchase
    → MonetizationSystem

Persistence
    → SaveSystem

Visual asset selection
    → Configuration

Visual presentation
    → TopBarHeader
```

Her state'in tek bir authoritative owner'ı olmalıdır.

---

# 26. Definition of Done

TopBarHeader tamamlanmadan önce:

* UI yalnızca presentation sorumluluğunda mı?
* Currency state EconomySystem tarafından mı yönetiliyor?
* Progress state doğru system tarafından mı yönetiliyor?
* UI runtime state'i sahipleniyor mu?
* Asset'ler configuration üzerinden mi bağlanıyor?
* Hard-coded asset path var mı?
* Event subscription lifecycle-safe mi?
* Initial state synchronization yapılıyor mu?
* EventBus gereksiz yere kullanılmış mı?
* UI doğrudan SaveSystem'e erişiyor mu?
* UI MonetizationSystem'in transaction logic'ini sahipleniyor mu?
* Animation gameplay state'ine bağımlı mı?
* Duplicate TopBarHeader instance oluşabilir mi?
* Hot path içerisinde gereksiz allocation var mı?
* Generic template yapısı korunuyor mu?
* Serialization güvenli mi?

Bu soruların cevabı net olmalıdır.
