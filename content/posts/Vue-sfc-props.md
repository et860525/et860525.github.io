---
title: "Single-File Components - Props"
date: 2023-08-09
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend"]
---

Component 會使用 `props` 來宣告它所要獲得的參數，以便讓其他的 Components 能傳對應的值給它。

<!--more-->

## 傳送參數

這裡會示範讓每一個 `Product.vue` 元件都有各自的 `title`。

- `Product.vue`
	```html
	<script setup>
	defineProps(['title'])
	</script>
	
	<template>
	  <h4>{{ title }}</h4>
	</template>
	```
	`defineProps` 是一個只存在 `<script setup>` 的巨集，它不需要像 `ref` 之類的要先導入，它的功用是**接收使用這個元件建立時所設定的參數**。
- `App.vue`
	```html
	<!-- Normal -->
	<Product title="Iphone 13" />
	<Product title="Macbook" />
	<Product title="Airpod" />
	
	<!-- Loop -->
	<script>
	const products = ref([
	  { id: 1, title: 'Iphone 13' },
	  { id: 2, title: 'Macbook' },
	  { id: 3, title: 'Airpod' }
	])
	</script>
	
	<Product 
	  v-for="product in products" 
	  :key="product.id" 
	  :title="product.title"
	/>
	```
	這裡可以看到使用 `<Product>` 組件時需要提供 `title` 參數值

如果設定的 prop 有大寫，那會以 **Kebab Case** 來呈現：

```html
<script setup>
  defineProps(['greetingWord'])
</script>
```

```html
<CustomProp greeting-word="hello"/>
```

## 單向綁定

通常都會把傳進來的值放在一個變數裡方便使用：

```js
const props = defineProps({
  data: {
    type: string,
    default: '',
  },
});

const msgCount = computed(() => {
  return props.data.length;
});
```

> 注意：以上的 `props` 是唯讀狀態

如果想要使用傳進來的數值，有兩種方式：

1. props 只接收最初的數值，這是用於給子組件的本地屬性。最好的方法就是使用一個變數儲存 props 的值：
	```js
	const props = defineProps(['initialCounter'])
	
	// counter 只有在這個組件初始化時獲得值
	const counter = ref(props.initialCounter)
	```
2. 使用 computed 來獲得值：
	```js
	const props = defineProps(['size'])
	
	// computed 就是在 size 改變時自動變動
	const normalizedSize = computed(() => props.size.trim().toLowerCase())
	```

## 資料驗證

`defineProps` 可以對傳進來的值做一些動作，例如：型別、初始值...等等。

```js
defineProps({
  propN: Number,
  propS: { type: String, default: 'Hello' },
})
```

還有更多的可以查詢 [Prop Validation](https://vuejs.org/guide/components/props.html#prop-validation)。

## 結語

目前這種傳值的方式是由「父組件 > 子組件」的方式。下一篇會說明，如何讓子組件與父組件進行溝通。