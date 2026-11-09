# eqvi-clientlib

Общие компоненты Eqvi для Nuxt 2, Vue 2 и Vuetify 2.

Отдельно от `artialdev-clientlib`: та библиотека общая для всех проектов Artial Dev, и
эквишным компонентам там не место.

## Устройство

Пакет без сборки — отдаёт исходники, как и `artialdev-clientlib`:

```
lib.js              точка импорта компонентов
vue/components/     сами компоненты
```

## Подключение

Импортом в компоненте:

```js
import { CreativeProfileEditor } from 'eqvi-clientlib';
```

Сборщик должен транспилировать пакет — в `nuxt.config.js`:

```js
build: { transpile: ['eqvi-clientlib'] },
```

## Компоненты

| Компонент | Назначение |
|---|---|
| `CreativeProfileEditor` | Редактор профиля креатива по дескриптору с сервера: секции, закрытые словари, свободный ввод с автокомплитом. |
