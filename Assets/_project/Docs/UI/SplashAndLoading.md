# Splash and Loading UI

## 1. Amaç

Bu doküman, oyunun açılışındaki Splash ve Loading UI yapısını tanımlar.

Splash UI, oyuncuya açılış sırasında görsel bir başlangıç ekranı sağlar.

Loading UI ise sistemlerin, configuration'ın, save data'nın veya gerekli gameplay içeriklerinin hazırlanması sırasında kullanıcıya durum gösterir.

Bu dokümanın kapsamı:

* Splash görünümü
* Splash asset'leri
* Splash configuration
* Loading görünümü
* Bootstrap ile UI arasındaki ilişki
* Asset reference yaklaşımı
* Splash lifecycle

Bu doküman Bootstrap'ın initialization logic'ini tanımlamaz.

Bootstrap davranışı:

```text
BOOTSTRAP.md
```

Asset loading stratejisi:

```text
SYSTEMS/AssetManagement.md
```

---

# 2. Temel Sorumluluk

Splash UI'nın görevi:

```text
Show
Hide
Update Visual State
```

Splash UI'nın görevi olmayan şeyler:

```text
Initialize Economy
Load Save Data
Initialize Analytics
Initialize Gameplay
Determine Game Flow
Load Level Rules
Calculate Rewards
```

Splash UI yalnızca initialization sürecinin görsel sunumunu yapar.

---

# 3. Genel Akış

Oyunun başlangıç akışı:

```text
Application Start
        ↓
Bootstrap
        ↓
SplashView.Show()
        ↓
Core Systems Initialization
        ↓
Configuration Ready
        ↓
Save Data Ready
        ↓
Runtime Systems Ready
        ↓
SplashView.Hide()
        ↓
GameFlow
        ↓
MainMenu
```

Bootstrap ve Splash UI birbirinden ayrıdır.

```text
Bootstrap
    → "Oyun hazır mı?"

SplashView
    → "Hazırlık sürecini nasıl gösteriyorum?"
```

---

# 4. Splash Assetleri

Splash için gerekli asset'ler doğrudan `SplashView` içerisinde hard-code edilmemelidir.

Önerilen yapı:

```text
Art/UI/Splash/
├── SplashBackground.png
└── GameLogo.png
```

Bunların hangi asset olarak kullanılacağı:

```text
SplashConfig
```

üzerinden tanımlanır.

Örnek:

```text
SplashConfig
├── Background
├── Logo
└── MinimumDisplayDuration
```

Akış:

```text
SplashBackground.png
        ↓
SplashConfig.Background
        ↓
SplashView
```

ve:

```text
GameLogo.png
        ↓
SplashConfig.Logo
        ↓
SplashView
```

Asset management stratejisi için:

```text
SYSTEMS/AssetManagement.md
```

kuralları uygulanır.

---

# 5. SplashConfig

Splash'a ait designer-tunable değerler ScriptableObject üzerinden tutulabilir.

Örnek:

```csharp
[CreateAssetMenu(
    fileName = "SplashConfig",
    menuName = "Project/Config/Splash Config")]
public class SplashConfig : ScriptableObject
{
    [SerializeField] private Sprite background;
    [SerializeField] private Sprite logo;
    [SerializeField] private float minimumDisplayDuration = 1f;

    public Sprite Background => background;
    public Sprite Logo => logo;
    public float MinimumDisplayDuration => minimumDisplayDuration;
}
```

Configuration yalnızca static/configuration data tutmalıdır.

Session sırasında değişen state `SplashConfig` içerisinde tutulmamalıdır.

Örneğin aşağıdaki kullanım doğru değildir:

```csharp
public bool HasAlreadyShown;
public float CurrentProgress;
public bool IsInitialized;
```

Bunlar runtime state'tir ve configuration asset'inde tutulmamalıdır.

---

# 6. SplashView

`SplashView`, configuration'dan aldığı asset'leri ekranda gösterir.

Örnek:

```csharp
public class SplashView : MonoBehaviour
{
    [SerializeField] private SplashConfig config;
    [SerializeField] private Image backgroundImage;
    [SerializeField] private Image logoImage;

    public void Show()
    {
        backgroundImage.sprite = config.Background;
        logoImage.sprite = config.Logo;

        gameObject.SetActive(true);
    }

    public void Hide()
    {
        gameObject.SetActive(false);
    }
}
```

`SplashView`:

* Asset seçmez
* Asset path aramaz
* Gameplay initialize etmez
* Save load etmez
* GameFlow değiştirmez
* Economy başlatmaz

Sadece presentation sorumluluğuna sahiptir.

---

# 7. SplashView ve Bootstrap

Bootstrap Splash UI'yı kullanabilir ancak Splash'ın içindeki initialization logic'i sahiplenmemelidir.

Önerilen ilişki:

```text
Bootstrap
    │
    ├── SplashView.Show()
    │
    ├── Initialize Systems
    │
    ├── Wait for Required Systems
    │
    ├── SplashView.Hide()
    │
    └── Hand-off to GameFlow
```

Bootstrap'ın görevi orchestration'dır.

SplashView'ın görevi presentation'dır.

---

# 8. Splash Görünürlük Durumu

Splash'ın ne kadar süre görüneceği ile sistemlerin ne kadar sürede initialize olduğu birbirinden ayrılmalıdır.

Örneğin:

```text
Minimum Splash Duration = 1 second

Initialization Duration = 0.2 second
```

Bu durumda:

```text
Initialization Ready
        ↓
Minimum Duration Completed
        ↓
Hide Splash
```

Tersi durumda:

```text
Minimum Duration Completed
        ↓
Initialization Still Running
        ↓
Keep Splash Visible
        ↓
Initialization Ready
        ↓
Hide Splash
```

Splash'ın minimum gösterim süresi yalnızca UX davranışıdır.

Gameplay veya system initialization bu süreye bağlanmamalıdır.

---

# 9. Loading UI

Loading UI, Splash'tan farklı olarak ilerleme durumunu gösterebilir.

Örneğin:

```text
Loading...
[████████░░] 80%
```

Ancak progress değeri gerçek bir progress'i temsil etmiyorsa sahte kesinlik yaratılmamalıdır.

Örneğin sistem gerçekten hangi aşamada olduğunu bilmiyorsa:

```text
Loading...
```

gibi indeterminate bir gösterim tercih edilebilir.

---

# 10. Loading Progress Ownership

Loading UI progress değerinin sahibi değildir.

Progress'i oluşturan system veya Bootstrap gerçek progress state'in sahibidir.

Örneğin:

```text
Bootstrap
    ↓
Initialization Progress
    ↓
LoadingView
```

LoadingView:

```text
"Progress = 0.75"
```

değerini gösterebilir.

Ancak:

```text
"Progress = 0.75"
```

değerini kendisi hesaplamamalıdır.

UI:

```text
Presentation
```

olmalıdır.

---

# 11. Initialization State

Bootstrap initialization durumunu temsil eden bir state sağlayabilir.

Örneğin:

```csharp
public enum BootstrapState
{
    NotStarted,
    Initializing,
    Ready,
    Failed
}
```

Loading UI bu state'e göre görünüm değiştirebilir.

Örneğin:

```text
NotStarted
    ↓
Splash

Initializing
    ↓
Loading

Ready
    ↓
Hide

Failed
    ↓
Error UI
```

Ancak bu state'in authoritative owner'ı UI değildir.

---

# 12. Error State

Initialization başarısız olursa Splash/Loading UI hata durumunu gösterebilir.

Örneğin:

```text
Initialization Failed

Unable to start the game.

[Retry]
```

UI yalnızca sonucu gösterir.

Retry davranışının sahibi Bootstrap veya ilgili initialization owner'ıdır.

Örneğin:

```text
Retry Button
    ↓
Bootstrap Retry
    ↓
Initialization
    ↓
Ready
```

UI kendi başına system initialization gerçekleştirmemelidir.

---

# 13. Retry

Retry gerekiyorsa initialization'ın idempotent olması gerekir.

Aynı initialization'ın ikinci kez çalıştırılması:

```text
Duplicate Event Subscription
Duplicate Save Load
Duplicate SDK Initialization
Duplicate Object Creation
```

gibi sorunlar oluşturmamalıdır.

Retry logic:

```text
Failure
 ↓
Cleanup / Reset
 ↓
Retry Initialization
 ↓
Ready
```

şeklinde açıkça tanımlanmalıdır.

---

# 14. Splash Animation

Splash animasyonları presentation sorumluluğundadır.

Örneğin:

```text
Logo Fade In
Background Fade In
Loading Indicator
Logo Fade Out
```

Bu animasyonlar tween veya mevcut proje animation sistemini kullanmalıdır.

Projede kullanılan tween sistemi ne ise o kullanılmalıdır.

Yeni bir tween framework eklenmemelidir.

Animation lifecycle:

```text
Show
 ↓
Play
 ↓
Hide
 ↓
Kill / Cleanup
```

Pooling veya disable/enable lifecycle'ı varsa animation state tamamen resetlenmelidir.

---

# 15. Gameplay Bağımlılığı

Splash UI gameplay system'lerine doğrudan bağımlı olmamalıdır.

Kaçınılması gereken:

```text
SplashView
    ↓
LevelManager
    ↓
BoardController
    ↓
Gameplay
```

Tercih edilen:

```text
Bootstrap
    ↓
Initialization
    ↓
SplashView
```

Gameplay daha sonra GameFlow tarafından başlatılır.

---

# 16. Main Menu ile Geçiş

Splash'ın kapanması doğrudan `MainMenuView` tarafından yönetilmemelidir.

Global flow:

```text
Bootstrap
    ↓
GameFlow
    ↓
MainMenu
```

Splash:

```text
Bootstrap'ın initialization presentation'ı
```

olarak kalır.

Bu sayede ileride:

```text
Boot
 ↓
Login
 ↓
Remote Config
 ↓
MainMenu
```

gibi farklı başlangıç akışları eklenebilir.

---

# 17. Asset Loading Strategy

Splash asset'leri için varsayılan yaklaşım:

```text
Direct Serialized Reference
```

olmalıdır.

Örneğin:

```text
SplashConfig
    ↓
Sprite Reference
```

Boot sırasında Splash'ın kendisini gösterebilmek için Splash'ın ihtiyaç duyduğu asset'lerin tekrar bir remote veya async asset-loading pipeline'ına bağımlı olması tercih edilmez.

Büyük veya remote içerikler için Addressables kullanılabilir.

Ancak:

```text
"Her asset Addressable olmalı."
```

şeklinde bir kural yoktur.

Detay:

```text
SYSTEMS/AssetManagement.md
```

---

# 18. Scene Placement

Splash UI'nın hangi scene içerisinde bulunduğu proje bootstrap mimarisine bağlıdır.

Önerilen yapılardan biri:

```text
Bootstrap Scene
├── Bootstrap
├── SplashView
└── Required Boot UI
```

veya mevcut proje mimarisine uygun şekilde:

```text
Persistent Root
└── SplashView
```

kullanılabilir.

Burada önemli olan GameObject'in nerede bulunduğundan çok ownership'in açık olmasıdır.

Splash UI'nın lifecycle'ından tek bir owner sorumlu olmalıdır.

---

# 19. Persistent Splash

Splash'ın persistent olması gerekiyorsa:

```text
DontDestroyOnLoad
```

kullanımı kontrollü olmalıdır.

Birden fazla Splash instance oluşturulmamalıdır.

Özellikle scene transition sırasında:

```text
Bootstrap Scene
    ↓
Main Menu Scene
```

geçişinde duplicate UI oluşmamalıdır.

---

# 20. Scene Transition

Splash'tan Main Menu'ye geçiş:

```text
Initialization Ready
        ↓
GameFlow Ready
        ↓
MainMenu State
        ↓
MainMenu Presentation
```

şeklinde ilerlemelidir.

Splash:

```text
Hide
```

edilir.

Main Menu:

```text
Show
```

edilir.

Bir UI diğer UI'nın lifecycle'ını doğrudan yönetmemelidir.

---

# 21. Generic Template Rule

Splash sistemi oyun içeriğine özel olmamalıdır.

Örneğin aşağıdaki kavramlar Splash sistemine eklenmemelidir:

```text
Kingdom
Stars
Buildings
Decorations
Lives
Board
```

Splash yalnızca generic initialization presentation sağlamalıdır.

Oyun içeriği değişse bile Splash sistemi mümkün olduğunca değişmeden kullanılabilmelidir.

---

# 22. Asset Integration Example

Örnek bir proje:

```text
Assets/_Project/
├── Art/
│   └── UI/
│       └── Splash/
│           ├── Background.png
│           └── Logo.png
│
├── ScriptableObjects/
│   └── UI/
│       └── SplashConfig.asset
│
└── Prefabs/
    └── UI/
        └── SplashView.prefab
```

Bağlantı:

```text
Background.png
       ↓
SplashConfig.Background
       │
Logo.png ───────→ SplashConfig.Logo
       ↓
SplashView
       ↓
Bootstrap
```

Burada:

* `Background.png` asset'tir.
* `SplashConfig.asset` asset seçimidir.
* `SplashView.prefab` presentation'dır.
* `Bootstrap` initialization orchestration'ıdır.

Bu ayrım template'in farklı oyunlarda tekrar kullanılmasını sağlar.

---

# 23. Source of Truth

Splash sisteminde source of truth:

```text
Asset
    → Concrete visual asset

SplashConfig
    → Which asset is used

SplashView
    → How it is presented

Bootstrap
    → When it is shown/hidden relative to initialization

GameFlow
    → What happens after boot
```

Hiçbir katman başka bir katmanın sorumluluğunu devralmamalıdır.

---

# 24. Definition of Done

Splash/Loading UI tamamlanmadan önce:

* Splash asset'leri doğru yerde mi?
* Asset seçimi configuration üzerinden mi?
* Asset path hard-code edilmiş mi?
* SplashConfig runtime state tutuyor mu?
* SplashView yalnızca presentation sorumluluğunda mı?
* Bootstrap initialization orchestration'ını mı yapıyor?
* Loading progress'in gerçek sahibi belli mi?
* Splash minimum display duration initialization logic'ten ayrılmış mı?
* Animation cleanup yapılıyor mu?
* Duplicate Splash instance oluşabilir mi?
* Scene transition lifecycle'ı belli mi?
* Addressables gerçekten gerekli mi?
* Boot-critical asset gereksiz async loading'e bağımlı mı?
* Splash generic template yapısını koruyor mu?
* UI gameplay state'ini sahipleniyor mu?

Bu soruların cevapları net olmalıdır.
