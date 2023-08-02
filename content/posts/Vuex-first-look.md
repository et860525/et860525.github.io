---
title: "Vuex - 初見筆記"
date: 2023-08-02
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend", "TypeScript"]
---

好久不見，因為最近在一堆接案與做一些 PT 來過最後一個暑假。今天我要來把我過去的一些筆記拿出來整理一下，並記錄一下最近有在用到很重要的 Vuex。

<!--more-->

## Vuex

[Vuex](https://vuex.vuejs.org/) 是 Vue.js 的**狀態管理**庫，他採用集中式管理所有元件( component ) 的狀態。

在使用 Vue.js 時，元件之間可能會需要共用資料，例如：Auth token、使用者的資訊、獲得的一些特定資料...等等。這時候就需要一個空間來管理這些資料的**狀態**，例如：

- 購物車狀態：使用者新增、修改購物車裡的物品或數量
- 通知狀態：IG 上未讀的訊息狀態

## 前端狀態管理

**瀏覽器端**管理狀態：

- Cookie
- Local Storage

**前端框架**管理狀態：

- Vuex( Vue )
- Redux( React )
- ngrx/store( Angualr )

## 狀態管理模式

以下是一個簡單的**單向數據流** 概念：

- **state**：驅動應用的原始數據
- **view**：以聲明的方式映射到 view 上
- **actions**：讓用戶在 view 上輸入並讓狀態變化

![vuex-flow.png](/images/Vuex/vuex-flow.png)

但是，如果該應用本身有**多個組件共享同一個狀態**，那麼單向數據流很容易被破壞：

- **多個 view 依賴同一個狀態**：在多層的 components 傳遞 props 很麻煩，而且並不適用於兄弟 components
- **這些 view 都需要變更同一個狀態**：父子 components 通常都使用直接引用或利用事件來同步狀態

上面這兩個問題都讓程式碼很難以維護。

所以，可以把這些組件所共享的狀態拉出來，並使用一個全局實例來管理這些狀態。這樣不管組件在哪裡都能獲取到狀態。

![vuex.png](/images/Vuex/vuex.png)

流程為下：

1. 使用者在 Vue Component 裡點擊按鈕來觸發事件( Event Handler )
2. Event Handler 中的 Dispatch 讓 Actions 來呼叫相應的 action handler
3. Commit 讓 Mutations 呼叫相應的 mulate
4. 最後改變 Store 裡的 State，並渲染到元件中

>  - Store：一個 App 只能有一個 Store
>  - State：被 Store 所管理的狀態

## Installation

```bash
npm install vuex@next --save
```

## Usage

首先，建立一個 store 實例：

```js
// main.js
import { createApp } from 'vue'
import { createStore } from 'vuex'
import App from './App.vue';

const app = createApp(App);

// 建立一個 store 實例
const store = createStore({
  state () {
    return {
      count: 0
    }
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})

const app = createApp(app)

app.use(store)

app.mount('#app');
```

通常在實務上都會讓 `store` 實例與 `main.js` 分離：

- `store.js`
	```js
	import { createStore } from 'vuex'
	
	export const store = createStore({
	  state () {
	    return {
	      count: 0
	    }
	  },
	  mutations: {
	    increment (state) {
	      state.count++
	    }
	  }
	})
	```
- `main.js`
	```js
	import { createApp } from 'vue';
	import App from './App.vue';
	import { store } from './store';
	
	const app = createApp(App);
	
	app.use(store);
	
	app.mount('#app');
	```

建立完實例後，就可以開始使用 `store.state` 來獲取狀態了：

```js
store.commit('increment')
console.log(store.state.count) // 1
```

幾個需要知道的：

- **Dispatch**：會觸發 Action
- **Commit**：會觸發 Mutation( 所以上面才使用 commit 來觸發設定在 mutations 裡的 increment )
- **Action**：定義整個 App 中的所有行為，Action 是透過 Mutation 來改變 State 資料，在 Action 裡通常都是非同步的方式來處理
- **Mutation**：是真正唯一可以改變 State 的資料，屬於同步方式

## 支援 TypeScript

如果是 Vue + TypeScript 來使用 vuex，可以根據官方資訊 [TypeScript 支持](https://vuex.vuejs.org/zh/guide/typescript-support.html#usestore-%E7%BB%84%E5%90%88%E5%BC%8F%E5%87%BD%E6%95%B0%E7%B1%BB%E5%9E%8B%E5%A3%B0%E6%98%8E) 來進行相關設定。

- `store.ts`
	```ts
	import { InjectionKey } from 'vue'
	import { createStore, Store } from 'vuex'
	
	export interface State {
	  count: number
	}
	
	export const key: InjectionKey<Store<State>> = Symbol()
	
	export const store = createStore<State>({
	  state: {
	    count: 0
      },
      mutations: {
        increment(state: State) {
          state.count++;
        }
      }	
    })
	```
- `main.ts`
	```ts
	import { createApp } from 'vue';
	import App from './App.vue';
	import { store, key } from './store';
	
	const app = createApp(App);
	
	app.use(store, key);
	
	app.mount('#app');
	```
- `Home.vue`
	```html
	<template>
	  <button @click="store.commit('increment')">
	    Button
	  </button>
	  <p class="text-xl">{{store.state.count}}</p>
	</template>
	
	<script setup lang="ts">
	import { ref } from 'vue';
	import { useStore } from 'vuex';
	import { key }  from '../store';
	
	const store = useStore(key);
	
	//...
	</script>
	```

## 結語

如果是自己在使用 Vuex，都會盡量使用 TypeScript 來設定，所以這邊也順便紀錄一下。