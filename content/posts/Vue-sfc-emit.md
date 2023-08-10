---
title: "Single-File Components - Events"
date: 2023-08-10
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend"]
---

上一篇有提到，子組件設定 `props` 能讓父組件依照所設定的參數，傳進對應的值。那如果子組件想要回傳已處理好的狀態給父組件，那該怎麼做呢？

這就必須用到 `event` 來更新父組件狀態。

> 在 Vue 裡有一個說法：**Props in, Event out**

<!--more-->

## 監聽 Event

這裡直接使用官方的例子：每按一次 `Enlarge text` 按鈕就會讓該元件的標題變大。

首先，先在 `App.vue`( 父組件 ) 新增改變字體大小的狀態( `postFontSize` )，並把它加入到組件的上方來套用該 Style：

```html
<script setup>
const products = ref([
  /* ... */
])

const postFontSize = ref(1)
</script>

<template>
  <div :style="{ fontSize: postFontSize + 'em' }">
    <Product 
	  v-for="product in products" 
	  :key="product.id" 
	  :title="product.title" 
    />
  </div>
</template>
```

接著新增一個按鈕到 `Product.vue` 裡：

```html
<template>
  <h4>{{ title }}</h4>
  <button>Enlarge text</button>
</template>
```

目前這個 `button` 不會做任何事情，必須要讓父組件呼叫這個 `button` 做放大標題的功能。

所以可以在 `App.vue` 裡會使用 `@` 來觸發一個自定義事件，就像觸發原生 DOM 事件一樣：

```html
<Product
  ... 
  @enlarge-text="postFontSize += 0.1"
/>
```

子組件可以透過使用 `$emit` 來調用自身的發出事件：

```html
<template>
  <h4>{{ title }}</h4>
  <button @click="$emit('enlarge-text')">Enlarge text</button>
</template>
```

按下 `Enlarge text` 按鈕就可以看到結果了。

## defineEmits

與 `defineProps` 一樣，都是預先定義在 `<script setup>` 而且不用特別宣告的巨集。

`defineEmits` 可以讓這些事件更好管理：

```html
<script setup>
defineProps(['title'])
defineEmits(['enlarge-text'])
</script>
```

也可以使用一個變數來儲存它( 這裡就不需要使用 `$emit`，因為這裡是使用變數 `emit` )：

```html
<script setup>
//...
const emit = defineEmits(['enlarge-text'])
</script>


<template>
  <!--...-->
  <button @click="emit('enlarge-text')">Enlarge text</button>
</template>
```

## 使用 defineProps 與 defineEmits 的時機

根據 [Vue - Props](https://vuejs.org/guide/components/props.html#props-declaration)，只要是 SFC 檔案，就使用 `defineProps` 與 `defineEmits`。

相比 `props` 與 `emit`，`defineProps` 與 `defineEmits` 都提供型別推斷的功能。

```html
<script setup>
const props = defineProps(['foo'])

console.log(props.foo)
</script>
```

沒有 `<script setup>` 的則是使用 `props`：

```js
export default {
  props: ['foo'],
  setup(props) {
    // setup() receives props as the first argument.
    console.log(props.foo)
  }
}
```

## 結語

對於 Component 的溝通方式，最大的重點就是 **Props in, Event out**。