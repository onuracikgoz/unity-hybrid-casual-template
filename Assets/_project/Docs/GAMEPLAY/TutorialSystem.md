# Tutorial System

## 1. Amaç

`TutorialSystem`, oyuncuya oyunun temel mekaniklerini, UI etkileşimlerini veya gerekli gameplay davranışlarını kontrollü şekilde tanıtmak için kullanılan sistemdir.

Bu dokümanın amacı:

* Tutorial lifecycle'ını tanımlamak
* Tutorial state ile gameplay state'i ayırmak
* Tutorial'ın gameplay sistemlerini doğrudan sahiplenmesini engellemek
* Tutorial adımlarını data-driven hale getirmek
* Tutorial completion bilgisinin nasıl saklanacağını belirlemek
* Input, UI ve gameplay arasındaki tutorial akışını tanımlamak
* Tutorial'ın farklı oyun içeriklerine uyarlanabilir olmasını sağlamak

Tutorial, oyunun gerçek gameplay kurallarının sahibi değildir.

Tutorial, mevcut gameplay sistemlerini **yönlendiren ve kısıtlayan bir orchestration layer** olarak düşünülebilir.

---

# 2. Temel Mimari

Genel akış:

```text id="x4m8qa"
Tutorial Configuration
        ↓
TutorialSystem
        ↓
Tutorial Runtime State
        ↓
Tutorial Step
        ↓
Gameplay / UI Intent
        ↓
Gameplay System
        ↓
Tutorial Progress
```

Örneğin:

```text id="p7c2vz"
Tutorial Step
    ↓
"Tap the highlighted object"
    ↓
Input Controller
    ↓
Gameplay Command
    ↓
Gameplay System
    ↓
Action Completed
    ↓
TutorialSystem
    ↓
Next Step
```

Tutorial kendi gameplay implementasyonunu yazmak yerine mevcut gameplay akışını kullanmalıdır.

---

# 3. TutorialSystem Sorumlulukları

`TutorialSystem` aşağıdaki sorumluluklara sahip olabilir:

* Tutorial'ın aktif olup olmadığını yönetmek
* Tutorial sequence başlatmak
* Tutorial step'lerini yönetmek
* Current step'i belirlemek
* Step completion kontrolünü yönetmek
* Tutorial completion state'ini yönetmek
* Tutorial progression'ı persistence katmanına bildirmek
* Gameplay ve UI'a tutorial state sağlamak
* Gerekirse gameplay input'unu tutorial kurallarına göre kısıtlamak
* Tutorial'ın ilgili level/content ile ilişkisini yönetmek

`TutorialSystem` aşağıdakilerin sahibi değildir:

* Genel gameplay state
* Board state
* Economy state
* Player progression
* Save data
* UI presentation
* Input device implementation
* Analytics SDK
* Audio implementation

---

# 4. Tutorial Configuration

Tutorial step'leri designer tarafından değiştirilebilir olmalıdır.

Örneğin:

```text id="n5r8kc"
TutorialConfig
├── TutorialId
├── Trigger
├── Steps
└── Completion Rules
```

Step:

```text id="q3m7vx"
TutorialStep
├── StepId
├── Instruction
├── Target
├── Required Action
├── Optional Highlight
├── Optional Delay
└── Completion Condition
```

Bu alanlar örnektir.

Her projede tüm alanların bulunması gerekmez.

---

# 5. ScriptableObject Kullanımı

Tutorial configuration için ScriptableObject kullanılabilir.

Örneğin:

```csharp id="f8p2wm"
[CreateAssetMenu]
public class TutorialConfig : ScriptableObject
{
    public TutorialStepDefinition[] Steps;
}
```

Bu asset:

```text
Static / Designer-authored configuration
```

olarak değerlendirilmelidir.

Tutorial'ın runtime state'i burada mutable olarak tutulmamalıdır.

Yanlış:

```text id="v6q1zb"
TutorialConfig.CurrentStep = 4
TutorialConfig.IsCompleted = true
```

Doğru:

```text id="m9c4xa"
TutorialConfig
    ↓
Static Definition

TutorialSystem
    ↓
Runtime State
```

---

# 6. Tutorial Runtime State

Tutorial'ın runtime state'i `TutorialSystem` tarafından sahiplenilmelidir.

Örneğin:

```text id="w2k7pn"
CurrentTutorialId
CurrentStepIndex
IsActive
IsCompleted
```

Bu state session veya player progression'a göre ayrıştırılabilir.

Örneğin:

```text id="z8r3mc"
Session State
    ↓
CurrentStep
IsWaitingForAction

Persistent State
    ↓
CompletedTutorials
```

Bunların persistence modeli SaveSystem ile birlikte belirlenmelidir.

---

# 7. Tutorial Completion

Tutorial completion'ın source of truth'u `TutorialSystem` veya ilgili progression owner tarafından belirlenmelidir.

Örneğin:

```text id="a4n7qx"
TutorialSystem
    ↓
Tutorial Completed
    ↓
ProgressionSystem / SaveSystem
    ↓
Persistent State
```

UI:

```text id="h6p2vz"
Player completed tutorial
```

gibi bir state'i kendi içinde saklamamalıdır.

---

# 8. Tutorial Trigger

Tutorial farklı koşullarla başlayabilir.

Örneğin:

```text id="c9m4wb"
First Launch
Level Start
Feature Unlock
Specific Gameplay Event
Explicit User Action
```

Trigger'ın kaynağı ilgili system tarafından sağlanabilir.

Örneğin:

```text id="j3x8ra"
First Launch
    ↓
Bootstrap / TutorialSystem
```

veya:

```text id="p5v2nk"
Feature Unlocked
    ↓
ProgressionSystem
    ↓
TutorialSystem
```

Tutorial trigger'ları UI içine hard-code edilmemelidir.

---

# 9. Tutorial ve Bootstrap

İlk açılış tutorial'ı varsa Bootstrap ile TutorialSystem arasında açık sınır bulunmalıdır.

Bootstrap:

```text id="r8m3yc"
Initialize Systems
    ↓
Load Persistent Data
    ↓
Systems Ready
    ↓
Hand Off
```

TutorialSystem:

```text id="v4q7kp"
Determine Tutorial Eligibility
    ↓
Start Tutorial
```

Bootstrap tutorial'ın tüm step'lerini yönetmemelidir.

---

# 10. Tutorial ve GameFlow

Tutorial global game state değildir.

Örneğin:

```text id="x7c2mv"
GameFlow
    ↓
Gameplay
```

Gameplay state'i devam ederken:

```text id="q9n4za"
TutorialSystem
    ↓
Tutorial Active
```

olabilir.

Bu nedenle çoğu durumda GameFlow'a:

```text
Tutorial
```

adında ayrı bir global state eklemek gereksizdir.

Tutorial, mevcut GameFlow state'i içinde çalışan bir gameplay/UI mode olabilir.

---

# 11. Tutorial ve LevelSystem

Level tutorial'ı level lifecycle ile ilişkilendirilebilir:

```text id="m5r8xd"
LevelSystem
    ↓
Start Level
    ↓
TutorialSystem
    ↓
Check Tutorial
    ↓
Start Tutorial if Needed
```

LevelSystem:

```text
Level Started
```

bilgisini sağlar.

TutorialSystem:

```text
Bu level için tutorial gerekiyor mu?
```

sorusunu değerlendirir.

Tutorial'ın level lifecycle'ını sahiplenmesi gerekmez.

---

# 12. Tutorial ve GameplayModule

Tutorial gerçek gameplay action'larını mevcut gameplay sistemleri üzerinden gerçekleştirmelidir.

Örneğin:

```text id="f3k7wp"
Tutorial
    ↓
"Select Object"
    ↓
Gameplay Input
    ↓
Gameplay Command
    ↓
GameplayModule
```

Tutorial kendi içinde ayrı bir object selection sistemi oluşturmamalıdır.

Aksi halde tutorial ile gerçek gameplay arasında iki farklı davranış oluşabilir.

---

# 13. Tutorial Step

Bir tutorial step tek bir kullanıcı hedefini temsil edebilir.

Örneğin:

```text id="n6v2qa"
Step 1
"Tap the object"

Step 2
"Perform the action"

Step 3
"Complete the objective"
```

Step lifecycle:

```text id="w8m4zc"
Inactive
   ↓
Active
   ↓
Waiting For Action
   ↓
Completed
   ↓
Next Step
```

---

# 14. Step Completion

Tutorial step'in tamamlanması gerçek gameplay sonucuna bağlanmalıdır.

Örneğin:

```text id="q2r7mx"
User Input
    ↓
Gameplay Action
    ↓
Gameplay System
    ↓
Action Result
    ↓
TutorialSystem
    ↓
Step Completed
```

Tutorial:

```text id="c5n8va"
Player tapped button
```

gibi yalnızca UI click'ine güvenmek yerine gerekiyorsa gerçek gameplay sonucunu beklemelidir.

Örneğin bir hareket gerçekten gerçekleşmediyse tutorial tamamlanmamalıdır.

---

# 15. Tutorial Input

Tutorial input'un sahibi değildir.

Genel input pipeline korunmalıdır:

```text id="k7p3xb"
Device Input
    ↓
Input Controller
    ↓
Gameplay Command
    ↓
Gameplay System
```

Tutorial yalnızca:

```text id="z4m8qa"
Allowed Actions
Blocked Actions
Expected Action
```

gibi kuralları sağlayabilir.

---

# 16. Input Restriction

Tutorial sırasında oyuncunun yalnızca belirli bir action'ı yapması gerekiyorsa input kısıtlaması uygulanabilir.

Örneğin:

```text id="r6c2vp"
Tutorial Step
    ↓
Expected Command = Select
```

Bu durumda diğer command'ler:

```text id="n8x4ma"
Ignored
Blocked
Deferred
```

olabilir.

Ancak input filtering mümkün olduğunca merkezi bir input/control layer üzerinden uygulanmalıdır.

Tutorial'ın farklı UI ve gameplay component'lerine ayrı ayrı:

```csharp
enabled = false;
```

yazması tercih edilmez.

---

# 17. Tutorial Target

Tutorial bir UI veya gameplay target'ını gösterebilir.

Örneğin:

```text id="p3v9ck"
Target
    ↓
Object
Button
Tile
Character
Feature
```

Target'ın çözülmesi runtime hierarchy'ye bağlıysa güvenilir bir identifier veya explicit reference kullanılmalıdır.

Tutorial kodunda:

```csharp id="m7q2xa"
GameObject.Find("SomeButton");
```

gibi runtime lookup'lar kullanılmamalıdır.

---

# 18. Highlight

Tutorial target highlight'ı presentation sorumluluğudur.

Akış:

```text id="h5n8zr"
TutorialSystem
    ↓
Target Information
    ↓
Tutorial Presentation
    ↓
Highlight
```

TutorialSystem:

```text
Target = X
```

diyebilir.

Ancak highlight'ın:

* Animation
* Visual effect
* Pointer
* Glow
* Arrow

gibi görsel detaylarının sahibi UI/presentation layer olabilir.

---

# 19. Tutorial Overlay

Tutorial overlay UI'a ait bir presentation component olabilir.

Örneğin:

```text id="x4q7mb"
TutorialSystem
    ↓
Tutorial State
    ↓
TutorialOverlay
    ├── Instruction
    ├── Highlight
    └── Continue
```

Overlay:

* Text gösterir
* Target highlight eder
* Continue button gösterir
* Tutorial intent üretir

Ancak tutorial state'in sahibi değildir.

---

# 20. Tutorial Skip

Tutorial skip destekleniyorsa:

```text id="k8m2va"
Skip Intent
    ↓
TutorialSystem
    ↓
Validate Skip
    ↓
End Tutorial
```

şeklinde ilerlemelidir.

UI:

```text id="w3p6zr"
TutorialSystem.Skip()
```

gibi bir system operation'ı çağırabilir.

Skip'in hangi durumlarda izinli olduğu TutorialSystem tarafından belirlenmelidir.

---

# 21. Tutorial Pause

Tutorial gameplay sırasında aktifken GameFlow Pause state'e geçebilir.

Örneğin:

```text id="v7c4xn"
Tutorial Active
       ↓
GameFlow = Pause
       ↓
Tutorial Temporarily Suspended
```

Resume:

```text id="m2r8qa"
GameFlow = Gameplay
       ↓
Tutorial Resume
```

Tutorial step state'i pause sırasında kaybolmamalıdır.

---

# 22. Tutorial Persistence

Tutorial completion player-specific bir state ise SaveSystem ile persistence yapılabilir.

Örneğin:

```text id="f6k3wp"
TutorialSystem
    ↓
CompletedTutorial
    ↓
SaveSystem
    ↓
PlayerSaveData
```

İlk açılış tutorial'ı için:

```text id="q8n2mv"
HasCompletedFirstLaunchTutorial
```

gibi bir state tutulabilir.

Ancak SaveSystem'in kendisi tutorial logic'ini bilmek zorunda değildir.

SaveSystem yalnızca persistence sağlar.

---

# 23. Tutorial Reset

Development ve testing sırasında tutorial reset gerekebilir.

Örneğin:

```text id="r4m7xa"
Reset Tutorial
    ↓
TutorialSystem
    ↓
Clear Tutorial Progress
```

Bu işlem production player save data'sını yanlışlıkla temizlememelidir.

Developer/debug tooling ile runtime player data birbirinden ayrılmalıdır.

---

# 24. Tutorial ve Progression

Tutorial tamamlandığında bazı content'ler unlock edilebilir.

Örneğin:

```text id="c7p2zn"
Tutorial Complete
      ↓
ProgressionSystem
      ↓
Feature Unlocked
```

TutorialSystem unlock logic'inin tamamını sahiplenmemelidir.

Tutorial yalnızca completion sonucunu üretir.

ProgressionSystem:

```text
Unlock State
```

sorumluluğunu korur.

---

# 25. Tutorial ve Reward

Bazı tutorial'lar completion reward verebilir.

Akış:

```text id="v8m3qc"
Tutorial Complete
      ↓
Reward System
      ↓
Economy / Inventory / Progression
```

Reward'ın verilmesi Tutorial UI içinde yapılmamalıdır.

Yanlış:

```csharp id="n5x7kb"
economy.AddCoins(100);
```

Doğru:

```text id="j3r9va"
Tutorial Completion
    ↓
Reward Definition
    ↓
RewardSystem
```

---

# 26. Tutorial ve Analytics

Tutorial analytics event'leri doğrudan SDK'ya gönderilmemelidir.

Örneğin:

```text id="q6v2mz"
Tutorial Started
Tutorial Step Started
Tutorial Step Completed
Tutorial Skipped
Tutorial Completed
```

gibi event'ler `AnalyticsSystem` üzerinden gönderilebilir.

TutorialSystem olayın semantiğini bilir.

AnalyticsSystem SDK entegrasyonunu bilir.

---

# 27. Tutorial ve Audio

Tutorial sırasında ses veya haptic feedback gerekiyorsa TutorialSystem doğrudan audio SDK veya haptic implementation çağırmamalıdır.

Örneğin:

```text id="m8c4xp"
Tutorial Event
    ↓
AudioSystem / HapticSystem
```

kullanılabilir.

Presentation feedback ile gameplay rule birbirinden ayrılmalıdır.

---

# 28. Tutorial Async / Timing

Tutorial step'leri arasında delay veya animation bekleme gerekiyorsa timing lifecycle kontrollü olmalıdır.

Örneğin:

```text id="w5n7qa"
Step Completed
    ↓
Optional Delay
    ↓
Next Step
```

Tutorial kapanırsa veya level değişirse bekleyen operation temizlenmelidir.

Unmanaged:

```text id="x9p3mc"
while (true)
{
    ...
}
```

veya ownership'i olmayan infinite coroutine kullanılmamalıdır.

Projenin mevcut async/timing yaklaşımı neyse onunla tutarlı olunmalıdır.

---

# 29. Tutorial Restart / Level Restart

Level restart sırasında tutorial davranışı açıkça tanımlanmalıdır.

Örneğin:

```text id="k4r8vz"
Level Restart
    ↓
TutorialSystem
    ↓
Check Tutorial Policy
```

Olası politikalar:

```text
Restart tutorial
Continue tutorial
Skip completed tutorial
Do not show tutorial again
```

Bu karar level/game design'e aittir.

UI component'leri kendi başına karar vermemelidir.

---

# 30. Tutorial State Machine

Tutorial complexity arttığında explicit state machine kullanılabilir.

Örneğin:

```text id="p7x2nb"
Inactive
   ↓
Starting
   ↓
ShowingStep
   ↓
WaitingForAction
   ↓
CompletingStep
   ↓
NextStep
   ↓
Completed
```

Ancak basit bir tutorial için ayrı bir FSM abstraction'ı zorunlu değildir.

Mevcut project architecture'da zaten uygun bir state mechanism varsa onunla entegre olunmalıdır.

---

# 31. Tutorial ile UI Navigation

Tutorial sırasında UI navigation kısıtlanabilir.

Örneğin:

```text id="c8m4ya"
Tutorial Active
    ↓
Navigation Policy
    ↓
Allowed UI Actions
```

Ancak BottomNavigationBar veya Settings gibi UI component'lerinin içine tutorial logic'i dağıtılmamalıdır.

Merkezi bir tutorial/input policy varsa oradan uygulanmalıdır.

---

# 32. Tutorial Failure

Tutorial'ın kendisi genellikle gameplay failure üretmez.

Örneğin oyuncu yanlış butona bastığında:

```text id="r5q9vk"
Wrong Action
    ↓
Tutorial Feedback
    ↓
Wait for Correct Action
```

olabilir.

Tutorial'ın:

```text
Lose
```

üretmesi ancak oyunun tasarımında gerçekten tutorial failure anlamına geliyorsa yapılmalıdır.

Global Win/Lose flow'u yine GameFlow tarafından yönetilir.

---

# 33. Tutorial Completion ve GameFlow

Tutorial tamamlandığında GameFlow otomatik olarak state değiştirmek zorunda değildir.

Örneğin:

```text id="n6p2xc"
Tutorial Complete
    ↓
Gameplay Continues
```

olabilir.

Eğer tutorial completion sonrası MainMenu'ya geçilecekse:

```text id="v3k8qa"
Tutorial Complete
    ↓
Intent / Result
    ↓
GameFlow
    ↓
MainMenu
```

şeklinde yapılmalıdır.

TutorialSystem global transition'ın sahibi olmamalıdır.

---

# 34. Generic Template Rule

TutorialSystem generic kalmalıdır.

Template core içinde:

```text id="q8m4zp"
Match-3 Move Tutorial
Kingdom Tutorial
Building Tutorial
Star Tutorial
```

gibi oyuna özel kavramlar bulunmamalıdır.

Bunun yerine:

```text id="f5r7cx"
Tutorial
Step
Target
Action
Objective
Completion
Progress
```

gibi generic kavramlar kullanılmalıdır.

Oyuna özel tutorial içeriği configuration/content layer'da tanımlanmalıdır.

---

# 35. Source of Truth

Tutorial mimarisindeki temel sahiplik:

```text id="m9x2vb"
TutorialConfig
    ↓
Static Tutorial Definition

TutorialSystem
    ↓
Runtime Tutorial State

Gameplay Systems
    ↓
Actual Gameplay State

ProgressionSystem
    ↓
Persistent Progress / Unlocks

RewardSystem
    ↓
Reward Application

SaveSystem
    ↓
Persistence

AnalyticsSystem
    ↓
Analytics SDK

Tutorial UI
    ↓
Presentation

GameFlow
    ↓
Global Game State
```

Temel kural:

> Tutorial, gameplay'i yeniden implement etmez. Mevcut gameplay'i yönlendirir, oyuncuya gösterir ve gerekli aksiyonun gerçekleşmesini bekler.

---

# 36. Definition of Done

`TutorialSystem` tamamlanmış sayılmadan önce:

* [ ] Tutorial configuration ile runtime state ayrılmış mı?
* [ ] Tutorial runtime state'in sahibi belli mi?
* [ ] Tutorial completion persistence'dan ayrılmış mı?
* [ ] ScriptableObject üzerinde mutable player state tutulmuyor mu?
* [ ] Tutorial gameplay logic'i yeniden implement etmiyor mu?
* [ ] Gameplay action'ları mevcut gameplay systems üzerinden mi ilerliyor?
* [ ] Input device logic TutorialSystem'e gömülmemiş mi?
* [ ] Tutorial input restriction merkezi ve kontrollü mü?
* [ ] Tutorial target resolution güvenli mi?
* [ ] `GameObject.Find` kullanılmıyor mu?
* [ ] Highlight ve overlay presentation layer'da mı?
* [ ] Tutorial UI state'in sahibi değil mi?
* [ ] Tutorial skip system üzerinden mi ilerliyor?
* [ ] Tutorial completion Progression/Reward/Save sistemleriyle doğru şekilde ayrılmış mı?
* [ ] Tutorial doğrudan SDK çağırmıyor mu?
* [ ] Analytics AnalyticsSystem üzerinden mi ilerliyor?
* [ ] Audio/Haptic ilgili sistemler üzerinden mi yönetiliyor?
* [ ] Async/timing operation'ların lifecycle'ı güvenli mi?
* [ ] Level restart davranışı tanımlı mı?
* [ ] Pause/resume davranışı tanımlı mı?
* [ ] Duplicate tutorial start kontrol ediliyor mu?
* [ ] Global GameFlow state ile tutorial state birbirine karıştırılmıyor mu?
* [ ] Generic template sınırları korunuyor mu?
* [ ] Gereksiz abstraction eklenmemiş mi?
* [ ] Test edilebilirlik korunuyor mu?
