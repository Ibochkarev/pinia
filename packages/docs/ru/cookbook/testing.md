# Тестирование хранилища

Хранилища, по замыслу, будут использоваться во многих местах и могут значительно усложнить тестирование, чем оно должно быть. К счастью, это не обязательно должно быть так. При тестировании хранилищ нам нужно позаботиться о трех вещах:

- Экземпляр `pinia`: хранилища не могут работать без него
- `actions`: в большинстве случаев они содержат самую сложную логику наших хранилищ. Разве не было бы неплохо, если бы над ними издевались по умолчанию?
- Плагины: Если вы полагаетесь на плагины, вам также придется устанавливать их для тестов

В зависимости от того, что или как вы тестируете, нам нужно позаботиться об этих трех по-разному:

- [Тестирование хранилища](#testing-stores)
  - [Модульное тестирование хранилища](#unit-testing-a-store)
  - [Модульное тестирование компонентов](#unit-testing-components)
    - [Инициализация состояния](#initial-state)
    - [Настройка поведения экшенов](#customizing-behavior-of-actions)
    - [Указание функции createSpy](#specifying-the-createspy-function)
    - [Mocking getters](#mocking-getters)
    - [Плагины pinia](#pinia-plugins)
  - [E2E tests](#e2e-tests)
  - [Компоненты модульного тестирования (Vue 2)](#unit-test-components-vue-2)

## Модульное тестирование хранилища

Для модульного тестирования хранилища наиболее важной частью является создание экземпляра `pinia`:

```js
// stores/counter.spec.ts
import { setActivePinia, createPinia } from 'pinia'
import { useCounter } from '../src/stores/counter'

describe('Counter Store', () => {
  beforeEach(() => {
    // создаем свежую pinia и делаем ее активной, чтобы она автоматически подхватывалась
    // подхватывается любым вызовом useStore() без необходимости передавать его ему:
    // `useStore(pinia)`.
    setActivePinia(createPinia())
  })

  it('increments', () => {
    const counter = useCounter()
    expect(counter.n).toBe(0)
    counter.increment()
    expect(counter.n).toBe(1)
  })

  it('increments by amount', () => {
    const counter = useCounter()
    counter.increment(10)
    expect(counter.n).toBe(10)
  })
})
```

Если у вас есть плагины для хранилтища, необходимо знать одну важную вещь: **плагины не будут использоваться, пока `pinia` не будет установлена в App**. Это можно решить, создав пустое или поддельное приложение:

```js
import { setActivePinia, createPinia } from 'pinia'
import { createApp } from 'vue'
import { somePlugin } from '../src/stores/plugin'

// тот же код, что и выше...

// вам не нужно создавать одно приложение для каждого теста
const app = createApp({})
beforeEach(() => {
  const pinia = createPinia().use(somePlugin)
  app.use(pinia)
  setActivePinia(pinia)
})
```

## Компоненты для модульного тестирования

Этого можно достичь с помощью `createTestingPinia()`, которая возвращает экземпляр pinia, созданный для помощи в модульном тестировании компонентов.

Начните с установки `@pinia/testing`:

```shell
npm i -D @pinia/testing
```

И не забудьте создать тестирующую pinia в ваших тестах при установке компонента:

```js
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
// import any store you want to interact with in tests
import { useSomeStore } from '@/stores/myStore'

const wrapper = mount(Counter, {
  global: {
    plugins: [createTestingPinia()],
  },
})

const store = useSomeStore() // использовать тестирующую pinia!

// состоянием можно управлять напрямую
store.name = 'my new name'
// также может быть сделано с помощью патча
store.$patch({ name: 'new name' })
expect(store.name).toBe('new name')

// экшены по умолчанию отключены, что означает, что они не выполняют свой код по умолчанию.
// Смотрите ниже, чтобы настроить это поведение.
store.someAction()

expect(store.someAction).toHaveBeenCalledTimes(1)
expect(store.someAction).toHaveBeenLastCalledWith()
```

Пожалуйста, обратите внимание, что если вы используете Vue 2, для `@vue/test-utils` требуется [немного иная конфигурация](#unit-test-components-vue-2).

### Начальное состояние

Вы можете установить начальное состояние **всех ваших хранилищ** при создании тестирующей pinia, передав ей объект `initialState`. Этот объект будет использоваться тестирующей pinia для _исправления_ хранилищ при их создании. Допустим, вы хотите инициализировать состояние этого хранилища:

```ts
import { defineStore } from 'pinia'

const useCounterStore = defineStore('counter', {
  state: () => ({ n: 0 }),
  // ...
})
```

Поскольку хранилище имеет имя _"counter"_, необходимо добавить соответствующий объект в `initialState`:

```ts
// где-то в вашем тесте
const wrapper = mount(Counter, {
  global: {
    plugins: [
      createTestingPinia({
        initialState: {
          counter: { n: 20 }, // запустить счетчик с 20 вместо 0
        },
      }),
    ],
  },
})

const store = useSomeStore() // использует тестирующую pinia!
store.n // 20
```

### Настройка поведения экшенов

`createTestingPinia` вставляет все действия магазина, если не сказано иначе. Это позволяет вам тестировать ваши компоненты и магазины отдельно.

Если вы хотите изменить это поведение и нормально выполнять свои действия во время тестов, укажите `stubActions: false` при вызове `createTestingPinia`:

```js
const wrapper = mount(Counter, {
  global: {
    plugins: [createTestingPinia({ stubActions: false })],
  },
})

const store = useSomeStore()

// Теперь этот вызов БУДЕТ выполнять реализацию, определенную хранилищем
store.someAction()

// ...но он все еще обернут шпионом, поэтому вы можете проверять вызовы
expect(store.someAction).toHaveBeenCalledTimes(1)
```

### Указание функции createSpy

При использовании Jest или vitest с `globals: true`, `createTestingPinia` автоматически создает заглушки экшенов', используя функцию spy, основанную на существующем фреймворке тестирования (`jest.fn` или `vitest.fn`). Если вы используете другой фреймворк, вам необходимо указать опцию [createSpy](/api/interfaces/pinia_testing.TestingOptions.html#createspy):

```js
import sinon from 'sinon'

createTestingPinia({
  createSpy: sinon.spy, // используйте sinon's spy для завершения действий
})
```

Вы можете найти больше примеров в [тестах пакета тестирования](https://github.com/vuejs/pinia/blob/v2/packages/testing/src/testing.spec.ts ).

### Mocking getters

По умолчанию любой параметр получения будет вычислен как при обычном использовании, но вы можете вручную принудительно ввести значение, установив для параметра получения любое значение, которое вы хотите:

```ts
import { defineStore } from 'pinia'
import { createTestingPinia } from '@pinia/testing'

const useCounter = defineStore('counter', {
  state: () => ({ n: 1 }),
  getters: {
    double: (state) => state.n * 2,
  },
})

const pinia = createTestingPinia()
const counter = useCounter(pinia)

counter.double = 3 // 🪄 геттеры доступны для записи только в тестах

// установить в undefined, чтобы сбросить поведение по умолчанию
// @ts-expect-error: обычно это число
counter.double = undefined
counter.double // 2 (=1 x 2)
```

### Плагины Pinia

Если у вас есть какие-либо плагины pinia, обязательно передайте их при вызове `createTestingPinia()`, чтобы они были правильно применены. ***Не добавляйте их с помощью `testingPinia.use(мой плагин)`***, как вы бы сделали с обычной pinia:

```js
import { createTestingPinia } from '@pinia/testing'
import { somePlugin } from '../src/stores/plugin'

// внутри какого-то теста
const wrapper = mount(Counter, {
  global: {
    plugins: [
      createTestingPinia({
        stubActions: false,
        plugins: [somePlugin],
      }),
    ],
  },
})
```

## Тесты E2E

Когда дело доходит до pinkie, вам не нужно ничего менять для тестов e2e, в этом весь смысл тестов e2e! Возможно, вы могли бы протестировать HTTP-запросы, но это выходит далеко за рамки данного руководства 😄.

## Компоненты модульного тестирования (Vue 2)

При использовании [Vue Test Utils 1](https://v1.testutils.vuejs.org/), установите Pinia на `local Vue`:

```js
import { PiniaVuePlugin } from 'pinia'
import { createLocalVue, mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'

const localVue = createLocalVue()
localVue.use(PiniaVuePlugin)

const wrapper = mount(Counter, {
  localVue,
  pinia: createTestingPinia(),
})

const store = useSomeStore() // uses the testing pinia!
```
