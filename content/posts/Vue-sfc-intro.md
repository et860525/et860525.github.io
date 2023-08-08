---
title: "Vuex - Single-File Components 介紹"
date: 2023-08-08
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend"]
---

這邊會把跟 Component 有關的使用方式整理並寫出來。Single-File Component 就是 Vue 最核心的功能，

<!--more-->

## Single-File Component

**Single-File Component, SFC** 是將某個元件以 `.vue` 檔案的方式包裝，再透過 `import` 的方式引入這個檔案來作為子元件。

SFC 通常包含三個部分：

- `<template>`：HTML 模板
- `<script>`：JavaScript 邏輯
-  `<style>`：CSS樣式

以下是 SFC 的案例：

- Vue2( Options )
	```html
  <template>
    <div class="app">
      <div>Hello Vue</div>
    </div>
  </template>
	
  <script>
    export default {
      data() {
        return {
          name: 'Mango'
        }
      }
    }
  </script>
	
	<style scoped>
	.app {
	  font-weight: bold;
	}
	</style>
	```
- Vue3( Composition )
	```html
	<template>
    <div class="app">
      <div>Hello Vue</div>
    </div>
	</template>
	
	<script setup>
	import { ref } from 'vue' 
	const greeting = ref('Mango!')
	</script>
	
	<style scoped>
	.app {
	  font-weight: bold;
	}
	</style>
	```

一個元件就是一個 `.vue` 檔案，這樣可以讓元件更容易管理，並提高可維護性。

> 注意: SFC 的 `.vue` 檔案並非網頁標準，需要先透過 `@vue/compiler-sfc` 來將 SFC 編譯成瀏覽器看得懂的 JavaScript 程式碼，才能被執行。

## 預處理器( Pre-Processors )

使用 `lang` 屬性來設定使用的語言，最常用的是 TypeScript：

```html
<script lang="ts">
  // 使用 TypeScript
</script>
```

`lang` 屬性也可以用在 `template` 與 `style` 區塊：

```html
<template lang="pug">
p {{ msg }}
</template>

<style lang="scss">
  $primary-color: #333;
  body {
    color: $primary-color;
  }
</style>
```

## 使用 src 引用

如果你的 `template`、`script`、`style` 太大的話，你可以先把它先做成一個檔案，再使用 `src` 來引入：

```html
<template src="./template.html"></template>
<style src="./style.css"></style>
<script src="./script.js"></script>
```

> 相對位置與絕對位置都能使用

## `<script setup>`

`<script setup>` 是一個編譯期的語法糖，在 SFC 裡使用  Composition API 才會使用的。

相比只使用 `<script>` 便利，其優點為：

- 簡潔的程式碼
- 可以使用純淨的 TypeScript
- 更好的運行效能
- 更好的型別推斷

## 結語

SFC 真的很方便，它可以把很多東西都先模組化，當需要的時後直接引入並使用，這樣很多東西就都不用再重寫了👍。

這邊把我的筆記整理出一個簡單理解 SFC 方式，往後的會用實際的例子來慢慢介紹 SFC 的更多方式。

## Reference

- [Single-File Components](https://vuejs.org/guide/scaling-up/sfc.html)
- [SFC Syntax Specification](https://vuejs.org/api/sfc-spec.html)