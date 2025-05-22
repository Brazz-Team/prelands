# Документация по архитектуре модульного лендинга

## Общая концепция

Лендинг построен по модульному принципу: каждый функциональный блок (модуль) изолирован в отдельные JS и CSS файлы, а взаимодействие между модулями и этапами реализовано через систему flow (потоков). Это позволяет легко масштабировать, переиспользовать и поддерживать код.

---

## Структура проекта

- **index.html** — основной шаблон, где подключаются все модули и системные скрипты.
- **assets/** — папка с модулями, стилями, изображениями и системными файлами.

---

## Модульный подход

### Именование файлов
- Каждый модуль имеет кодовое название: `m_<название_модуля>.<js|css|png|jpg>`
- Примеры: `m_faq.js`, `m_faq.css`, `m_comments.js`, `m_comments.css`, `m_gameCarton.js`, `m_gameCarton.css`
- Для каждого модуля создаются отдельные файлы JS и CSS, которые подключаются в `index.html`.

### Изоляция модулей
- Каждый модуль работает только со своей секцией/блоком DOM (например, секция с id `faq` для модуля FAQ).
- Стили модуля пишутся с привязкой к id секции, чтобы избежать конфликтов.
- JS-модули используют IIFE (Immediately Invoked Function Expression) для изоляции переменных.
- Если нужно взаимодействие с localStorage, его тоже изолировать в том же файле js

#### Пример: FAQ
```js
// assets/m_faq.js
((w) => {
  const land_tools = w.__landing_tools;
  const faqSection = document.getElementById("faq");
  // ...
})(window);
```
```css
/* assets/m_faq.css */
#faq .__faq-content { ... }
```

#### Пример: Comments
```js
// assets/m_comments.js
((w) => {
  const mainSection = document.getElementById("main");
  const commentsSection = document.getElementById("comments");
  // ...
})(window);
```

---

## Основной flow (поток)

### Что такое flow?
Flow — это последовательность этапов (steps), через которые проходит пользователь. Каждый flow описывается в объекте `window.__landing_settings` в index.html.

#### Пример структуры flow:
```js
window.__landing_settings = {
  flow: ["main", "check", "gameCarton", "redirect"],
  availableFlows: [
    {
      id: "main",
      steps: [
        { id: "main_open_section", command: "toggleSection", props: { id: "main", toggle: true }, run: "always" },
        { id: "main_faq_button_visible", command: "toggleItemVisibility", props: { id: "faq_activator", toggle: true }, run: "auto" },
        // ...
      ]
    },
    // ...
  ]
}
```

### Назначение flow
- Управляет этапами показа секций/модулей.
- Позволяет сохранять прогресс пользователя (через localStorage).
- Позволяет гибко настраивать сценарии показа (например, после прохождения опроса — игра, затем корзина).

---

## Системные файлы

### land_tools.js
Содержит утилиты для управления состоянием секций и flow:
- `ToggleItemVisibility(element, bool)` — показать/скрыть элемент (по атрибуту `item-visible`).
- `ToggleItemExpanding(element, bool)` — развернуть/свернуть элемент (по атрибуту `item-expand`).
- `ToggleItemActive(element, bool)` — активировать/деактивировать элемент (по атрибуту `item-active`).
- `ToggleSection(sectionId, bool)` — показать/скрыть секцию по id.
- `AnimateOut(element)` — анимировать скрытие элемента.
- `NextStep()` — перейти к следующему шагу текущего flow.
- `GetInputByName(name)` — получить input по имени.

#### Пример использования:
```js
land_tools.ToggleSection('faq', true); // Открыть секцию FAQ
land_tools.NextStep(); // Перейти к следующему шагу flow
```

### land_store.js
Содержит api хранилища (store). Все данные сохраняются в localStorage с уникальным ключом для каждого лендинга.

Примеры
- **gameCartonStore** - можно посмотреть в Base лендинге

### Как инициализировать и пользоваться своим store для модуля
```js
// Инициализация api для работы с local storage
((w) => {
  const { InitStore } = w.__landing_store;

  InitStore("gameCarton", (UpdateStore, GetStore) => {
    const sOpenedBoxes = GetStore("openedBoxes");
    const sWinningBoxIndex = GetStore("winningBoxIndex");

    let openedBoxes = sOpenedBoxes ?? [];
    let winningBoxIndex = sWinningBoxIndex ?? null;

    const GetOpenedBoxes = () => openedBoxes;
    const UpdateOpenedBoxes = (boxes) => {
      UpdateStore("openedBoxes", boxes);
      openedBoxes = boxes;
    };

    const GetWinningBoxIndex = () => winningBoxIndex;
    const UpdateWinningBoxIndex = (boxIndex) => {
      UpdateStore("winningBoxIndex", boxIndex);
      winningBoxIndex = boxIndex;
    };

    return {
      GetOpenedBoxes,
      UpdateOpenedBoxes,
      GetWinningBoxIndex,
      UpdateWinningBoxIndex,
    };
  });
})(window);
```
потом в логике js можно обращаться к данным функциям
```js
((w) => {
  const land_store = w.__landing_store;

  // Обновляем список открытых коробок в store
  land_store.gameCartonStore.UpdateOpenedBoxes([
    ...land_store.gameCartonStore.GetOpenedBoxes(),
    index,
  ]);

  // Получаем список открытых коробок
  land_store.gameCartonStore.GetOpenedBoxes()
})(window);
```

### land_devtools.js
Devtools для отладки flow и шагов:
- Открывается по Ctrl+Shift+J.
- Позволяет вручную выбрать flow и step, сбросить состояние, ускорить анимации.
- Используется только при is_debug: true.

---

## Кастомные события (Custom Events)

Для взаимодействия между модулями, секциями и системой flow активно используются кастомные события. Это позволяет реализовать реакцию на открытие/закрытие секций

### Когда используются кастомные события?
- При открытии или закрытии секций (например, для инициализации содержимого или блокировки скролла)

### Стандартные события
- `opened` — секция или модальное окно было открыто
- `closed` — секция или модальное окно было закрыто

### Как слушать события
```js
const section = document.getElementById('main');
section.addEventListener('opened', (e) => {
  // Реакция на открытие секции
  console.log('Секция открыта', e.detail);
});
section.addEventListener('closed', (e) => {
  // Реакция на закрытие секции
  console.log('Секция закрыта', e.detail);
});
```

### Как инициировать события
Внутри land_tools.js используются функции:
```js
const NewSectionOpenedEvent = (sectionId) => {
  const section = document.getElementById(sectionId);
  section.dispatchEvent(new CustomEvent("opened", { detail: { sectionId } }));
};
const NewSectionClosedEvent = (sectionId) => {
  const section = document.getElementById(sectionId);
  section.dispatchEvent(new CustomEvent("closed", { detail: { sectionId } }));
};
```
! P.S. Эти события автоматически вызывает функция toggleSection, либо command в step = toggleSection

### Пример: блокировка скролла при открытии модального окна
```js
const modal = document.getElementById('gameCarton_lose_modal');
modal.addEventListener('opened', () => {
  document.body.classList.add('__no-scroll');
});
modal.addEventListener('closed', () => {
  document.body.classList.remove('__no-scroll');
});
```

## Вложенность и композиция модулей

- Каждый лендинг содержит как минимум один основной flow (обычно `main`).
- Секции (section) могут содержать другие модули (например, секция main содержит блоки FAQ и Comments).
- Модули могут быть как самостоятельными секциями, так и частью других секций.

#### Пример вложенности:
```html
<section id="main">
  <div class="__section-content">
    ...
    <section id="comments">
      <div class="__section-content">
      ...
      </div>
    </div>

    <section id="faq">
      <div class="__section-content">
      ...
      </div>
    </div>
  </div>
</section>
```

ВАЖНО! Если это модуль, и он открывается с помощью ToggleSection, это должно быть обязательно sectiond#(id модуля)>div.__section-content, и внутри уже можно делать любую структуру

---

## Настройка flow и шагов

### Описание шагов (steps)
Каждый step — это объект с полями:
- `id` — уникальный идентификатор шага
- `command` — команда (см. ниже)
- `props` — параметры для команды
- `run` — режим выполнения ("auto", "always" или не указан)
- `store` — сохранять ли этот шаг в localStorage (по умолчанию true)

### Команды (command)
- `toggleSection` — открыть/закрыть секцию (props: `{ id, toggle }`)
- `toggleItemVisibility` — показать/скрыть элемент (props: `{ id, toggle }`)
- `redirect` — перейти по ссылке (props: `{ href }`)
- `nextFlow` — перейти к следующему flow

#### Пример step:
```js
{
  id: "main_open_section",
  command: "toggleSection",
  props: { id: "main", toggle: true },
  run: "always"
}
```

### run: auto / always
- `run: "always"` — шаг выполняется всегда при инициализации flow (например, открытие секции).
- `run: "auto"` — шаг выполняется автоматически, если он следующий после текущего.
- Если не указано — шаг выполняется только при явном переходе (NextStep).

#### Когда использовать:
- `always` — для обязательных шагов (открытие секции, показ кнопки и т.д.).
- `auto` — для автоматических переходов (например, скрыть секцию и сразу перейти к следующему шагу).

---

## Работа с localStorage и восстановление flow

- Прогресс пользователя (flow и step) сохраняется в localStorage.
- При обновлении страницы пользователь продолжает с того же места.
- Для сброса состояния используется `land_store.ResetAll(landing_id)` или devtools.

---

## Best practices
- Каждый модуль должен быть максимально изолирован (свои id, классы, стили).
- Не использовать глобальные переменные вне window.__landing_tools/store.
- Все взаимодействие между модулями — только через flow и события.
- Для сложных секций (игра, корзина) — отдельные модули с собственным store.
- Для отладки можно использовать devtools (is_debug: true).
- Каждый новый модуль по-хорошему должен иметь уникальное кодовое имя, например cartAlcatras, commentsHyber (например если будет несколько модулей комментариев)
- Любая текстовка, должна быть в index.html, в скрытом поле если надобно, но не в js

---

## Пример структуры лендинга

```
index.html
assets/
  m_faq.js
  m_faq.css
  m_comments.js
  m_comments.css
  m_gameCarton.js
  m_gameCarton.css
  m_cartAlcatras.js
  m_cartAlcatras.css
  index.js - тут логика по сути main модуля
  land_tools.js
  land_store.js
  land_devtools.js
  ...
```

---

## Синхронизация между модулями
Иногда, например, в лендинге модули могут нуждаться в данных о товаре, они же необходимы для передачи рекламодателю, для этого реализован commonStore в window.__landing_store

### Пример в index.js, перед тем как вызвать NextStep()
```js
// Тут мы вытаскиваем из дерева название товара и тд, переменные productOldPrice, productNewPrice, selectedSizeItem и тд
...

// Берем изображение товара из выбранного варианта товара
const selectedImageSrc = selectedColorItem
  .querySelector("img")
  .getAttribute("src");

const customProductName = productName.trim() + " (" + selectedColorItem.getAttribute("item-display").trim() + ")";

commonStore.SetProductName(customProductName);
commonStore.SetProductImage(selectedImageSrc);
commonStore.SetProductOldPrice(productOldPrice);
commonStore.SetProductNewPrice(productNewPrice);
// Тут массив, чтобы можно было передать несколько вариантов, например может быть цвет и размер, или еще больше
commonStore.SetProductProperties([{title: "Size", value: selectedSizeItem.innerText}]);

// И переходим к следующему шагу
land_tools.NextStep();
```

### Потом к примеру в cartAlcatras.js
```js
const land_store = window.__landing_store;
const commonStore = land_store.commonStore;

// Устаналиваем название товара в скрытое поле
const productNameInput = land_tools.GetInputByName("product_name_landings");
productNameInput.value = commonStore.GetProductName();

// Устанавливаем изображение товара в скрытое поле
const productImageInput = land_tools.GetInputByName(
  "product_image_landings"
);
productImageInput.value = commonStore.GetProductImage();

// Устанавливаем название товара
const cartAlcatrasOfferName = document.querySelector(
  "#cartAlcatrasOfferName"
);
cartAlcatrasOfferName.innerHTML = commonStore.GetProductName();

// Устанавливаем изображение товара
const cartAlcatrasCurrentPhoto = document.querySelector(
  "#cartAlcatrasCurrentPhoto"
);
cartAlcatrasCurrentPhoto.src = commonStore.GetProductImage();

// Устанавливаем свойства товара
const cartAlcatrasOfferProperties = document.querySelector(
  "#cartAlcatrasOfferProperties"
);
// Как раз проходимся по массиву с пропсами товара, и отображаем их
commonStore.GetProductProperties().forEach((property) => {
  cartAlcatrasOfferProperties.innerHTML += `<div class="offer-property">${property.title}: ${property.value}</div>`;
});

```

Таким образом корзина не знает откуда взялись эти необходимые данные в каждом лендинге, но они там есть, таким образом нам всего лишь в main модуле положить эти данные в commonStore

---

## Контакты и поддержка
По вопросам архитектуры и доработок — обращайтесь к разработчику платформы @suchimauz (tg).
