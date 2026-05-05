[README.md](https://github.com/user-attachments/files/27407736/README.md)
# KAROLINE

**Дисциплина:** Мобилни приложения  
**Преподавател:** гл. ас. д-р Георги Пашев  
**Студент (ФН):** 2301681072  
**GitHub repo:** `MobileApps2025-2301-68-1072`

---

## 1. Идея

**KAROLINE** е мобилно приложение за модерен онлайн магазин за дамски дрехи. Потребителят разглежда колекция от облекла, разпределени в категории (Blouse, Pants, Jacket, Dress, Top, Bottoms), добавя продукти в любими и в количка с избрани размер и количество, и преминава през визуален checkout процес.

Приложението следва дизайн с pink Material 3 палитра, поддържа light и dark тема, и съхранява всички данни локално чрез Room (SQLite), така че количка и любими се запазват след рестарт на приложението.

---

## 2. Как работи

При първо стартиране Room базата се попълва автоматично с **16 предефинирани продукта** в 6 категории чрез `RoomDatabase.Callback.onCreate`. След това всички екрани четат продукти през `KarolineRepository`, който експозва `Flow` от Room DAO и се наблюдава реактивно от `ViewModel`-ите чрез `LiveData` (Observer pattern).

Сценарий на потребителя:

1. **Login екран** (UI-only) — въвежда се потребителско име/парола и се натиска стрелката за вход; не се извършва истинска автентикация (UX-само).
2. **Home екран** — показва hero банер „NEW COLLECTION", филтри (Popular / New / Sale), Sale карта с „UP TO 70% OFF" и решетка от всички продукти, заредени през `Flow<List<Product>>`.
3. **Categories екран** — pill бутони за всичките 6 категории. Селекция отваря **Category Products** с филтрирани продукти.
4. **Product Detail екран** — голяма снимка, име, цена, описание, избор на размер (Material `Chip`-ове) и количество (бутони +/-). Сърчицето в горния десен ъгъл маркира продукта като **Favorite** (Add/Remove → запис в `favorites` таблицата).
5. **Cart екран (CRUD сърцето на проекта)** — изброява добавените позиции през `INNER JOIN` между `cart_items` и `products`. Поддържа:
   - **Create** — Add to Cart от Product Detail (sums quantity ако вече съществува със същия размер)
   - **Read** — реактивно показване на списъка и общата сума
   - **Update** — бутоните +/- сменят количеството на ред в DB
   - **Delete** — бутонът × премахва позицията; ако quantity падне до 0, репозиторият също изтрива записа
6. **Favorites екран** — четеца списъка на favorited продукти през `JOIN` между `favorites` и `products`. Тап върху сърчицето в Product Detail добавя/маха.
7. **Checkout екран** (UI-only) — таб Shipping/Payments, поле за карта с CVV/expiry, чекбокс за условия. Бутонът **Finish** изчиства количката, показва Toast „Order placed successfully!" и връща потребителя към Home.

Всички данни в `cart_items` и `favorites` са с `ForeignKey` към `products` с `ON DELETE CASCADE`, което гарантира референциална цялост.

---

## 3. Архитектура

**MVVM + Repository слой** с реактивни данни през Kotlin Coroutines `Flow` → `LiveData`.

```
┌──────────────────────────────────────────────────────┐
│                  UI (Activity + XML)                 │
│  Login · Register · Home · Categories · CategoryList │
│  ProductDetail · Cart · Favorites · Checkout         │
└──────────────────┬───────────────────────────────────┘
                   │ observe LiveData
                   ▼
┌──────────────────────────────────────────────────────┐
│                    ViewModels                        │
│  Home · Category · ProductDetail · Cart · Favorites  │
└──────────────────┬───────────────────────────────────┘
                   │ Flow<...>
                   ▼
┌──────────────────────────────────────────────────────┐
│                KarolineRepository                    │
│  единствена точка за достъп до данни (single source) │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│             Room DAO + AppDatabase                   │
│  ProductDao · CartDao · FavoriteDao                  │
│  entities: Product · CartItem · Favorite             │
└──────────────────────────────────────────────────────┘
```

### Технологичен стек

| Слой | Технология | Версия |
|------|------------|--------|
| Език | Kotlin | 2.0.21 |
| Build | AGP / Kotlin DSL / KSP | 9.1.1 |
| Min SDK / Target SDK | 24 / 36 | — |
| UI | XML + ViewBinding + Material 3 | 1.13.0 |
| База данни | Room | 2.7.2 |
| Реактивност | Kotlin Coroutines + Flow + LiveData | 1.10.2 / 2.9.4 |
| Тестове | JUnit4 + Espresso + coroutines-test | 4.13.2 / 3.7.0 |

### Пакетна структура

```
com.example.karoline/
├── KarolineApp.kt              → Application клас, държи Repository singleton
├── data/
│   ├── KarolineRepository.kt   → агрегира трите DAO-та
│   └── local/
│       ├── AppDatabase.kt      → Room database + seed callback
│       ├── Product.kt          → entity (id, name, price, category, …)
│       ├── CartItem.kt         → entity + CartItemWithProduct join DTO
│       ├── Favorite.kt         → entity (productId PK)
│       ├── ProductDao.kt       → READ операции
│       ├── CartDao.kt          → пълен CRUD върху количката
│       ├── FavoriteDao.kt      → toggle на любими
│       ├── Categories.kt       → константи на категориите
│       └── SeedData.kt         → 16 предефинирани продукта
└── ui/
    ├── ViewModelFactory.kt     → споделен Factory за всички VM
    ├── auth/                   → Login + Register (UI-only)
    ├── home/                   → MainActivity + HomeViewModel
    ├── category/               → списък категории + продукти на категория
    ├── detail/                 → Product Detail + add to cart + favorite
    ├── cart/                   → Cart (CRUD) + adapter
    ├── favorites/              → списък на любимите
    ├── checkout/               → Checkout (UI-only)
    ├── common/                 → ProductAdapter (RecyclerView)
    └── util/                   → BottomNavHelper, DrawableHelper
```

---

## 4. Потребителски поток

```
LoginActivity (launcher)
    ↓ (стрелка)
MainActivity (Home)
    ↓                       ↓                    ↓
[Product card]      [bottom nav: Cart]     [View all → Categories]
    ↓                       ↓                    ↓
ProductDetailActivity   CartActivity        CategoriesActivity
    ↓ (Add to Cart)        ↓ (CRUD)             ↓ (избор)
CartActivity ←──────────────┘            CategoryProductsActivity
    ↓ (Checkout Now)                             ↓
CheckoutActivity                          ProductDetailActivity
    ↓ (Finish)
MainActivity (Toast)
```

Bottom navigation е достъпен от Home, Cart, Categories и Favorites — Favorites таб води до `FavoritesActivity` със списъка на маркираните продукти.

---

## 5. Стъпки за стартиране

### Изисквания

- Android Studio Panda (или по-нов)
- Android SDK 36
- JDK 21
- Минимум API 24 (Android 7.0) на устройството/емулатора

### Билд

```bash
git clone https://github.com/<user>/MobileApps2025-2301-68-1072.git
cd MobileApps2025-2301-68-1072
./gradlew clean build
```

или отвори проекта през **File → Open** в Android Studio и изчакай Gradle Sync.

### Стартиране

1. Стартирай емулатор (API 24+) или свържи устройство
2. **Run → Run 'app'** (Shift+F10)
3. При първо стартиране базата автоматично се попълва с 16 продукта

### Тестове

```bash
./gradlew test                       # unit тестове (host)
./gradlew connectedAndroidTest       # Espresso UI тестове (емулатор)
```

Отчетите са в `app/build/reports/`.

---

## 6. Тестови акаунти

Login екранът е **UI-only** и не валидира кредитите — всеки въведен текст (включително празни полета) пуска потребителя в приложението. Това е по дизайн, защото authentication backend е извън обхвата на оценка 5.

| Поле | Стойност (примерна) |
|------|---------------------|
| Username | `karoline_user` |
| Password | `1234` |

Регистрацията също е UI-only — формата запазва нищо локално, бутонът връща към Login.

Checkout формата приема всякакви стойности за карта/CVV/expiry — натиска се **Finish** → показва Toast „Order placed successfully!" → изчиства количката → връща към Home.

---

## 7. Скрийншотове

Скрийншотовете са в `/docs/images/` (попълват се след първи build):

- `01_login.png` — Login екран
- `02_register.png` — Create Account
- `03_home.png` — Home (банер + филтри + продукти)
- `04_categories.png` — Категории
- `05_category_products.png` — Продукти от избрана категория
- `06_product_detail.png` — Product Detail със size/quantity избор
- `07_cart.png` — Shopping Cart с CRUD контроли
- `08_favorites.png` — Favorites
- `09_checkout.png` — Checkout (UI-only)
- `10_dark_mode.png` — Dark theme

---

## 8. APK

Подписаният release APK е в `/apk/app-release.apk` (≤ 60 MB).

Генерира се през Android Studio: **Build → Generate Signed Bundle/APK → APK → release**, или с команда:

```bash
./gradlew assembleRelease
cp app/build/outputs/apk/release/app-release.apk apk/app-release.apk
```

---

## 9. Покритие на изискванията за оценка 5

| Изискване | Статус | Файл/доказателство |
|-----------|--------|--------------------|
| Kotlin, Min SDK 24, Target SDK най-нов | ✅ | `app/build.gradle.kts` |
| MVVM + Repository | ✅ | `data/KarolineRepository.kt` + `ui/*/...ViewModel.kt` |
| ≥ 2 екрана + навигация | ✅ | 9 Activity-та, Intent навигация |
| Material 3 + light/dark тема | ✅ | `values/themes.xml` + `values-night/themes.xml` |
| Локална БД (Room) | ✅ | `data/local/AppDatabase.kt` |
| Пълни CRUD през UI | ✅ | `CartDao.kt` + `CartActivity.kt` (+/-/×) |
| Данни се запазват след рестарт | ✅ | Room SQLite файл `karoline.db` |
| Observer pattern (LiveData/Flow) | ✅ | всички VM използват `Flow.asLiveData()` |
| Unit тестове | ✅ | `app/src/test/java/.../*Test.kt` (4 файла, 25+ тестa) |
| Espresso UI тест | ✅ | `app/src/androidTest/java/.../CartFlowEspressoTest.kt` |
| README | ✅ | този файл |
| APK | ✅ | `/apk/app-release.apk` |
| GitHub repo с реална история | ⏳ | при първи push |
