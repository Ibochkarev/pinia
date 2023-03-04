# Определение хранилища

<VueSchoolLink
  href="https://vueschool.io/lessons/define-your-first-pinia-store"
  title="Узнайте, как определять и использовать хранилища в Pinia"
/>

Прежде чем погрузиться в основные концепции, нам нужно знать, что хранилище определяется с помощью `defineStore()` и что он требует **уникального** имени, передаваемого в качестве первого аргумента:

```js
import { defineStore } from 'pinia'

// Вы можете назвать возвращаемое значение `defineStore()` как угодно,
// но лучше всего использовать имя хранилища и окружить его `use`
// и `Store` (например, `useUserStore`, `useCartStore`, `useProductStore`)
// первый аргумент - это уникальный идентификатор хранилища в вашем приложении
export const useAlertsStore = defineStore('alerts', {
    // ... другие параметры
})
```

Это _name_, также называемое _id_, необходимо и используется Pinia для подключения хранилища к devtools. Именование возвращаемой функции _use..._ - это соглашение, принятое в композитах, чтобы сделать ее использование идиоматичным.

`defineStore()` принимает два различных значения для своего второго аргумента: функцию Setup или объект Options.

## Option Stores

Подобно Vue's Options API, мы также можем передать объект Options со свойствами `state`, `actions` и `getters`.

```js {2-10}
export const useCounterStore = defineStore('counter', {
    state: () => ({ count: 0, name: 'Eduardo' }),
    getters: {
        doubleCount: (state) => state.count * 2,
    },
    actions: {
        increment() {
            this.count++
        },
    },
})
```

Вы можете рассматривать `state` как `data` магазина, `getters` как `computed` свойства магазина, а `actions` как `methods`.

Опциональные хранилища должны быть интуитивно понятными и простыми для начала работы.

## Настройка хранилищ

Существует и другой возможный синтаксис для определения хранилищ состояний. Подобно [setup function](https://vuejs.org/api/composition-api-setup.html) API Vue Composition, мы можем передать функцию, которая определяет реактивные свойства и методы и возвращает объект со свойствами и методами, которые мы хотим раскрыть.

```js
export const useCounterStore = defineStore('counter', () => {
    const count = ref(0)
    const name = ref('Eduardo')
    const doubleCount = computed(() => count.value * 2)
    function increment() {
        count.value++
    }

    return { count, name, doubleCount, increment }
})
```

В _Setup Stores_:

-   `ref()`s становятся свойствами `state
-   `computed()`s становятся `getters`
-   `function()`s становятся `actions`

Хранилища настроек дают гораздо больше гибкости, чем [Option Stores](#option-stores), поскольку вы можете создавать наблюдатели внутри магазина и свободно использовать любые [composable](https://vuejs.org/guide/reusability/composables.html#composables). Однако помните, что использование composables усложняется при использовании [SSR](../cookbook/composables.md).

## Какой синтаксис мне выбрать?

Как и в случае с [Vue's Composition API и Options API](https://vuejs.org/guide/introduction.html#which-to-choose), выбирайте тот, с которым вам удобнее всего работать. Если вы не уверены, попробуйте сначала [Option Stores](#option-stores).

## Использование хранилища

Мы _определяем_ магазин, потому что хранилище не будет создан, пока `use...Store()` не будет вызван внутри компонента `<script setup>` (или внутри `setup()` **как все композиты**):

```vue
<script setup>
import { useCounterStore } from '@/stores/counter'

// доступ к переменной `store` в любом месте компонента ✨
const store = useCounterStore()
</script>
```

:::tip
Если вы еще не используете компоненты `setup`, [вы все еще можете использовать Pinia с _map helpers_](../cookbook/options-api.md).:::

Вы можете определить столько хранилищ, сколько захотите, и **вы должны определить каждое хранилище в отдельном файле**, чтобы получить максимальную отдачу от Pinia (например, автоматически позволить вашему бандлеру разделять код и обеспечивать TypeScript-интерпретацию).

Как только хранилище создано, вы можете получить доступ к любому свойству, определенному в `state`, `getters` и `actions` непосредственно в хранилище. Мы рассмотрим их подробно на следующих страницах, но автозаполнение поможет вам.

Обратите внимание, что `store` - это объект, обернутый `reactive`, что означает отсутствие необходимости писать `.value` после геттеров, но, как и `props` в `setup`, **мы не можем его деструктурировать**:

```vue
<script setup>
const store = useCounterStore()
// ❌ Это не будет работать, потому что нарушает реактивность.
// это то же самое, что деструктуризация из `props`.
const { name, doubleCount } = store // [!code warning]
name // всегда будет "Eduardo" // [!code warning]
doubleCount // всегда будет 0 // [!code warning]

setTimeout(() => {
    store.increment()
}, 1000)

// ✅ это будет реактивным.
// 💡 но вы также можете просто использовать `store.doubleCount` напрямую
const doubleValue = computed(() => store.doubleCount)
</script>
```

Для того чтобы извлечь свойства из хранилища, сохранив его реактивность, необходимо использовать `storeToRefs()`. Она создаст рефссылки для каждого реактивного свойства. Это полезно, когда вы используете только состояние из хранилища, но не вызываете никаких действий. Обратите внимание, что вы можете деструктурировать действия непосредственно из хранилища, поскольку они также привязаны к самому хранилищу:

```vue
<script setup>
import { storeToRefs } from 'pinia'

const store = useCounterStore()
// `name` и `doubleCount` являются реактивными refs.
// Это также извлечет ссылки на свойства, добавленные плагинами.
// но пропустит любое действие или не реактивное (не ref/reactive) свойство
const { name, doubleCount } = storeToRefs(store)
// действие инкремента может быть просто деструктурировано
const { increment } = store
</script>
```
