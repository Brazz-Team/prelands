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
Содержит хранилища (store) для flow, игры, продукта, devtools. Все данные сохраняются в localStorage с уникальным ключом для каждого лендинга.

- **flowStore** — хранит текущий flow и step, методы:
  - `GetCurrentFlow()`, `GetCurrentStep()`, `UpdateCurrentFlow(flow)`, `UpdateCurrentStep(step)`

Примеры
- **gameStore** — хранит состояние мини-игры (например, открытые коробки).
- **productStore** — хранит выбранные параметры товара (цвет, размер и т.д.).
- **devtoolsStore** — настройки для отладки (видимость devtools, ускорение анимаций).
- **ResetAll(landing_id)** — сбросить все данные лендинга в localStorage.

#### Пример:
```js
const currentFlow = land_store.flowStore.GetCurrentFlow();
const currentStep = land_store.flowStore.GetCurrentStep();
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

### Пример: взаимодействие между модулями
```js
// mainSection.js
mainSection.addEventListener('opened', () => {
  // логика секции main после открытия
});
```

---

## Вложенность и композиция модулей

- Каждый лендинг содержит как минимум один основной flow (обычно `main`).
- Секции (section) могут содержать другие модули (например, секция main содержит блоки FAQ и Comments).
- Модули могут быть как самостоятельными секциями, так и частью других секций.

#### Пример вложенности:
```html
<section id="main">
  ...
  <div id="comments">...</div>
  <div id="faq">...</div>
</section>
```

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
  land_tools.js
  land_store.js
  land_devtools.js
  ...
```

---

## Синхронизация между модулями
Бывает ситуация, например как с корзиной, куда попадают какие-то данные из main, например, назвние товара, старая цена, новая цена
и если есть выбор варианта товара, нужно реализовывать так же, как это реализовано в секции main

Пример смотреть в m_cartAlcatras.js, там есть querySelector-ы, которые сейчас гвоздями забиты, вообще хотелось бы конечно избавиться от этого, буду думать

P.S. Просто не придумал пока как это можно отшить друг от друга
Пока только идея была сделать какой-то общий store, куда могут обращаться все модули и брать оттуда данные по ключам, которые должны быть заранее обговорены (старая цена, новая цена, основная картинка товара и тд), и уже этот store каждый модуль может сам дополнить или изменять, так можно полностью изолировать некоторые модули от конкретной реализации main

---

## Контакты и поддержка
По вопросам архитектуры и доработок — обращайтесь к разработчику платформы @suchimauz (tg).
