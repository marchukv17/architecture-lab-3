# Architecture review

> Заповнює студент. Не описуйте лише назву принципу: покажіть причинно-наслідковий зв'язок між кодом, ризиком і виправленням.

| Principle | Evidence (file:line) | Practical risk | Fix |
|---|---|---|---|
| **Information hiding** | `src/checkout/CheckoutController.ts:22-27`, `src/orders/OrderService.ts:4,29-30`, `src/orders/mapPaymentStatus.ts:3-6` — структура `StripePayload` та специфічні статуси Stripe (`succeeded`, `requires_action`) протікають у контролер та замовлення | Деталі реалізації Stripe розмазані по суміжних модулях. Додавання другого провайдера або нового статусу `processing` змусить змінювати всі споживачі та підвищить ризик пропустити гілку обробки. | Інкапсулювати формат запиту та нормалізацію відповідей провайдера всередині модуля `Payments`, повертаючи назовні єдині стабільні статуси (`paid`, `processing`, `failed`). |
| **Local reasoning** | `src/shared/calculateTotal.ts:11-22` — функція читає глобальний `runtimeConfig`, мутує об'єкти вхідного масиву `lines` (`line.lineTotal = ...`) і записує в глобальний `auditLog` | Неможливо передбачити результат обчислення лише за аргументами. Мутація вхідного масиву створює приховані побічні ефекти, а unit-тести вимагають підготовки глобального стану. | Зробити `calculateTotal` чистою функцією: передавати `taxRate` і `discountPercent` явно через параметри, повертати обчислену суму без мутацій, а логування винести в шар оркестрації. |
| **Ownership** | `src/orders/OrderService.ts:15-25` — модуль замовлень напряму зчитує та перезаписує залишки у `database.stock` | `Inventory` не контролює власний інваріант (`available >= 0`). Зовнішній код може списати товар у мінус або залишити частково списані позиції при помилці посеред циклу. | Заборонити прямий запис у `database.stock` ззовні; додати в `Inventory` атомарну операцію `reserve()`, яка самостійно перевіряє наявність усіх позицій і змінює залишки. |
| **Dependency direction / Low coupling** | `src/payments/internal/StripeClient.ts:2` імпортує `src/orders/internal/orderMutations.ts`, тоді як `src/orders/OrderService.ts:4` імпортує `StripeClient.ts` (цикл `Orders <-> Payments`) | Модулі жорстко зв'язані через приватну реалізацію (`internal`). Будь-який рефакторинг внутрішньої структури `Orders` ламає `Payments` і унеможливлює незалежне тестування. | Розірвати цикл залежностей; будувати взаємодію лише через стабільний публічний контракт модуля `Payments` без імпорту з директорій `internal`. |
| **Explicit contracts** | `src/orders/OrderService.ts:7-12,29` — жорстко зашитий екземпляр `StripeClient` без абстракції платіжного провайдера | Код оформлення замовлення прив'язаний до конкретного класу `StripeClient`. Підключення другого навчального провайдера потребуватиме дублювання коду та розгалужень `if/else` у бізнес-логіці. | Визначити явний публічний контракт модуля `Payments` та інтерфейс адаптера провайдера зі стандартизованим входом і нормалізованим виходом. |

## Change impact prediction (заповнити до реалізації change request)

**Requirement:** Підтримати другий навчальний payment provider та новий нормалізований статус `processing` без зміни публічного API `CheckoutController` і зовнішньої поведінки сценаріїв `paid` та `failed`.

**Expected files:**
* `src/payments/normalizePaymentStatus.ts` — для централізації перетворення статусів провайдера в `paid`, `processing`, `failed`.
* `src/orders/OrderService.ts`, `src/orders/mapPaymentStatus.ts`, `src/notifications/mapPaymentStatus.ts`, `src/reports/mapPaymentStatus.ts` — для переведення на єдині нормалізовані статуси системи та обробки стану `processing`.
* `tests/paymentNormalization.test.ts` — нові unit-тести для перевірки нормалізації всіх статусів (включно з `processing` та невідомим статусом).

**Modules that should not change:**
* Публічний контракт `CheckoutController.checkout()` (формат запиту та відповіді для вже існуючих сценаріїв `paid` і `failed`).
* `Inventory` (`src/inventory/StockRepository.ts`) — логіка зберігання залишків не залежить від платіжних статусів.

## Actual impact (заповнити після реалізації)

**Actual files changed:**
* `src/payments/normalizePaymentStatus.ts` (створено єдину функцію нормалізації статусів із безпечним fallback для невідомих станів).
* `src/orders/OrderService.ts` (переведено на використання `normalizePaymentStatus` та додано підтримку стану `processing`).
* `src/orders/mapPaymentStatus.ts`, `src/notifications/mapPaymentStatus.ts`, `src/reports/mapPaymentStatus.ts` (прибрано дублювання правил мапінгу; функції делегують виклик у `Payments`).
* `tests/paymentNormalization.test.ts` (додано unit-тести для `paid`, `processing`, `failed` та невідомого статусу).

**Difference from prediction and explanation:**
Фактичний перелік змінених файлів повністю збігся з початковим прогнозом. Існуючі функції `mapPaymentStatus` у `Orders`, `Notifications` та `Reports` були збережені як тонкі обгортки над `normalizePaymentStatus`, щоб не зламати наявні імпорти й забезпечити безпечний рефакторинг без зміни зовнішньої поведінки `CheckoutController`.