# Миграция с 0.x (v1) на v2

Начиная с версии `2.0.0-rc.4`, pinia поддерживает как Vue 2, так и Vue 3! Это означает, что все новые обновления будут применяться к этой версии 2, так что пользователи как Vue 2, так и Vue 3 смогут извлечь из нее пользу. Если вы используете Vue 3, это ничего не изменит для вас, так как вы уже использовали rc, и вы можете проверить [CHANGELOG](https://github.com/vuejs/pinia/blob/v2/packages/pinia/CHANGELOG.md) для подробного объяснения всего, что изменилось. В противном случае, **это руководство для вас**!

## Устаревшие

Давайте рассмотрим все изменения, которые вам нужно применить к своему коду. Во-первых, убедитесь, что вы уже используете последнюю версию 0.x, чтобы увидеть все изменения:

```shell
npm i 'pinia@^0.x.x'
# или с yarn
yarn add 'pinia@^0.x.x'
```

Если вы используете ESLint, подумайте об использовании [этого плагина](https://github.com/gund/eslint-plugin-deprecation), чтобы найти все устаревшие употребления. В противном случае вы сможете увидеть их в перечеркнутом виде. Вот API, которые были устаревшими и были удалены:

- `createStore()` становится `defineStore()`.
- В подписках `storeName` становится `storeId`.
- `PiniaPlugin` был переименован в `PiniaVuePlugin` (плагин Pinia для Vue 2)
- `$subscribe()` больше не принимает _boolean_ в качестве второго параметра, вместо этого передавайте объект с `detached: true`.
- Плагины Pinia больше не получают напрямую `id` хранилища. Вместо этого используйте `store.$id`.

## Кардинальные изменения

После их удаления вы можете перейти на версию v2 с:

```shell
npm i 'pinia@^2.x.x'
# или с yarn
yarn add 'pinia@^2.x.x'
```

И начните обновлять свой код.

### Общий тип хранилища

Добавлен в [2.0.0-rc.0](https://github.com/vuejs/pinia/blob/v2/packages/pinia/CHANGELOG.md#200-rc0-2021-07-28)

Замените любое использование типа `GenericStore` на `StoreGeneric`. Это новый универсальный тип хранилища, который может принимать любой тип хранилища. Если вы писали функции, использующие тип `Store` без передачи его generics (например, `Store<Id, State, Getters, Actions>`), вам также следует использовать `StoreGeneric`, поскольку тип `Store` без generics создает пустой тип хранилища.

```ts
function takeAnyStore(store: Store) {} // [!code --]
function takeAnyStore(store: StoreGeneric) {} // [!code ++]

function takeAnyStore(store: GenericStore) {} // [!code --]
function takeAnyStore(store: StoreGeneric) {} // [!code ++]
```

## `DefineStoreOptions` для плагинов

Если вы писали плагины, используя TypeScript, и расширили тип `DefineStoreOptions` для добавления пользовательских опций, вам следует переименовать его в `DefineStoreOptionsBase`. Этот тип будет применяться как к хранилищам настроек, так и к хранилищам опций.

```ts
declare module 'pinia' {
  export interface DefineStoreOptions<S, Store> { // [!code --]
  export interface DefineStoreOptionsBase<S, Store> { // [!code ++]
    debounce?: {
      [k in keyof StoreActions<Store>]?: number
    }
  }
}
```

## `PiniaStorePlugin` был переименован

Тип `PiniaStorePlugin` был переименован в `PiniaPlugin`.

```ts
import { PiniaStorePlugin } from 'pinia' // [!code --]
import { PiniaPlugin } from 'pinia' // [!code ++]

const piniaPlugin: PiniaStorePlugin = () => { // [!code --]
const piniaPlugin: PiniaPlugin = () => { // [!code ++]
  // ...
}
```

**Примечание: это изменение может быть выполнено только после обновления до последней версии Pinia без изъянов**.

## `@vue/composition-api` версия

Поскольку pinia теперь полагается на `effectScope()`, вы должны использовать по крайней мере версию `1.1.0` из `@vue/composition-api`:

```shell
npm i @vue/composition-api@latest
# или с yarn
yarn add @vue/composition-api@latest
```

## webpack 4 support

Если вы используете webpack 4 (Vue CLI использует webpack 4), вы можете столкнуться с ошибкой следующего вида:

```
ERROR  Failed to compile with 18 errors

 error  in ./node_modules/pinia/dist/pinia.mjs

Can't import the named export 'computed' from non EcmaScript module (only default export is available)
```

Это связано с модернизацией файлов dist для поддержки родных модулей ESM в Node.js. Файлы теперь используют расширение `.mjs` и `.cjs`, чтобы позволить Node извлечь из этого пользу. Чтобы решить эту проблему, у вас есть две возможности:

- Если вы используете Vue CLI 4.x, обновите свои зависимости. Это должно включать в себя исправление, приведенное ниже.

  - Если обновление невозможно, добавьте это в ваш `vue.config.js`:

            ```js
            // vue.config.js
            module.exports = {
                configureWebpack: {
                    module: {
                        rules: [
                            {
                                test: /\.mjs$/,
                                include: /node_modules/,
                                type: 'javascript/auto',
                            },
                        ],
                    },
                },
            }
            ```

- Если вы вручную управляете webpack, вам придется сообщить ему, как работать с файлами `.mjs`:

    ```js
    // webpack.config.js
    module.exports = {
        module: {
            rules: [
                {
                    test: /\.mjs$/,
                    include: /node_modules/,
                    type: 'javascript/auto',
                },
            ],
        },
    }
    ```

## Devtools

Pinia v2 больше не использует Vue Devtools v5, для него требуется Vue Devtools v6. Найдите ссылку на скачивание на [Vue Devtools documentation](https://devtools.vuejs.org/guide/installation.html#chrome) для **бета-канала** расширения.

## Nuxt

Если вы используете Nuxt, pinia теперь имеет специальный Nuxt пакет 🎉. Установите его с помощью:

```bash
npm i @pinia/nuxt
# или с yarn
yarn add @pinia/nuxt
```

Также убедитесь в **обновлении пакета `@nuxtjs/composition-api`**.

Затем адаптируйте ваш `nuxt.config.js` и ваш `tsconfig.json`, если вы используете TypeScript:

```js
// nuxt.config.js
module.exports {
  buildModules: [
    '@nuxtjs/composition-api/module',
    'pinia/nuxt', // [!code --]
    '@pinia/nuxt', // [!code ++]
  ],
}
```

```json
// tsconfig.json
{
  "types": [
    // ...
    "pinia/nuxt/types" // [!code --]
    "@pinia/nuxt" // [!code ++]
  ]
}
```

Также рекомендуется прочитать [специальный раздел Nuxt](../ssr/nuxt.md).
