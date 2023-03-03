---
editLink: false
---

[Документация API](../index.md) / [pinia](../modules/pinia.md) / SubscriptionCallbackMutationDirect

# Интерфейс: SubscriptionCallbackMutationDirect

[pinia](../modules/pinia.md).SubscriptionCallbackMutationDirect

Контекст, передаваемый callback подписки при непосредственном изменении состояния хранилища с помощью `store.someState = newValue` или `store.$state.someState = newValue`.

## Иерархия %{#Hierarchy}%

-   [`_SubscriptionCallbackMutationBase`](pinia._SubscriptionCallbackMutationBase.md)

    ↳ **`SubscriptionCallbackMutationDirect`**

## Свойства %{#Properties}%

### события %{#Properties-events}%

• **events**: `DebuggerEvent`

🔴 Только для DEV, НЕ используйте для производственного кода. Различные вызовы мутаций. Происходит из <https://vuejs.org/guide/extras/reactivity-in-depth.html#reactivity-debugging> и позволяет отслеживать мутации в devtools и плагинах **только во время разработки**.

#### Переопределения %{#Properties-events-Overrides}%

[\_SubscriptionCallbackMutationBase](pinia._SubscriptionCallbackMutationBase.md).[events](pinia._SubscriptionCallbackMutationBase.md#events)

---

### storeId %{#Properties-storeId}%

• **storeId**: `string`

`id` хранилища, выполняющего мутацию.

#### Унаследовано от %{#Properties-storeId-Inherited-from}%

[\_SubscriptionCallbackMutationBase](pinia._SubscriptionCallbackMutationBase.md).[storeId](pinia._SubscriptionCallbackMutationBase.md#storeid)

---

### тип %{#Properties-type}%

• **type**: [`direct`](../enums/pinia.MutationType.md#direct)

Тип мутации.

#### Переопределения %{#Properties-type-Overrides}%

[\_SubscriptionCallbackMutationBase](pinia._SubscriptionCallbackMutationBase.md).[type](pinia._SubscriptionCallbackMutationBase.md#type)
