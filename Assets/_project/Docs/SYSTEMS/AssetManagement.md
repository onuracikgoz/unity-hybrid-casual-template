# Asset Management

## 1. Amaç

Bu doküman, projedeki asset'lerin nasıl referanslanacağını, yükleneceğini, kullanılacağını ve yaşam döngüsünün nasıl yönetileceğini tanımlar.

Asset yönetimi ile asset'in gameplay veya UI içerisinde nasıl kullanılacağı birbirinden ayrılmalıdır.

Temel ayrım:

```text
Asset
  ↓
Configuration
  ↓
Feature / System
  ↓
Runtime Usage
```

Örneğin:

```text
Assets/_Project/Art/UI/Logo.png
        ↓
SplashConfig
        ↓
SplashView
        ↓
Runtime
```

Asset klasör yapısı tek başına asset'in hangi feature tarafından kullanılacağını belirlemez.

---

# 2. Asset Klasör Yapısı

Asset'ler içerik türüne göre organize edilmelidir.

Örnek:

```text
Assets/_Project/
├── Art/
│   ├── UI/
│   ├── Characters/
│   ├── Environment/
│   ├── VFX/
│   └── Materials/
│
├── Audio/
│   ├── Music/
│   ├── SFX/
│   └── Voice/
│
├── Prefabs/
├── Scenes/
├── ScriptableObjects/
└── ...
```

Klasör yapısı asset'in fiziksel konumunu organize eder.

Ancak aşağıdaki soruların cevabı klasör isminden çıkarılmamalıdır:

* Bu asset hangi feature'a ait?
* Nerede gösterilecek?
* Hangi sistem kullanacak?
* Ne zaman yüklenecek?
* Ne zaman serbest bırakılacak?

Bu bilgiler ilgili configuration, feature veya system tarafından tanımlanmalıdır.

---

# 3. Asset Kullanım Katmanları

Asset kullanımı üç temel katmana ayrılır.

## 3.1 Asset

Fiziksel dosya veya Unity asset'i.

Örnek:

```text
Logo.png
CoinIcon.png
WinEffect.prefab
ButtonClick.wav
```

---

## 3.2 Configuration

Asset'in hangi içerik veya feature için kullanılacağını tanımlar.

Örneğin:

```csharp
[CreateAssetMenu]
public class SplashConfig : ScriptableObject
{
    [SerializeField] private Sprite background;
    [SerializeField] private Sprite logo;
    [SerializeField] private float minimumDisplayDuration;
}
```

Burada `SplashConfig`, asset ile Splash feature'ı arasındaki bağlantıdır.

```text
Logo.png
    ↓
SplashConfig.logo
    ↓
SplashView
```

---

## 3.3 Runtime Usage

Feature veya system configuration üzerinden asset'i kullanır.

Örneğin:

```csharp
public class SplashView : MonoBehaviour
{
    [SerializeField] private SplashConfig config;
    [SerializeField] private Image logoImage;

    private void Awake()
    {
        logoImage.sprite = config.Logo;
    }
}
```

Runtime kodu mümkün olduğunca doğrudan asset path'i bilmemelidir.

---

# 4. Asset Referansı Nerede Tanımlanır?

Bir asset'in hangi feature tarafından kullanılacağı mümkün olduğunca ilgili configuration üzerinden tanımlanmalıdır.

Örnek:

```text
Art/UI/Logo.png
        ↓
SplashConfig
        ↓
SplashView
```

Başka bir örnek:

```text
Art/UI/CoinIcon.png
        ↓
CurrencyConfig
        ↓
EconomySystem
        ↓
TopBarHeader
```

Gameplay asset'i:

```text
Art/Gameplay/Tiles/RedTile.png
        ↓
TileConfig
        ↓
TileSystem
        ↓
TileView
```

Bu sayede asset değiştirmek için runtime kodunu değiştirmek gerekmez.

---

# 5. Direct Asset References

Küçük, boot-critical veya sürekli kullanılan asset'ler için doğrudan Unity reference tercih edilebilir.

Örneğin:

```csharp
[SerializeField] private Sprite logo;
```

veya:

```csharp
[SerializeField] private GameObject tilePrefab;
```

Bu yaklaşım özellikle aşağıdaki durumlarda uygundur:

* Splash screen asset'leri
* Küçük UI asset'leri
* Küçük icon'lar
* Scene ile birlikte yaşam süren prefab'lar
* Her zaman gerekli olan configuration asset'leri
* Küçük ve statik içerikler

Örnek:

```text
SplashConfig
    ├── Background
    └── Logo
```

Bu asset'ler doğrudan serialized reference olarak bağlanabilir.

---

# 6. Addressables

Addressables, asset'in runtime sırasında yüklenmesi veya gerektiğinde serbest bırakılması gerektiğinde kullanılabilir.

Özellikle aşağıdaki durumlarda uygundur:

* Büyük asset'ler
* Çok sayıda level
* Downloadable content
* Uzun süre kullanılmayan içerikler
* Memory üzerinde sürekli tutulması gerekmeyen içerikler
* Remote content
* Büyük prefab veya VFX paketleri

Örnek:

```text
LevelDefinition
    ↓
Addressable Level Content
    ↓
Load
    ↓
Gameplay
    ↓
Release
```

Addressables kullanılacaksa load ve release ownership açıkça tanımlanmalıdır.

Asset'i kimin yüklediği, kimin kullandığı ve kimin release ettiği belirsiz olmamalıdır.

---

# 7. Resources

`Resources` yalnızca gerçekten gerekli olduğu durumlarda kullanılmalıdır.

Genel asset yönetim sistemi olarak `Resources` klasörüne bağımlı bir mimari oluşturulmamalıdır.

Özellikle aşağıdaki kullanım şekillerinden kaçınılmalıdır:

```csharp
Resources.Load(...)
```

çağrılarını gameplay veya UI kodunun farklı noktalarına dağıtmak.

Bunun yerine asset erişimi merkezi ve açık bir yapı üzerinden yönetilmelidir.

---

# 8. Boot-Critical Assets

Oyunun başlaması için gerekli olan asset'ler özel olarak değerlendirilmelidir.

Örneğin:

```text
Splash Background
Splash Logo
Boot UI
Loading Indicator
Critical Configuration
```

Bu asset'ler Bootstrap'ın başlayabilmesi için tekrar bir asset-loading sistemine bağımlı hale getirilmemelidir.

Tercih edilen yapı:

```text
Application Start
    ↓
Bootstrap
    ↓
SplashView
    ↓
Direct Serialized Reference
    ↓
Splash Visible
```

Boot sırasında gerekli olmayan büyük asset'ler daha sonra yüklenebilir.

Örneğin:

```text
Boot
 ↓
Main Menu
 ↓
Level Selected
 ↓
Load Level Content
```

---

# 9. Asset Loading ve Asset Reference Ayrımı

Asset'in nasıl referanslandığı ile nasıl yüklendiği aynı karar değildir.

Örneğin:

```text
SplashConfig
    ↓
Sprite Reference
```

asset'in hangi asset olduğunu belirler.

Bunun nasıl memory'e getirildiği ise asset management stratejisidir.

Temel seçenekler:

```text
Direct Reference
Addressables
Resources
```

Bu karar feature'ın ihtiyaçlarına göre verilmelidir.

---

# 10. Asset Lifetime

Her runtime asset'in yaşam süresi mümkün olduğunca açık olmalıdır.

Örnek:

```text
Boot Asset
    → Application Lifetime

Main Menu Asset
    → Main Menu Lifetime

Level Asset
    → Level Lifetime

Temporary VFX
    → Pool Lifetime
```

Bir asset yalnızca ihtiyaç duyulduğu süre boyunca tutulmalıdır.

Özellikle büyük asset'lerin gereksiz şekilde global olarak tutulmasından kaçınılmalıdır.

---

# 11. Load / Release Ownership

Asset'i yükleyen tarafın lifecycle sorumluluğu açık olmalıdır.

Örneğin:

```text
LevelSystem
    ↓
Load Level Content
    ↓
Gameplay
    ↓
Level Completed
    ↓
Release Level Content
```

Bir system başka bir system adına asset yükleyip kontrolünü belirsiz şekilde bırakmamalıdır.

Ownership mümkün olduğunca şu soruya cevap verebilmelidir:

> "Bu asset artık kullanılmadığında kim release edecek?"

---

# 12. UI Assetleri

UI asset'leri ilgili UI configuration tarafından tanımlanmalıdır.

Örneğin:

```text
UI/
├── SplashAndLoading.md
├── ShopView.md
└── TopBarHeader.md
```

Splash:

```text
SplashConfig
    ├── Background
    └── Logo
```

Shop:

```text
ShopConfig
    ├── CurrencyIcon
    ├── ProductIcons
    └── PurchaseFeedback
```

UI component'leri asset path'lerini hard-code etmemelidir.

---

# 13. Gameplay Assetleri

Gameplay asset'leri gameplay configuration veya ilgili system configuration üzerinden tanımlanmalıdır.

Örneğin:

```text
TileConfig
    ├── Sprite
    ├── DestroyVFX
    └── SpawnVFX
```

ve:

```text
EnemyConfig
    ├── Prefab
    ├── HitVFX
    └── DeathVFX
```

Gameplay kodunun şu şekilde asset araması tercih edilmez:

```csharp
Resources.Load<GameObject>("Enemies/Enemy");
```

Bunun yerine:

```text
EnemyConfig
    ↓
EnemySystem
    ↓
EnemyPrefab
```

kullanılmalıdır.

---

# 14. Prefab Assetleri

Prefab'lar da asset management kurallarına tabidir.

Prefab'in nerede kullanılacağı ilgili feature tarafından belirlenmelidir.

Örneğin:

```text
Prefabs/VFX/CoinCollect.prefab
        ↓
RewardConfig
        ↓
RewardSystem
        ↓
Pool
        ↓
Runtime
```

Sık kullanılan ve kısa ömürlü prefab'lar mümkün olduğunca pooling ile kullanılmalıdır.

---

# 15. Pooling ve Asset Management

Asset'in Addressables veya direct reference olması pooling kararından bağımsızdır.

Örneğin:

```text
Addressable Prefab
    ↓
Load
    ↓
Pool
    ↓
Spawn / Despawn
    ↓
Release when Pool Lifetime Ends
```

Pool'a alınan object release edilmeden önce pool ownership'i ile asset ownership'i birbirine karıştırılmamalıdır.

Pool object'in yaşam döngüsü açıkça tanımlanmalıdır.

---

# 16. Asset Değiştirme

Bir asset'in değiştirilmesi mümkün olduğunca runtime kodunda değişiklik gerektirmemelidir.

Örneğin:

```text
Old Logo
    ↓
SplashConfig.logo
```

yerine:

```text
New Logo
    ↓
SplashConfig.logo
```

yapılabilmelidir.

Aynı prensip:

* UI icon'ları
* VFX
* SFX
* Character prefab'ları
* Level content
* Reward visuals
* Currency visuals

için de geçerlidir.

---

# 17. Asset Naming

Asset isimleri mevcut proje naming convention'ına uymalıdır.

Örnek:

```text
UI_Logo
UI_CoinIcon
VFX_CoinCollect
SFX_ButtonClick
PF_Enemy
SO_SplashConfig
SO_LevelConfig
```

Proje içinde farklı bir naming convention mevcutsa yeni convention oluşturulmaz. Mevcut convention korunur.

---

# 18. Asset Dependency Kuralları

Asset dependency'leri mümkün olduğunca tek yönlü ve anlaşılır olmalıdır.

Tercih edilen:

```text
Config
  ↓
Feature
  ↓
Runtime
```

Kaçınılması gereken:

```text
UI
 ↕
Gameplay
 ↕
Config
 ↕
System
```

Özellikle UI asset'i veya UI component'i gameplay system'inin runtime state'ini sahiplenmemelidir.

---

# 19. Hard-Coded Asset Path

Aşağıdaki yaklaşım mümkün olduğunca kullanılmamalıdır:

```csharp
Resources.Load<Sprite>("UI/CoinIcon");
```

veya:

```csharp
Addressables.LoadAssetAsync<Sprite>("coin_icon");
```

gibi asset key'lerinin feature kodunun farklı noktalarına dağıtılması.

Asset erişimi açık bir configuration veya merkezi asset management abstraction'ı üzerinden yapılmalıdır.

Ancak sırf abstraction oluşturmak için gereksiz `IAssetProvider`, `IAssetLoader`, `AssetServiceFactory` gibi katmanlar eklenmemelidir.

Abstraction ancak gerçek bir ihtiyaç varsa eklenmelidir.

---

# 20. Asset Management ve Documentation

Her feature kendi asset kullanımını kendi dokümanında tanımlamalıdır.

Örneğin:

```text
UI/SplashAndLoading.md
```

şunları açıklayabilir:

```text
Required Assets
    ├── SplashConfig
    ├── Background
    └── Logo
```

`AssetManagement.md` ise bu asset'lerin **nasıl yönetileceğini** açıklar.

Dolayısıyla:

```text
AssetManagement.md
    → Asset nasıl yönetilir?

SplashAndLoading.md
    → Splash hangi asset'leri kullanır?

SplashConfig
    → Hangi concrete asset kullanılacak?

SplashView
    → Asset runtime'da nasıl gösterilecek?
```

---

# 21. Asset Management Karar Ağacı

Yeni bir asset eklenirken şu sıra izlenmelidir:

```text
Asset oluşturuldu
        ↓
Hangi feature/system kullanıyor?
        ↓
Configuration gerekiyor mu?
        ↓
Boot-critical mi?
        ↓
Direct Reference yeterli mi?
        ↓
Büyük / dinamik / remote içerik mi?
        ↓
Addressables gerekli mi?
        ↓
Lifetime nedir?
        ↓
Kim load ediyor?
        ↓
Kim release ediyor?
```

Her asset için Addressables kullanmak zorunlu değildir.

Her asset için yeni bir manager oluşturmak da zorunlu değildir.

---

# 22. Source of Truth

Asset yönetiminde source of truth aşağıdaki şekilde olmalıdır:

```text
Asset File
    ↓
Concrete asset

ScriptableObject / Configuration
    ↓
Asset selection

Feature / System
    ↓
Asset usage

Asset Management
    ↓
Loading / lifetime strategy
```

Hiçbir runtime component, kendi asset seçimini gizli veya dağınık şekilde yapmamalıdır.

---

# 23. Definition of Done

Asset ile ilgili bir feature tamamlanmadan önce:

* Asset doğru klasörde mi?
* Hangi feature/system tarafından kullanıldığı belli mi?
* Gerekliyse configuration üzerinden referanslanıyor mu?
* Runtime kodunda hard-coded path var mı?
* Direct Reference / Addressables / Resources seçimi bilinçli mi?
* Boot-critical asset gereksiz şekilde async loading'e bağımlı mı?
* Asset lifetime belli mi?
* Load ownership belli mi?
* Release ownership belli mi?
* Pooled prefab'lar doğru şekilde yönetiliyor mu?
* UI asset'i gameplay state'i sahipleniyor mu?
* Gereksiz asset abstraction oluşturulmuş mu?
* Serialization güvenli mi?
* Asset değiştiğinde runtime kodunu değiştirmek gerekiyor mu?

Bu soruların cevabı net olmalıdır.
