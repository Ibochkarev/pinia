# Плагины

Хранилища Pinia могут быть полностью расширены благодаря API низкого уровня. Вот список того, что вы можете сделать:

-   Добавлять новые свойства к хранилищам
-   Добавлять новые опции при определении хранилищ
-   Добавлять новые методы к хранилищам
-   Обернуть существующие методы
-   Перехватывать действия и их результаты
-   Реализовать side effects, такие как [Local Storage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)
-   Применять **только** к определенным хранилищам

Плагины добавляются к экземпляру pinia с помощью `pinia.use()`. Простейший пример - добавление статического свойства ко всем хранилищам путем возврата объекта:

```js
import { createPinia } from 'pinia'

// добавьте свойство `secret` в каждое создаваемое хранилище
// после установки плагина это свойство может находиться в другом файле
function SecretPiniaPlugin() {
    return { secret: 'the cake is a lie' }
}

const pinia = createPinia()
// предоставить плагину pinia
pinia.use(SecretPiniaPlugin)

// в другом файле
const store = useStore()
store.secret // 'the cake is a lie'
```

Это полезно для добавления глобальных объектов, таких как маршрутизатор, модальный менеджер или менеджер тостов.

## Введение

Плагин Pinia - это функция, которая по желанию возвращает свойства для добавления в хранилище. Она принимает один необязательный аргумент, _context_:

```js
export function myPiniaPlugin(context) {
    context.pinia // pinia, созданная с помощью `createPinia()`.
    context.app // текущее приложение, созданное с помощью `createApp()` (только для Vue 3)
    context.store // хранилище, которое дополняет плагин
    context.options // объект опций, определяющий хранилизе, переданный в `defineStore()`.
    // ...
}
```

Затем эта функция передается в `pinia` с помощью `pinia.use()`:

```js
pinia.use(myPiniaPlugin)
```

Плагины применяются только к хранилищам, созданным **после самих плагинов и после передачи `pinia` в приложение**, иначе они не будут применены.

## Дополнение хранилища

Вы можете добавлять свойства в каждое хранилище, просто возвращая их объект в плагине:

```js
pinia.use(() => ({ hello: 'world' }))
```

Вы также можете установить свойство непосредственно в `store`, но **по возможности используйте возвращаемую версию, чтобы они могли автоматически отслеживаться devtools**:

```js
pinia.use(({ store }) => {
    store.hello = 'world'
})
```

Any property _returned_ by a plugin will be automatically tracked by devtools so in order to make `hello` visible in devtools, make sure to add it to `store._customProperties` **in dev mode only** if you want to debug it in devtools:

```js
// из примера выше
pinia.use(({ store }) => {
    store.hello = 'world'
    // убедитесь, что ваш бандлер обрабатывает это. Webpack и vite должны делать это по умолчанию
    if (process.env.NODE_ENV === 'development') {
        // добавьте все ключи, которые вы установили в хранилище
        store._customProperties.add('hello')
    }
})
```

Note that every store is wrapped with [`reactive`](https://v3.vuejs.org/api/basic-reactivity.html#reactive), automatically unwrapping any Ref (`ref()`, `computed()`, ...) it contains:

```js
const sharedRef = ref('shared')
pinia.use(({ store }) => {
    // каждое хранилище имеет свое индивидуальное свойство `hello`.
    store.hello = ref('secret')
    // он автоматически разворачивается
    store.hello // 'secret'

    // все хранилища совместно используют свойство `shared`.
    store.shared = sharedRef
    store.shared // 'shared'
})
```

Вот почему вы можете получить доступ ко всем вычисляемым свойствам без `.value` и почему они реактивны.

### Добавление нового состояния

Если вы хотите добавить новые свойства состояния в хранилище или свойства, которые будут использоваться во время гидратации, **вам придется добавить их в двух местах**:

-   В `store`, чтобы вы могли получить к нему доступ с помощью `store.myState`.
-   В `store.$state`, чтобы его можно было использовать в devtools и **сериализовать во время SSR**.

Кроме того, вам, конечно же, придется использовать `ref()` (или другой реактивный API), чтобы разделить значение между различными доступами:

```js
import { toRef, ref } from 'vue'

pinia.use(({ store }) => {
    // чтобы правильно работать с SSR, нам нужно убедиться, что мы не переопределяем
    // существующее значение
    if (!Object.prototype.hasOwnProperty(store.$state, 'hasError')) {
        // hasError определяется внутри плагина, поэтому каждое хранилище имеет свое индивидуальное
        // свойство состояния
        const hasError = ref(false)
        // установка переменной на `$state`, позволяет сериализовать ее во время SSR
        store.$state.hasError = hasError
    }
    // нам нужно передать ссылку из состояния в хранилище, таким образом.
    // оба доступа: store.hasError и store.$state.hasError будут работать
    // и использовать одну и ту же переменную
    // См. https://vuejs.org/api/reactivity-utilities.html#toref
    store.hasError = toRef(store.$state, 'hasError')

    // в этом случае лучше не возвращать `hasError`, так как он
    // будет отображаться в разделе `state` в devtools
    // в любом случае, и если мы вернем его, devtools отобразит его дважды.
})
```

Обратите внимание, что изменения состояния или добавления, которые происходят внутри плагина (что включает вызов `store.$patch()`), происходят до того, как хранилище становится активным, и поэтому **не вызывают никаких подписок**.

:::warning
Если вы используете **Vue 2**, на Pinia распространяются [те же предостережения о реактивности](https://v2.vuejs.org/v2/guide/reactivity.html#Change-Detection-Caveats), что и на Vue. Вам нужно будет использовать `Vue.set()` (Vue 2.7) или `set()` (из `@vue/composition-api` для Vue <2.7) для создания новых свойств состояния, таких как `secret` и `hasError`:

```js
import { set, toRef } from '@vue/composition-api'
pinia.use(({ store }) => {
    if (!Object.prototype.hasOwnProperty(store.$state, 'hello')) {
        const secretRef = ref('secret')
        // Если данные предназначены для использования во время SSR, вам следует
        // установить его в свойстве `$state`, чтобы он был сериализован и
        // подхватывается во время гидратации
        set(store.$state, 'secret', secretRef)
    }
    // также установить его непосредственно в хранилище, чтобы вы могли получить к нему доступ
    // обоими способами: `store.$state.secret` / `store.secret`.
    set(store, 'secret', toRef(store.$state, 'secret'))
    store.secret // 'secret'
})
```

:::

## Добавление новых внешних свойств

При добавлении внешних свойств, экземпляров классов, пришедших из других библиотек, или просто вещей, которые не являются реактивными, вы должны обернуть объект с `markRaw()` перед передачей его pinia. Вот пример добавления маршрутизатора в каждое хранилище:

```js
import { markRaw } from 'vue'
// адаптируйте это в зависимости от того, где находится ваш маршрутизатор
import { router } from './router'

pinia.use(({ store }) => {
    store.router = markRaw(router)
})
```

## Вызов `$subscribe` внутри плагинов

Вы можете использовать [store.$subscribe](./state.md#subscribing-to-the-state) и [store.$onAction](./actions.md#subscribing-to-actions) внутри плагинов:

```ts
pinia.use(({ store }) => {
    store.$subscribe(() => {
        // реагировать на изменения в хранилище
    })
    store.$onAction(() => {
        // реагировать на экшен хранилища
    })
})
```

## Добавление новых опций

Можно создавать новые опции при определении хранилищ, чтобы впоследствии использовать их из плагинов. Например, вы можете создать опцию `debounce`, которая позволит вам отменить любое действие:

```js
defineStore('search', {
    actions: {
        searchContacts() {
            // ...
        },
    },

    // это будет прочитано плагином позже
    debounce: {
        // отложить действие searchContacts на 300 мс
        searchContacts: 300,
    },
})
```

Затем плагин может считать эту опцию для обертывания действий и заменить оригинальные действия:

```js
// используйте любую библиотеку debounce
import debounce from 'lodash/debounce'

pinia.use(({ options, store }) => {
    if (options.debounce) {
        // мы переопределяем экшен новыми действиями
        return Object.keys(options.debounce).reduce(
            (debouncedActions, action) => {
                debouncedActions[action] = debounce(
                    store[action],
                    options.debounce[action]
                )
                return debouncedActions
            },
            {}
        )
    }
})
```

Обратите внимание, что пользовательские параметры передаются в качестве 3-го аргумента при использовании синтаксиса setup:

```js
defineStore(
    'search',
    () => {
        // ...
    },
    {
        // это будет прочитано плагином позже
        debounce: {
            // debounce действие SearchContacts на 300 мс
            searchContacts: 300,
        },
    }
)
```

## TypeScript

Все, что показано выше, может быть сделано с поддержкой типизации, поэтому вам никогда не понадобится использовать `any` или `@ts-ignore`.

### Типизация плагинов

Плагин Pinia может быть типизирован следующим образом:

```ts
import { PiniaPluginContext } from 'pinia'

export function myPiniaPlugin(context: PiniaPluginContext) {
    // ...
}
```

### Ввод новых свойств хранилища

При добавлении новых свойств в хранилища, вы также должны расширить интерфейс `PiniaCustomProperties`.

```ts
import 'pinia'
import type { Router } from 'vue-router'

declare module 'pinia' {
    export interface PiniaCustomProperties {
        // используя сеттер, мы можем разрешить как строки, так и рефы
        set hello(value: string | Ref<string>)
        get hello(): string

        // вы можете определить и более простые значения
        simpleNumber: number

        // введите маршрутизатор, добавленный плагином выше (#adding-new-external-properties)
        router: Router
    }
}
```

После этого его можно безопасно записывать и читать:

```ts
pinia.use(({ store }) => {
    store.hello = 'Hola'
    store.hello = ref('Hola')

    store.simpleNumber = Math.random()
    // @ts-expect-error: мы не напечатали это правильно
    store.simpleNumber = ref(Math.random())
})
```

`PiniaCustomProperties` - это общий тип, который позволяет вам ссылаться на свойства хранилища. Представьте следующий пример, где мы копируем начальные опции в `$options` (это будет работать только для хранилищ опций):

```ts
pinia.use(({ options }) => ({ $options: options }))
```

Мы можем правильно напечатать это, используя 4 общих типа `PiniaCustomProperties`:

```ts
import 'pinia'

declare module 'pinia' {
    export interface PiniaCustomProperties<Id, S, G, A> {
        $options: {
            id: Id
            state?: () => S
            getters?: G
            actions?: A
        }
    }
}
```

:::tip
При расширении типов в generics, они должны быть названы **точно так же, как в исходном коде**. `Id` не может быть назван `id` или `I`, а `S` не может быть назван `State`. Вот что означает каждая буква:

-   S: Состояние
-   G: Геттеры
-   A: Экшены
-   SS: Setup Store / Store

:::

### Типизация нового состояния

При добавлении новых свойств состояния (как в `store`, так и в `store.$state`), вам нужно добавить тип в `PiniaCustomStateProperties`. В отличие от `PiniaCustomProperties`, он получает только тип `State`:

```ts
import 'pinia'

declare module 'pinia' {
    export interface PiniaCustomStateProperties<S> {
        hello: string
    }
}
```

### Типизация новых опций создания

При создании новых опций для `defineStore()`, вы должны расширить `DefineStoreOptionsBase`. В отличие от `PiniaCustomProperties`, она раскрывает только два дженерика: State и тип Store, позволяя вам ограничить то, что может быть определено. Например, вы можете использовать имена действий:

```ts
import 'pinia'

declare module 'pinia' {
    export interface DefineStoreOptionsBase<S, Store> {
        // позволяют определить количество мс для любого из экшенов
        debounce?: Partial<Record<keyof StoreActions<Store>, number>>
    }
}
```

:::tip
Существует также тип `StoreGetters` для извлечения _геттеров_ из типа Store. Вы также можете расширить опции _setup stores_ или _option stores_ **только**, расширив типы `DefineStoreOptions` и `DefineSetupStoreOptions` соответственно.
:::

## Nuxt.js

При [использовании pinia вместе с Nuxt](../ssr/nuxt.md) вам придется сначала создать [Nuxt plugin](https://nuxtjs.org/docs/2.x/directory-structure/plugins). Это даст вам доступ к экземпляру `pinia`:

```ts
// plugins/myPiniaPlugin.ts
import { PiniaPluginContext } from 'pinia'

function MyPiniaPlugin({ store }: PiniaPluginContext) {
    store.$subscribe((mutation) => {
        // реагировать на изменения в хранилище
        console.log(`[🍍 ${mutation.storeId}]: ${mutation.type}.`)
    })

    // Обратите внимание, что это должно быть набрано, если вы используете TS
    return { creationTime: new Date() }
}

export default defineNuxtPlugin(({ $pinia }) => {
    $pinia.use(MyPiniaPlugin)
})
```

Обратите внимание, что в примере выше используется TypeScript, вам нужно удалить аннотации типов `PiniaPluginContext` и `Plugin`, а также их импорт, если вы используете файл `.js`.

### Nuxt.js 2

Если вы используете Nuxt.js 2, типы немного отличаются:

```ts
// plugins/myPiniaPlugin.ts
import { PiniaPluginContext } from 'pinia'
import { Plugin } from '@nuxt/types'

function MyPiniaPlugin({ store }: PiniaPluginContext) {
    store.$subscribe((mutation) => {
        // реагировать на изменения в хранилище
        console.log(`[🍍 ${mutation.storeId}]: ${mutation.type}.`)
    })

    // Обратите внимание, что это должно быть набрано, если вы используете TS
    return { creationTime: new Date() }
}

const myPlugin: Plugin = ({ $pinia }) => {
    $pinia.use(MyPiniaPlugin)
}

export default myPlugin
```
