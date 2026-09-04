# Shop View

## 1. Amaç

`ShopView`, oyuncuya satın alınabilir veya kazanılabilir içeriklerin sunulduğu UI modülüdür.

Bu dokümanın amacı:

* Shop UI sorumluluklarını tanımlamak
* Product ve Offer verilerinin nereden geldiğini belirlemek
* Economy ile Shop arasındaki sınırı netleştirmek
* Monetization ile Shop arasındaki sınırı netleştirmek
* Ürün fiyatlarının UI tarafından hesaplanmasını engellemek
* Purchase akışının UI'dan bağımsız olmasını sağlamak
* Shop asset'lerinin configuration ve Asset Management ile ilişkisini tanımlamak
* Farklı oyun içeriklerine uyarlanabilir generic bir Shop mimarisi oluşturmak

`ShopView` bir **presentation layer** bileşenidir.

Shop state'inin, economy state'inin veya satın alma işleminin sahibi değildir.

---

# 2. Temel Mimari

Shop için temel akış:

```text id="g3p7a2"
Shop Configuration
       ↓
Product / Offer Definitions
       ↓
Shop System
       ↓
Available Offers
       ↓
ShopView
       ↓
Product Card
```

Satın alma akışı:

```text id="n8k4q1"
User
 ↓
ShopView
 ↓
Purchase Intent
 ↓
MonetizationSystem / Purchase Owner
 ↓
Purchase Result
 ↓
Relevant System
 ↓
Reward / Economy / Progression
 ↓
ShopView
```

UI satın alma işlemini gerçekleştiren katman değildir.

---

# 3. ShopView Sorumlulukları

`ShopView` aşağıdaki sorumluluklara sahip olabilir:

* Shop ekranını açmak
* Shop ekranını kapatmak
* Ürünleri listelemek
* Product Card'ları göstermek
* Ürün adı göstermek
* Ürün ikonunu göstermek
* Ürün açıklamasını göstermek
* Ürün fiyatını göstermek
* Reward preview göstermek
* Locked / unavailable durumunu göstermek
* Purchase butonunu göstermek
* Kullanıcı satın alma niyetini ilgili sisteme iletmek
* Purchase sonucunu presentation olarak göstermek
* Shop state değişikliklerini UI'a yansıtmak

`ShopView` aşağıdakileri yapmamalıdır:

* Currency azaltmak
* Reward vermek
* IAP SDK çağırmak
* Advertisement SDK çağırmak
* Ürün fiyatı hesaplamak
* Purchase validation yapmak
* Satın alma sonucunu kendi başına başarılı kabul etmek
* Save data'yı doğrudan değiştirmek
* Monetization provider detaylarını bilmek

---

# 4. Shop ile Monetization Ayrımı

Shop ve Monetization aynı şey değildir.

Shop:

```text id="d9f2m1"
WHAT CAN BE PURCHASED?
```

sorusuna cevap verir.

Monetization:

```text id="k5t8v3"
HOW IS THE PURCHASE EXECUTED?
```

sorusuna cevap verir.

Örneğin:

```text id="z1h7cx"
ShopView
    ↓
Offer: Starter Bundle
    ↓
Purchase Intent
    ↓
MonetizationSystem
    ↓
IAP Provider
```

ShopView'in Apple, Google veya başka bir payment provider'ın API'sini bilmesine gerek yoktur.

---

# 5. Economy ile Ayrım

EconomySystem oyuncunun oyun içi currency state'inin sahibidir.

Örneğin:

```text id="a4s6de"
EconomySystem
    ↓
Soft Currency
Hard Currency
Energy
Tokens
```

Shop ise bu currency'lerin kullanıldığı ürünleri gösterebilir.

Örneğin:

```text id="x6m2pq"
ShopView
    ↓
Offer
    ↓
Price
    ↓
EconomySystem
```

Satın alma sonucunda currency değişimi gerekiyorsa:

```text id="b8n3zr"
Purchase Success
       ↓
Reward / Economy Logic
       ↓
EconomySystem
       ↓
Currency State Changed
       ↓
UI Event
```

şeklinde ilerlemelidir.

UI:

```csharp id="w2e7ka"
currency -= price;
```

yapmamalıdır.

---

# 6. Product ve Offer Ayrımı

Shop mimarisinde `Product` ve `Offer` kavramları gerektiğinde ayrılabilir.

Örneğin:

```text id="u5r1cx"
Product
    ↓
Base purchasable content

Offer
    ↓
Product + Price + Availability + Presentation
```

Örneğin aynı ürün farklı offer'larda bulunabilir:

```text id="q8p4nd"
Product A
   ├── Normal Offer
   ├── Discount Offer
   └── Starter Offer
```

Ancak proje basitse bu ayrımı zorunlu hale getirmemek gerekir.

Tek bir `ShopItemDefinition` yapısı yeterliyse gereksiz abstraction oluşturulmamalıdır.

---

# 7. Configuration

Designer tarafından değiştirilebilen shop değerleri configuration üzerinden yönetilmelidir.

Örneğin:

```text id="r3y7kc"
ShopConfig
    ↓
Offers
    ↓
Product Definitions
```

Configuration içinde örneğin:

* Product ID
* Display name
* Icon
* Description
* Reward definition
* Price definition
* Availability configuration
* Purchase type
* Optional visual metadata

bulunabilir.

Ancak runtime state configuration üzerinde tutulmamalıdır.

---

# 8. ScriptableObject Kullanımı

Shop configuration için ScriptableObject uygun olabilir.

Örneğin:

```csharp id="m4v9xs"
[CreateAssetMenu]
public class ShopConfig : ScriptableObject
{
    public ShopOfferDefinition[] Offers;
}
```

Bu yapı static/designer-authored data içindir.

Şunlar ScriptableObject üzerinde runtime mutable state olarak tutulmamalıdır:

```text id="k7c2qa"
PlayerHasPurchased
PlayerCurrency
RemainingPurchases
CurrentOwnedAmount
```

Bunlar runtime sistemlerin sorumluluğundadır.

---

# 9. Price Source of Truth

Fiyatın source of truth'u purchase türüne göre değişebilir.

Örneğin oyun içi currency ile:

```text id="j6w1zf"
OfferConfig
    ↓
Price Definition
    ↓
EconomySystem
```

Gerçek para ile:

```text id="p3x8vb"
Product ID
    ↓
MonetizationSystem
    ↓
Store / IAP Provider
    ↓
Localized Store Price
```

Özellikle gerçek para ürünlerinde UI'ın kendi fiyatını hesaplaması veya hard-code etmesi doğru değildir.

Store tarafından sağlanan fiyat localization, currency ve bölge kuralları nedeniyle runtime'da farklı olabilir.

---

# 10. Purchase Intent

ShopView yalnızca satın alma niyetini iletmelidir.

Örneğin:

```text id="c9m2yt"
Product Card
    ↓
Buy Button
    ↓
Purchase Intent(ProductId)
    ↓
Purchase Owner
```

UI:

```csharp id="n7q4pe"
OnBuyClicked(productId);
```

gibi bir intent üretebilir.

Ancak:

```csharp id="v5h8rd"
IAPManager.Buy(productId);
```

gibi provider-specific çağrıları doğrudan UI içinde yapmak tercih edilmez.

---

# 11. Purchase Lifecycle

Satın alma işlemi UI'ın düşündüğü kadar basit olmayabilir.

Temel lifecycle:

```text id="e2k6zs"
Idle
 ↓
Purchase Requested
 ↓
Pending
 ↓
Success / Failure / Cancelled
```

UI bu durumları presentation olarak yansıtabilir.

Örneğin:

```text id="f4r8ma"
Purchase Requested
    ↓
Disable Buy Button
    ↓
Show Pending
```

Sonuç:

```text id="q2n6wv"
Success
    ↓
Show Success Feedback
```

veya:

```text id="s8d3kx"
Failure
    ↓
Restore UI
    ↓
Show Error
```

Purchase'ın gerçekten tamamlandığına karar veren katman UI değildir.

---

# 12. Idempotency

Purchase sistemi duplicate purchase request'lerine karşı güvenli olmalıdır.

UI tarafında:

```text id="y5v1cq"
Buy Button
    ↓
Pending
    ↓
Disable / Prevent Duplicate Request
```

yapılabilir.

Ancak sistem tarafında da purchase lifecycle güvenli olmalıdır.

UI'ın bir butonu disable etmesi tek başına transaction safety sağlamaz.

---

# 13. Purchase Result

Purchase başarıyla tamamlandığında UI yalnızca sonucu gösterir.

Örneğin:

```text id="a9m4fp"
Purchase Success
       ↓
Reward Granted
       ↓
Economy / Progression State Changed
       ↓
Events
       ↓
ShopView
       ↓
Refresh
```

UI'ın:

```text
Purchase Success
       ↓
Give Reward
```

yapması yanlıştır.

Reward'ın sahibi ilgili gameplay/system katmanıdır.

---

# 14. Reward Ownership

Bir shop ürünü reward içeriyorsa reward'ın uygulanması ShopView'e ait değildir.

Örneğin:

```text id="w7z2hr"
Starter Bundle
    ↓
100 Currency
10 Energy
1 Booster
```

Purchase sonucu:

```text id="p6d8ka"
Purchase Success
       ↓
RewardSystem
       ├── EconomySystem
       ├── EnergySystem
       └── InventorySystem
```

gibi ilerleyebilir.

ShopView yalnızca reward preview gösterebilir.

---

# 15. Reward Preview

Product Card üzerinde:

```text id="v8m3xs"
+100 Currency
+10 Energy
+1 Booster
```

gösterilebilir.

Bu değerlerin source of truth'u configuration veya reward definition olmalıdır.

UI reward değerlerini kendi içinde hesaplamamalıdır.

Örneğin:

```csharp id="b4k7zn"
rewardAmount = price * 10;
```

gibi business logic UI içinde bulunmamalıdır.

---

# 16. Purchase Availability

Bir offer'ın satın alınabilir olup olmadığı sistem tarafından belirlenmelidir.

Örneğin:

```text id="r9t2cw"
Offer
 ↓
Availability Rules
 ↓
Available / Locked / Sold Out / Expired
 ↓
ShopView
```

ShopView bu sonucu görsel olarak sunar.

UI kendi başına:

```csharp id="j3m8qa"
if (playerLevel >= 5)
```

gibi oyun kuralı hesaplamamalıdır.

Availability kuralı başka sistemlere aitse ShopSystem veya ilgili feature owner tarafından sağlanmalıdır.

---

# 17. Limited Offers

Zaman sınırlı offer varsa zaman hesabının sahibi ShopSystem veya ilgili feature system olmalıdır.

UI:

```text id="t6x1vk"
Remaining Time
```

gösterebilir.

Ancak UI kendi başına offer'ın geçerlilik kararını vermemelidir.

Örneğin:

```csharp id="h8p3md"
if (DateTime.Now > offerEndTime)
{
    HideOffer();
}
```

gibi business rule'ların UI'a dağılması tercih edilmez.

Doğru:

```text id="u4q7sn"
Offer System
    ↓
Availability State
    ↓
ShopView
```

Countdown yalnızca presentation için UI'da çalışabilir.

---

# 18. Shop Refresh

Shop state değiştiğinde ShopView güncellenebilmelidir.

Örneğin:

```text id="c5y9rd"
ShopSystem
    ↓
ShopStateChanged
    ↓
ShopView
    ↓
Refresh
```

Ancak her frame:

```csharp id="n6k2ws"
Update()
{
    RefreshShop();
}
```

yapılmamalıdır.

Event-driven veya explicit refresh tercih edilmelidir.

---

# 19. Shop Açılışı

Shop açıldığında:

```text id="p8v4la"
Open Shop
    ↓
Read Current Shop State
    ↓
Refresh
    ↓
Show
```

uygulanmalıdır.

UI yalnızca event bekleyerek kendisini güncel tutmamalıdır.

Özellikle popup veya view disabled durumdayken state değişmiş olabilir.

---

# 20. Product Card

Shop item'ları ayrı presentation component'leri ile temsil edilebilir.

Örneğin:

```text id="q1r6mc"
ShopView
├── ProductCard
├── ProductCard
├── ProductCard
└── ProductCard
```

`ProductCard` aşağıdakileri gösterebilir:

```text id="v3k7xa"
Icon
Title
Description
Reward Preview
Price
Purchase Button
Availability State
```

ProductCard purchase logic'in sahibi değildir.

---

# 21. Product Card Pooling

Shop'taki ürün sayısı sık değişiyorsa veya çok sayıda card oluşturuluyorsa pooling kullanılabilir.

Örneğin:

```text id="m8z2pf"
Shop Offer List
    ↓
Product Card Pool
    ↓
Visible Cards
```

Pooling gerekiyorsa `ProductCard` reset sırasında:

* Text
* Icon
* Price
* Reward preview
* Button state
* Locked state
* Pending state
* Event subscriptions
* Animation state

tamamen temizlenmelidir.

Az sayıda ve statik ürün varsa pooling zorunlu değildir.

---

# 22. UI State

ShopView presentation state tutabilir.

Örneğin:

```text id="k4s8nd"
IsVisible
IsPurchasePending
SelectedTab
CurrentPresentationState
```

Ancak bunlar gameplay/economy state değildir.

Örneğin:

```text id="y7c1hm"
ShopView.IsPurchasePending
```

ile:

```text id="w3f6qa"
PurchaseSystem.IsPending
```

aynı şey değildir.

Gerçek purchase lifecycle'ın source of truth'u purchase owner'dır.

UI state yalnızca presentation için kullanılmalıdır.

---

# 23. Tabs ve Categories

Shop birden fazla kategoriye sahipse UI navigation ile shop content navigation ayrılmalıdır.

Örneğin:

```text id="z5q9rb"
ShopView
├── Featured
├── Currency
├── Boosters
└── Special
```

Kategori seçimi:

```text id="f6m2vx"
Category Selection
    ↓
ShopView / Shop Navigation
    ↓
Filtered Offers
```

olabilir.

Kategori filtering yalnızca basit presentation filtering ise UI'da yapılabilir.

Business rule içeren availability filtering ise ilgili system tarafından sağlanmalıdır.

---

# 24. Shop ve BottomNavigationBar

Bottom navigation'dan Shop açılabilir:

```text id="n2v7ck"
BottomNavigationBar
       ↓
Shop Intent
       ↓
Navigation Owner
       ↓
ShopView
```

Shop'un açılması tek başına GameFlow state değişikliği değildir.

Örneğin MainMenu içindeki Shop:

```text id="x8r4wp"
GameFlow = MainMenu
        ↓
UI Navigation = Shop
```

şeklinde çalışabilir.

GameFlow ile UI navigation birbirine karıştırılmamalıdır.

---

# 25. Shop ve GameFlow

Shop özel bir global game state değilse GameFlow'a yeni state eklenmemelidir.

Yanlış:

```text id="a2c9vf"
Boot
MainMenu
Shop
Gameplay
Pause
Win
Lose
```

Eğer Shop yalnızca MainMenu içindeki bir popup/view ise:

```text id="g7n5xm"
GameFlow
├── MainMenu
│    ├── Shop
│    ├── Settings
│    └── Profile
│
└── Gameplay
```

daha uygun olabilir.

GameFlow yalnızca gerçekten global lifecycle anlamına gelen state'leri yönetmelidir.

---

# 26. Monetization ve Ads

Shop ekranında reklam karşılığı reward bulunabilir.

Örneğin:

```text id="r4m8zt"
Watch Ad
    ↓
Intent
    ↓
MonetizationSystem
    ↓
Ad Provider
    ↓
Ad Result
    ↓
Reward System
```

ShopView doğrudan ad SDK çağırmamalıdır.

UI yalnızca:

```text id="k2p6vy"
Watch Ad Intent
```

üretir.

Reklamın gösterilmesi, tamamlanması ve reward eligibility kararı Monetization/Reward katmanında yönetilmelidir.

---

# 27. Analytics

Shop interaction'ları analytics'e gönderilecekse UI doğrudan analytics SDK'sını çağırmamalıdır.

Örneğin:

```text id="m5x9qb"
Shop Opened
Offer Viewed
Purchase Requested
Purchase Success
Purchase Failed
```

gibi event'ler sistem katmanından analytics'e aktarılabilir.

Özellikle purchase success event'i yalnızca UI'ın buton callback'ine göre üretilmemelidir.

Gerçek transaction sonucunu purchase owner belirlemelidir.

---

# 28. Asset Management

Shop asset'leri:

```text id="c7n2pa"
Art
 ↓
Shop Configuration
 ↓
Offer Definition
 ↓
ShopView
```

şeklinde bağlanabilir.

Örneğin:

```text id="y4f8sm"
Art/UI/Shop/Icons/Currency.png
        ↓
OfferConfig.RewardIcon
        ↓
ProductCard
```

Asset loading/lifetime kuralları `SYSTEMS/AssetManagement.md` tarafından belirlenir.

ShopView içine rastgele:

```csharp id="w9q3kd"
Resources.Load(...)
```

eklenmemelidir.

---

# 29. Serialization Safety

Shop configuration serialized data içeriyorsa aşağıdaki değişikliklere dikkat edilmelidir:

* Serialized field rename
* Field type change
* Enum value değişimi
* ScriptableObject structure değişimi
* Product ID değişimi
* Prefab reference değişimi

Serialized alanların isimleri değiştirilirken gerektiğinde:

```csharp id="p3r7xa"
[FormerlySerializedAs("oldFieldName")]
```

kullanılmalıdır.

Product ID gibi runtime/persistence tarafından kullanılan kimlikler rastgele değiştirilmemelidir.

---

# 30. Performance

Shop genellikle sürekli çalışan hot path değildir.

Yine de:

* Her frame tüm offer listesi yeniden oluşturulmamalı
* Gereksiz allocation yapılmamalı
* Product Card'lar gereksiz yere instantiate/destroy edilmemeli
* Gerektiğinde pooling kullanılmalı
* Asset'ler tekrar tekrar yüklenmemeli
* Event subscription lifecycle güvenli olmalı
* UI refresh yalnızca gerekli olduğunda yapılmalı

Özellikle scrollable shop'larda görünür item sayısı ile toplam offer sayısı ayrıştırılabilir.

---

# 31. Error Handling

Purchase failure UI tarafından yönetilen bir transaction kararı değildir.

Sistem sonucu örneğin:

```text id="v8c4nm"
Success
Failure
Cancelled
Unavailable
Pending
```

olarak sağlayabilir.

ShopView bunları presentation'a dönüştürür:

```text id="q6x1zr"
Success
    ↓
Success Feedback

Failure
    ↓
Error Feedback

Cancelled
    ↓
Restore UI

Pending
    ↓
Loading State
```

Hata mesajlarının kullanıcıya uygun text karşılığı configuration/localization üzerinden sağlanabilir.

---

# 32. Localization

Shop'taki:

* Product name
* Description
* Price presentation
* Error message
* Button text
* Availability text

localization sistemine uygun olmalıdır.

UI içinde hard-coded kullanıcı-facing string'ler mümkün olduğunca tutulmamalıdır.

Örneğin:

```text id="j9m3wc"
"Buy Now"
```

yerine localization key/configuration yaklaşımı kullanılabilir.

Ancak localization abstraction'ı yalnızca gerçekten gerekli olduğu ölçüde kullanılmalıdır.

---

# 33. Generic Template Rule

Template Shop sistemi belirli bir oyunun ekonomik modeline bağımlı olmamalıdır.

Core template içinde:

```text id="d4y8kp"
Star Pack
Kingdom Bundle
Building Pack
Decoration Chest
```

gibi belirli oyun kavramları bulunmamalıdır.

Bunun yerine:

```text id="n7q2vx"
Product
Offer
Reward
Price
Currency
Availability
Purchase
```

gibi generic kavramlar kullanılmalıdır.

Oyuna özel ürünler configuration/content layer'da tanımlanmalıdır.

---

# 34. Source of Truth

Shop mimarisinde temel sahiplik:

```text id="x3m8qa"
ShopConfig
    ↓
Static Offer Configuration

ShopSystem
    ↓
Available Offer State

EconomySystem
    ↓
Currency State

RewardSystem
    ↓
Reward Application

MonetizationSystem
    ↓
Purchase Execution

SaveSystem
    ↓
Persistence

AnalyticsSystem
    ↓
Analytics SDK

AssetManagement
    ↓
Asset Loading / Lifetime

ShopView
    ↓
Presentation
```

Temel kural:

> `ShopView` satın alma işlemini **yönetmez**, satın alma niyetini **gösterir ve iletir**.

---

# 35. Definition of Done

`ShopView` tamamlanmış sayılmadan önce:

* [ ] UI yalnızca presentation sorumluluğunda mı?
* [ ] Product/Offer configuration ile runtime state ayrılmış mı?
* [ ] Economy state ShopView dışında mı?
* [ ] Reward uygulaması ShopView dışında mı?
* [ ] Purchase işlemi ShopView dışında mı?
* [ ] IAP SDK doğrudan UI'dan çağrılmıyor mu?
* [ ] Ad SDK doğrudan UI'dan çağrılmıyor mu?
* [ ] UI `PlayerSaveData` değiştirmiyor mu?
* [ ] Purchase intent ile gerçek purchase sonucu ayrılmış mı?
* [ ] Purchase pending state güvenli mi?
* [ ] Duplicate purchase request'leri engelleniyor mu?
* [ ] Offer availability system tarafından belirleniyor mu?
* [ ] UI business rule hesaplamıyor mu?
* [ ] Reward preview configuration/source-of-truth üzerinden mi geliyor?
* [ ] Shop state değişiklikleri gerektiğinde event-driven mı?
* [ ] Gereksiz `Update()` polling kullanılmıyor mu?
* [ ] Product Card lifecycle güvenli mi?
* [ ] Pool kullanılıyorsa reset eksiksiz mi?
* [ ] Asset loading `AssetManagement.md` ile uyumlu mu?
* [ ] Analytics UI SDK çağrısı şeklinde uygulanmıyor mu?
* [ ] Localization uygun şekilde kullanılıyor mu?
* [ ] Serialization güvenliği korunuyor mu?
* [ ] GameFlow ile UI navigation birbirine karıştırılmıyor mu?
* [ ] Generic template sınırları korunuyor mu?
* [ ] Gereksiz abstraction eklenmemiş mi?
