# Bottom Navigation Bar UI

## 1. Amaç

Bu doküman, oyunun alt navigation bar yapısını tanımlar.

Bottom Navigation Bar, oyuncunun uygulama içerisindeki ana UI bölümleri arasında geçiş yapmasını sağlar.

Örnek:

```text id="q7m3x1"
┌─────────────────────────────────────┐
│                                     │
│            Current View             │
│                                     │
├─────────────────────────────────────┤
│   Home   │   Play   │   Shop   │ ⚙ │
└─────────────────────────────────────┘
```

Gerçek tab sayısı ve içerikleri oyuna göre değişebilir.

Bu yapı belirli bir oyunun ekranlarına veya domain'lerine hard-code edilmemelidir.

---

# 2. Sorumluluk

BottomNavigationBar'ın sorumlulukları:

* Navigation seçeneklerini göstermek
* Aktif navigation item'ı göstermek
* Kullanıcı seçimlerini almak
* İlgili UI navigation komutunu tetiklemek
* Visual state'i güncellemek

BottomNavigationBar'ın sorumluluğu olmayan şeyler:

* GameFlow state yönetmek
* Gameplay başlatmak
* Level başlatmak
* Economy değiştirmek
* Purchase gerçekleştirmek
* Save data değiştirmek
* Progression hesaplamak
* Screen'in domain logic'ini yönetmek

Temel prensip:

```text id="x2p8m4"
BottomNavigationBar
        ↓
Navigation Intent
        ↓
UI Navigation Owner
        ↓
Target View
```

---

# 3. UI Navigation ve GameFlow Ayrımı

UI navigation ile global game flow aynı şey değildir.

Örneğin:

```text id="w6n3k9"
Home
Shop
Profile
Settings
```

arasında geçiş yapmak UI navigation'dır.

Buna karşılık:

```text id="j4r8q2"
Boot
MainMenu
Gameplay
Pause
Win
Lose
```

global GameFlow state'leridir.

Bu iki kavram birbirine bağlanabilir ancak aynı sistem tarafından yönetilmemelidir.

---

# 4. GameFlow İlişkisi

GameFlow global oyun state'inin sahibidir.

Örneğin:

```text id="m8q1v5"
GameFlow
├── Boot
├── MainMenu
├── Gameplay
├── Pause
├── Win
└── Lose
```

BottomNavigationBar ise UI navigation yapar.

Örneğin:

```text id="p5x9c3"
BottomNavigationBar
        ↓
Select Shop
        ↓
ShopView
```

Bu işlem otomatik olarak:

```text
GameFlow → Gameplay
```

veya:

```text
GameFlow → MainMenu
```

gibi global state değişikliğine sebep olmamalıdır.

---

# 5. Navigation Model

Navigation item'ları generic bir model ile tanımlanabilir.

Örneğin:

```csharp id="n4k7s2"
public enum NavigationId
{
    Home,
    Gameplay,
    Shop,
    Profile,
    Settings
}
```

Gerçek projede kullanılacak enum değerleri oyuna göre değişebilir.

Daha data-driven bir yapı gerekiyorsa ScriptableObject kullanılabilir:

```text id="c8m2v6"
NavigationConfig
├── Id
├── Icon
├── SelectedIcon
├── Label
└── Target
```

Ancak yalnızca birkaç sabit tab varsa sırf data-driven olmak için gereksiz bir ScriptableObject sistemi oluşturulmamalıdır.

---

# 6. Navigation Assetleri

Navigation item'larının görsel asset'leri configuration üzerinden bağlanabilir.

Örneğin:

```text id="r9p4x7"
Art/UI/Navigation/Home.png
Art/UI/Navigation/HomeSelected.png
```

↓

```text id="y2k6m8"
NavigationConfig
├── Icon
└── SelectedIcon
```

↓

```text id="v5n1q3"
NavigationItemView
```

Asset'in nereden ve nasıl yükleneceği:

```text id="g8m4z2"
SYSTEMS/AssetManagement.md
```

kurallarına göre belirlenir.

---

# 7. Navigation Item

Her item kendi presentation state'ini gösterebilir.

Örneğin:

```text id="s6q3p9"
NavigationItem
├── Icon
├── SelectedIcon
├── Label
├── NotificationBadge
└── Button
```

NavigationItem:

* Target screen'i initialize etmez
* Target screen'in domain logic'ini çalıştırmaz
* Economy state değiştirmez
* Save data değiştirmez

Sadece navigation intent üretir.

---

# 8. Selection State

Aktif tab bilgisi BottomNavigationBar veya UI Navigation Owner tarafından yönetilebilir.

Örneğin:

```text id="k7r2m5"
CurrentNavigation = Shop
```

Bu state UI navigation state'idir.

Gameplay state'i değildir.

Dolayısıyla:

```text
Shop Selected
```

şu anlama gelmez:

```text
GameFlow = Gameplay
```

veya:

```text
LevelManager = Shop
```

---

# 9. Navigation Controller

Birden fazla navigation item ve view arasında koordinasyon gerekiyorsa bir `NavigationController` kullanılabilir.

Örneğin:

```text id="d4m8q1"
BottomNavigationBar
        ↓
NavigationController
        ↓
Show Target View
```

NavigationController'ın görevi:

* Aktif view'ı belirlemek
* Target view'ı göstermek
* Önceki view'ı gizlemek
* Navigation state'i yönetmek

NavigationController gameplay system'i haline gelmemelidir.

---

# 10. View Lifecycle

Navigation değiştiğinde view lifecycle açık olmalıdır.

Örneğin:

```text id="f8n3v6"
Select Shop
    ↓
Current View.Hide()
    ↓
ShopView.Show()
```

View'lar scene'de hazır bulunabilir veya ihtiyaç halinde yüklenebilir.

Asset loading strategy feature'ın ihtiyacına göre belirlenir.

---

# 11. View Loading

Küçük ve sık kullanılan UI'lar için direct serialized reference kullanılabilir.

Örneğin:

```text id="q2x7m4"
BottomNavigationBar
    ↓
ShopView Reference
```

Daha büyük veya dinamik UI içerikleri için Addressables kullanılabilir:

```text id="p6r1k8"
NavigationController
    ↓
Load ShopView
    ↓
Show
    ↓
Hide
    ↓
Release when appropriate
```

Ancak her view'ın Addressable yapılması zorunlu değildir.

---

# 12. Navigation ve Scene

Navigation'ın scene değiştirip değiştirmeyeceği architecture tarafından belirlenmelidir.

Örneğin:

```text id="x4m9q2"
Home
Shop
Profile
```

aynı scene içerisindeki view'lar olabilir.

Alternatif:

```text id="v7k3n8"
Home Scene
Shop Scene
Profile Scene
```

kullanılabilir.

Ancak UI tab navigation için gereksiz scene transition kullanılmamalıdır.

Küçük UI bölümleri için view-level navigation daha uygun olabilir.

---

# 13. Gameplay Tab

Bir navigation item gameplay'e geçiş yapıyorsa BottomNavigationBar gameplay logic'ini kendisi çalıştırmamalıdır.

Örneğin:

```text id="m5q8r3"
Play Button
    ↓
Gameplay Navigation Request
    ↓
GameFlow
    ↓
Gameplay State
    ↓
LevelSystem
    ↓
Gameplay
```

Burada önemli ayrım:

```text
BottomNavigationBar
    → Intent

GameFlow
    → Global State

LevelSystem
    → Level Lifecycle
```

---

# 14. Gameplay'den Navigation'a Dönüş

Gameplay sırasında BottomNavigationBar görünmüyorsa navigation UI gameplay UI'dan bağımsız olabilir.

Örneğin:

```text id="z8p4c6"
Gameplay
    ↓
Exit Gameplay
    ↓
GameFlow → MainMenu
    ↓
Home View
```

Gameplay sistemi doğrudan BottomNavigationBar'ın visual state'ini yönetmemelidir.

---

# 15. Settings

Settings tab veya button varsa:

```text id="q3n7m1"
BottomNavigationBar
    ↓
Settings Request
    ↓
SettingsView / SettingsPopup
```

Settings içeriği:

```text id="r6v2k9"
UI/SettingsPopup.md
```

içerisinde tanımlanır.

BottomNavigationBar settings'in iç mantığını sahiplenmez.

---

# 16. Shop

Shop navigation item'ı:

```text id="h4m8q2"
BottomNavigationBar
    ↓
Shop
    ↓
ShopView
```

akışını tetikleyebilir.

Shop:

```text id="c7x3n9"
UI/ShopView.md
```

içerisinde tanımlanır.

Purchase logic:

```text id="p2k6r5"
SYSTEMS/Monetization.md
```

içerisinde tanımlanır.

BottomNavigationBar purchase logic'ine erişmemelidir.

---

# 17. Notification Badge

Navigation item üzerinde badge bulunabilir.

Örneğin:

```text id="n8q4m2"
Shop
   ● 3
```

Badge'in gösterilmesi gereken durumun sahibi ilgili system olmalıdır.

Örneğin:

```text id="s5r9k1"
RewardSystem
    ↓
Unread Reward Count
    ↓
Event
    ↓
ShopNavigationItem
    ↓
Badge
```

BottomNavigationBar unread count hesaplamamalıdır.

---

# 18. Economy ile İlişki

Navigation bar üzerinde currency gösteriliyorsa currency'nin sahibi yine EconomySystem'dir.

BottomNavigationBar:

```text id="k3p7x5"
Currency = Presentation
```

olarak davranmalıdır.

Currency hesaplama:

```text id="q9m2r6"
EconomySystem
```

sorumluluğundadır.

---

# 19. Profile ile İlişki

Profile navigation item'ı:

```text id="v4n8q1"
Profile Button
    ↓
Profile View
```

akışını tetikleyebilir.

Profile data'sının owner'ı Profile/Player system'dir.

BottomNavigationBar profile state'ini sahiplenmez.

---

# 20. Event Kullanımı

Navigation için her action'da global EventBus kullanılması zorunlu değildir.

Örneğin açık bir local relationship varsa direct reference yeterlidir:

```text id="j6q3m8"
BottomNavigationBar
    ↓
NavigationController
```

Global veya loosely coupled bir iletişim gerekiyorsa event kullanılabilir:

```text id="x1v7p4"
Navigation Request
    ↓
Event System
    ↓
Navigation Owner
```

Karar mevcut architecture'a göre verilmelidir.

Gereksiz event abstraction oluşturulmamalıdır.

---

# 21. Input

BottomNavigationBar input'u gameplay input'undan ayrı düşünülmelidir.

Önerilen akış:

```text id="m8r2q5"
Pointer / Touch
        ↓
UI Button
        ↓
Navigation Intent
        ↓
NavigationController
```

Gameplay input pipeline:

```text id="c4n7x9"
Touch / Mouse / Keyboard / Gamepad
        ↓
Input Controller
        ↓
Gameplay Command
        ↓
Gameplay System
```

Bu iki pipeline birbirine karıştırılmamalıdır.

---

# 22. Animation

Navigation tab değişimlerinde animasyon kullanılabilir.

Örneğin:

```text id="z5q8m3"
Unselected
    ↓
Selected
    ↓
Icon Scale
    ↓
Label Fade
```

Animation yalnızca presentation state'i değiştirmelidir.

Gameplay veya economy state'i animation completion'a bağlanmamalıdır.

Kullanılan tween sistemi proje convention'ına uygun olmalıdır.

---

# 23. Pooling

BottomNavigationBar'ın kendisi genellikle pooling gerektirmez.

Ancak navigation item içerisinde sık oluşturulan geçici UI elementleri varsa pooling düşünülebilir.

Örneğin:

```text id="p7m2k4"
Notification Popup
Floating Feedback
Temporary Badge
```

Pooling uygulanıyorsa object state tamamen resetlenmelidir.

---

# 24. Performance

Navigation UI sürekli aktif olan bir UI olabileceği için:

* `Update` gereksiz kullanılmamalıdır.
* Her frame navigation state kontrol edilmemelidir.
* Gereksiz layout rebuild yapılmamalıdır.
* Tekrarlanan `GetComponent` çağrıları yapılmamalıdır.
* UI event listener'ları tekrar tekrar oluşturulmamalıdır.
* Gereksiz string allocation yapılmamalıdır.

Navigation state değiştiğinde UI güncellenmesi tercih edilir.

```text id="n3r8v6"
State Changed
    ↓
Update Visual
```

yerine:

```text id="d6q1m9"
Every Frame
    ↓
Check State
    ↓
Maybe Update
```

yaklaşımı tercih edilmez.

---

# 25. Serialization

Navigation configuration veya prefab'larda serialized fields kullanılıyorsa field isimleri ve türleri data contract'ın parçasıdır.

Örneğin:

```csharp id="x8k4p2"
[SerializeField] private Sprite selectedIcon;
```

gibi alanların yeniden adlandırılması serialization kaybına neden olabilir.

Gerekli durumlarda:

```csharp id="q5m7r1"
[FormerlySerializedAs("selectedSprite")]
```

kullanılmalıdır.

Prefab ve ScriptableObject dependency'leri değiştirilirken dikkatli olunmalıdır.

---

# 26. Generic Template Rule

BottomNavigationBar belirli bir oyunun navigation modeline hard-code edilmemelidir.

Örneğin template core'unda:

```text id="b3n8q6"
Kingdom
Star
Building
Decoration
```

gibi domain-specific tab'ler bulunmamalıdır.

Bunun yerine generic navigation kullanılmalıdır:

```text id="r7m2x4"
Home
Gameplay
Shop
Profile
Settings
```

Ancak bunlar da zorunlu değildir.

Her oyun yalnızca ihtiyaç duyduğu navigation item'larını kullanmalıdır.

---

# 27. Source of Truth

BottomNavigationBar için ownership:

```text id="y4p8m2"
Navigation State
    → NavigationController / UI Navigation Owner

Global Game State
    → GameFlow

Gameplay State
    → Gameplay Systems

Currency
    → EconomySystem

Progression
    → ProgressionSystem

Purchase
    → MonetizationSystem

Visual Asset Selection
    → Configuration

Visual Presentation
    → BottomNavigationBar
```

---

# 28. Örnek Tam Akış

Shop'a tıklama:

```text id="q8m3v5"
Player Touch
    ↓
Shop Navigation Item
    ↓
Navigation Request
    ↓
NavigationController
    ↓
ShopView.Show()
    ↓
BottomNavigationBar.Select(Shop)
```

Shop satın alma işlemi:

```text id="x5r7n2"
ShopView
    ↓
Purchase Request
    ↓
MonetizationSystem
    ↓
Transaction Result
    ↓
EconomySystem
    ↓
Currency Changed
    ↓
TopBarHeader
```

BottomNavigationBar bu transaction zincirinin parçası değildir.

---

# 29. Definition of Done

BottomNavigationBar tamamlanmadan önce:

* UI navigation ile GameFlow ayrılmış mı?
* BottomNavigationBar global game state sahibi mi?
* Navigation state'in owner'ı belli mi?
* Navigation item'ları generic mi?
* Asset'ler configuration üzerinden mi bağlanıyor?
* Hard-coded asset path var mı?
* Shop/Monetization logic UI'ya girmiş mi?
* Economy state UI tarafından yönetiliyor mu?
* Profile data UI tarafından yönetiliyor mu?
* Badge state doğru system'den geliyor mu?
* Gereksiz EventBus kullanımı var mı?
* View lifecycle net mi?
* Duplicate navigation/view instance oluşabilir mi?
* Navigation state her frame kontrol ediliyor mu?
* Animation gameplay state'ine bağımlı mı?
* Serialization güvenli mi?
* Scene transition gerçekten gerekli mi?
* Generic template prensibi korunuyor mu?

Bu soruların cevapları net olmalıdır.
