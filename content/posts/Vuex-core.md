---
title: "Vuex - 核心概念"
date: 2023-08-04
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend", "TypeScript"]
---

以下是 Vuex 常用的操作，以及使用 TypeScript 所會遇到的狀況處理。

<!--more-->

## Getter

將 Store 中的 State 取出來並預先做一些處理。

先新增一個 `User` 進到 Store 裡，使用 `getters` 來加入一些字串：

- `store.ts`
	```ts
	import { InjectionKey } from 'vue'
	import { createStore, Store } from 'vuex'
	
	interface User {
      name: string,
      age: number,
      gender: string
	}
	
	export interface State {
      count: number,
      user: User,
	}
	
	export const key: InjectionKey<Store<State>> = Symbol()
	
	export const store = createStore<State>({
	  state: {
	    count: 0,
	    user: {
	      name: 'Paul',
	      age: 80,
	      gender: 'male'
	    }
	  },
	  getters: {
	    getUserInfo: state => {
	      return state.user.name + ' is ' + state.user.age + ' years old.'
	    }
	  },
	  //...	
	})
	```
- `Home.vue`
	```html
	<template>
	  <p class="text-md">{{ userInfo }}</p>
	</template>
	
	<script setup lang="ts">
	import { computed } from 'vue';
	import { useStore } from 'vuex';
	import { key }  from '../store';
	
	const store = useStore(key);
	const userInfo = computed(() => store.getters.getUserInfo );
	
	//...
	</script>
	```

## Mutations

**Mutations** 是 Vuex 唯一能修改 State 的方式。

```js
const store = createStore({
  state: {
    count: 1
  },
  mutations: {
    increment (state) {
      // mutate state
      state.count++
    }
  }
})
```

使用方式為：

```js
store.commit('increment')
```

### 帶有參數的 method

如果所設定的 `increment` 帶有參數：

```js
const store = createStore({
  state: {
    count: 1
  },
  mutations: {
    increment (state, n) {
      // mutate state
      state.count += n
    }
  }
})
```

使用方式為：

```js
store.commit('increment', 5)
```

其他更進階的使用方式可以參考 [Mutations](https://vuex.vuejs.org/guide/mutations.html)。

## Actions

與 Mutations 相似，但有所不同的是：

- Actions 不會改變 State，而是呼叫 Mutations 來改變
- Actions 可以異步( asynchronous ) 操作

```js
const store = createStore({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  },
  actions: {
    increment (context) {
      context.commit('increment')
    }
  }
})
```

可以簡寫成：

```js
//...
actions: {
  increment ({ commit }) {
    commit('increment')
  }
}
```

使用方式為：

```js
store.dispatch('increment')
```

如果想要感受一下異步操作，可以將 Actions 設定一個時間來執行：

```js
actions: {
  incrementAsync ({ commit }) {
    setTimeout(() => {
      commit('increment')
    }, 1000)
  }
}
```

## Module

當應用程式變大時，如果將所有狀態都集中在 store 實例中，就會變得複雜且難以維護。

所以 Vuex 提供了將 store 分割成**模塊(module)** 的方式，讓每個 module 都擁有自己的 state、mutation、action、getter 甚至是子模塊。

以下會實作 JavaScript 與 TypeScript 的作法。

### 使用 JavaScript

首先，建立一個資料夾名為 `store/` 來存放有關 Vuex 的文件。所以這邊可以將上面在使用的 `store.js` 放入 `store/` 中。

接著建立一個 `user.js` 檔案：

```js
export const userModule = {
  state: () => ({
  name: 'Paul',
    age: 80,
    gender: 'male',
  }),
  getters: {
    getUserInfo => {
      return state.name + ' is ' + state.age + ' years old.'
    }
  },
}
```

接下來將 `count` 狀態提取出來，變成一個獨立的 module ( 這裡為了方便，就不另外再建立新的檔案來存放 `count` 狀態 )：

```js
import { InjectionKey } from 'vue'
import { createStore, Store } from 'vuex'
import { countModule } from './buttonCount';
import { userModule } from './user'

const countModule = {
  state: () => ({
    count: 0
  }),
  mutations: {
    increment(state) {
      state.count++;
    }
  },
}


export const store = createStore({
  modules: {
    count: countModule,
    user: userModule  
  }
})
```

### 使用 TypeScript

使用 TypeScript 就會稍稍的有一些麻煩，這是因為，如果直接使用 `modules` 來引入我們預先設定好的 module，TypeScript 會抓不到這些 module 的 state。所以這裡我參考了[將modules 的類型添加到 state 中](https://juejin.cn/post/7005450507531583525) 來讓 TypeScript 來知道這些 state 的資訊。

> 當然，如果不理會這個問題並直接使用 modules 來呼叫，程式還是可以運作

- `user.ts`
	```ts
	import { Module } from 'vuex';
	
	export type User = {
	  name: string;
	  age: number;
	  gender: string;
	};
	
	export const userModule: Module<User, any> = {
	  state: (): User => ({
	    name: 'Paul',
	    age: 80,
	    gender: 'male',
	  }),
	  getters: {
	    getUserInfo: (state) => {
	      return state.name + ' is ' + state.age + ' years old.';
	    },
	  },
	};
	```
- `buttonCount.ts`：以防搞混，我先將 button 的 state 提取出來
	```ts
	import { Module } from 'vuex';
	
	type ICount = {
	  count: number;
	};
	
	export const countModule: Module<ICount, any> = {
	  state: (): ICount => ({
	    count: 0,
	  }),
	  mutations: {
	    increment(state: ICount) {
	      state.count++;
	    },
	  },
	};
	```
- `store.ts`
	```ts
	import { InjectionKey } from 'vue';
	import { createStore, Store, useStore as baseUseStore } from 'vuex';
	import { countModule } from './buttonCount';
	import { userModule } from './user';
	
	const modules = {
	  countModule,
	  userModule,
	};
	
	// 獲取所有 module 的 state
	type modulesState = {
	  [key in keyof typeof modules]: Exclude<Exclude<(typeof modules)[key]['state'], undefined>, () => any>;
	};
	
	export interface State extends modulesState {}
	
	export const key: InjectionKey<Store<modulesState>> = Symbol();
	
	export const store = createStore<modulesState>({
	  modules,
	});
	
	export function useStore() {
	  return baseUseStore(key);
	}
	```

  - `Home.vue`
  ```html
  <template>
    <p class="text-xl">{{ incrementCount }}</p>
  </template>

  <script>
  import { useStore } from '../store/store';

  const incrementCount = computed(() => store.state.countModule.count);
  </script>
  ```

	這樣編譯器就不會在使用其他 module 的 state 時顯示錯誤了。

### Namespace

如果今天有兩個 Module 且都擁有同個名為 `increment` 的方法，這時候就需要 namespace 來幫忙。

`store.js`：

```js
const moduleA = {
  // ...
  namespaced: true,
  mutations: {
    increment(state) {
      state.count++
    },
  },
}

const moduleB = {
  // ...
  namespaced: true,
  mutations: {
    increment(state) {
      state.count++
    },
  },
}

const store = createStore({ 
  modules: {
    a: moduleA,
    b: moduleB,
  }
})
```

外部使用：

```js
commit('a/increment')
commit('b/increment')
```

只要有設定 **namespace** 的 module，不管是使用 `getters`、`mutations`、`actions`，都必須要在前面加上該模組的名稱。這種方式也常運用在 `nested module` ：

```js
const store = createStore({
  modules: {
    account: {
      namespaced: true,

      // module assets
      state: () => ({ ... }), // module state is already nested and not affected by namespace option
      getters: {
        isAdmin () { ... } // -> getters['account/isAdmin']
      },
      actions: {
        login () { ... } // -> dispatch('account/login')
      },
      mutations: {
        login () { ... } // -> commit('account/login')
      },

      // nested modules
      modules: {
        // inherits the namespace from parent module
        myPage: {
          state: () => ({ ... }),
          getters: {
            profile () { ... } // -> getters['account/profile']
          }
        },

        // further nest the namespace
        posts: {
          namespaced: true,

          state: () => ({ ... }),
          getters: {
            popular () { ... } // -> getters['account/posts/popular']
          }
        }
      }
    }
  }
})
```

### 註冊動態 Module

可以在 `createStore` 之後使用 `store.registerModule` 建立 module：

```js
import { createStore } from 'vuex'

const store = createStore({ /* options */ })

// register a module `myModule`
store.registerModule('myModule', {
  // ...
})

// register a nested module `nested/myModule`
store.registerModule(['nested', 'myModule'], {
  // ...
})
```

## 結語

以上就是 Vuex 基本的使用方式。如果使用 TypeScript 比較多的問題是出現在 Modules，要能讓編譯器了解這些 Module 裡的 State，就要先費一番力了。