---
title: "Spring Boot 專案(五) - 單元測試 Mockito"
date: 2023-05-29
draft: false
author: "Chen Yu Fan"
tags: ["Java", "Spring", "Projects", "API"]
---

**單元測試(Unit Test)**，是針對程式撰寫時，對最小單位進行正確性驗證的測試。一個**單元(Unit)** 可以是一支程式、方法或過程，在物件導向的程式裡，最小的單元就是**方法**。

<!--more-->

## 單元、整合、端對端測試

簡單的歸納為：  
- **單元(Unit Test)**：主要由程式開發人員自行撰寫測試。  
- **整合(Integration Test)**：若公司有專職的測試人員（例如 QA 團隊）通常會由其擔任整合測試角色，沒有的話，一般來說會是請該模組開發人員負責。  
- **端對端測試(End to End Test)**：使用者角度出發的測試，可以透過人工對已經完整部屬的網站進行測試，為人工測式的主要範圍。

來源：[Java Unit Test — Mockito](https://vivifish.medium.com/java-%E5%96%AE%E5%85%83%E6%B8%AC%E8%A9%A6%E5%B7%A5%E5%85%B7-mockito-e5f0ce93579d)

## Mockito

Mockito 是 Java mock 框架，主要用來做 mock 測試的。可模擬任何 Spring 管理的 bean、方法的返回值、拋出異常等。

> mock：就是一個假物件，避免為了測試一個方法，就要建立整個 bean。用 mock 來驗證本身的邏輯，與第三方的互動是否正確

會用以下的方式來產生我們預期的結果：

```text
Mockito.when(對象.方法名()).thenReturn(自定義結果)
```

採取 3A 測試原則：

1. **Arrange (準備)**：初始化目標物件
2. **Act (執行)**：呼叫目標物件的方法
3. **Assert (驗證)**：驗證是否符合預期的結果

建立 Spring Boot 專案時就已經引入了

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 測試 Service

在 `test` 資料夾新增一個 `TestTodoService.java`，並加入以下程式碼：

```java
import org.springframework.boot.test.context.SpringBootTest;
import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
public class TestTodoService {
  @Autowired
  TodoService todoService;
	
  @MockBean
  TodoDao todoDao;
}
```

- `@MockBean`：測試所需要的 Spring Boot 容器

測試 `getTodos()`：

```java
@Test
public void testGetTodos () {
  // [Arrange]
  List<Todo> expectedTodosList = new ArrayList<Todo>();
  Todo todo = new Todo();
  todo.setId(1);
  todo.setTask("玩電腦");
  todo.setStatus(1);
  expectedTodosList.add(todo);
	
  // 模擬回傳結果 
  Mockito.when(todoDao.findAll()).thenReturn(expectedTodosList);
	
  // [Act]
  Iterable<Todo> actualTodoList = todoService.getTodos();
	
  // [Assert]
  assertEquals(expectedTodosList, actualTodoList);
}
```

當 test 寫完後，VSCode 旁邊會出現一個圖示讓你能執行 test 的按鈕

![Spring-mockito-test.png](/images/Spring-boot/Spring-mockito-test.png)

按下執行後就可以看到測試的狀態。再來寫一個測試 `updateTodo()` 的方法，`updateTodo()` 會回傳三種情況：

1. 成功
2. 找不到 Id
3. 例外

根據三種情況各別寫出測試單元：

```java
@Test
public void testUpdateTodoSuccess() {
  // [Arrange]
  Todo todo = new Todo();
  todo.setId(1);
  todo.setTask("玩電腦");
  todo.setStatus(1);
  Optional<Todo> resTodo = Optional.of(todo);
  
  // 模擬回傳結果
  Mockito.when(todoDao.findById(1)).thenReturn(resTodo);
  
  // [Act]
  Boolean actualTodo = todoService.updateTodo(1, todo);
  
  // [Assert]
  assertEquals(true, actualTodo);
}

@Test
public void testUpdateTodoNotExist() {
  // [Arrange]
  Todo todo = new Todo();
  todo.setStatus(2);
  Optional<Todo> resTodo = Optional.of(todo);

  // 模擬回傳結果 (資料庫沒有 id=150 的資料，故回傳 empty 物件)
  Mockito.when(todoDao.findById(150)).thenReturn(Optional.empty());

  // [Act]
  Boolean actualTodo = todoService.updateTodo(150, todo);

  // [Assert]
  assertEquals(false, actualTodo);
}

@Test
public void testUpdateTodoOccurException () {
  // [Arrange]
  Todo todo = new Todo();
  todo.setId(1);
  todo.setStatus(1);
  Optional<Todo> resTodo = Optional.of(todo);

  // 模擬回傳結果 (資料庫裡的 id=1 資料)
  Mockito.when(todoDao.findById(1)).thenReturn(resTodo);
  todo.setStatus(2);

  // 模擬發生 NullPointerException
  doThrow(NullPointerException.class).when(todoDao).save(todo);
  
  // [Act]
  Boolean actualTodo = todoService.updateTodo(2, todo);

  // [Assert]
  assertEquals(false, actualTodo);
}
```

### 測試 Controller

Controller 是讓使用者透過 HTTP 呼叫的，這裡就可以使用 [MockMvc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/MockMvc.html) 來實作。

`MockMvc` 元件能針對當前專案模擬出 HTTP 的請求，並取得 status code、response header、response body 等結果。以下是 MockMvc 的屬性：

- `mockMvc.perform`：執行一個請求，並對應到 controller
- `mockMvc.andExpect`：期待並驗證回應是否正確
- `mockMvc.andReturn`：最後回應的值(body)，可以再利用這個值，做其他 Assert 驗證 ( 這邊會用 `value()` 直接驗證 )

在 `test` 資料夾新增一個 `TestTodoController.java`，並加入以下程式碼：

```java
@SpringBootTest
@AutoConfigureMockMvc
public class TestTodoController {
  @Autowired
  private MockMvc mockMvc;
  
  @MockBean
  TodoService todoService;
}
```

- `@AutoConfigureMockMvc`：啟動時注入 MockMvc

先測試 `[GET] /api/todos`：

```java
@Test
public void testGetTodos() throws Exception {
  // [Arrange]
  List<Todo> expectedList = new ArrayList<Todo>();
  Todo mockTodo = new Todo();
  mockTodo.setId(1);
  mockTodo.setTask("看書");
  mockTodo.setStatus(1);
  
  expectedList.add(mockTodo);
  
  // 模擬回傳結果
  Mockito.when(todoService.getTodos()).thenReturn(expectedList);
  
  // 存放請求標頭的資料
  HttpHeaders httpHeaders = new HttpHeaders();
  httpHeaders.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
  
  // [Act] + [Assert] 發出請求並驗證
  mockMvc.perform(get("/api/todos")
  .headers(httpHeaders))
    .andDo(print())
    .andExpect(status().isOk())
    .andExpect(jsonPath("$[0].id").hasJsonPath())
    .andExpect(jsonPath("$[0].task").value(mockTodo.getTask()))
    .andExpect(header().string(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE));
}
```

- `andDo(print())` 方法，他能將該測試的請求與返回印在 Console 上
- `status().isOk()`：驗證 HTTP Status Code 為 200
- `jsonPath()`：獲取指定的 JSON 欄位的值。`$` 是開始，`.` 前往下一層
- `hasJsonPath()`：驗證該 JSON 欄位是否存在
- `value()`：驗證某個 JSON 欄位的值
- `header().string()`：驗證 Response 標頭的值

再來測試 `deleteTodo()`，它會有兩種回傳：

1. 成功
2. 找不到 Id

```java
@Test
public void testDeleteTodoSuccess() throws Exception {
  // 模擬回傳結果
  Mockito.when(todoService.deleteTodo(1)).thenReturn(true);
  
  // 存放請求標頭的資料
  RequestBuilder requestBuilder =
      MockMvcRequestBuilders
        .delete("/api/todos/1")
        .accept(MediaType.APPLICATION_JSON) // response 的型別
        .contentType(MediaType.APPLICATION_JSON); // request 的型別

  // 模擬呼叫
  mockMvc.perform(requestBuilder)
    .andDo(print())
    .andExpect(status().isNoContent());
}

@Test
public void testDeleteTodoIdNotExist() throws Exception {
  // 模擬回傳結果
  Mockito.when(todoService.deleteTodo(100)).thenReturn(false);

  // 存放請求標頭的資料
  HttpHeaders httpHeaders = new HttpHeaders();
  httpHeaders.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE); // response 的型別
  httpHeaders.add(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE); // request 的型別

  // 模擬呼叫
  mockMvc.perform(MockMvcRequestBuilders.delete("/api/todos/100").headers(httpHeaders))
    .andExpect(status().isBadRequest()); 
}
```

我在這兩個測試裡面嘗試了兩種 header 的設定，結果其實都是一樣的 ( 參考 [Class MockHttpServletRequestBuilder](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/request/MockHttpServletRequestBuilder.html) )。

如果要讓某些測試擁有一樣的 Header 設定，可以建立一個 `TestBase` 物件，並將相關設定放入，並由其他測試物件繼承即可。

```java
public class TestBase {
  protected HttpHeaders httpHeaders;
  
  @Before
  public void init() {
    httpHeaders = new HttpHeaders();
    httpHeaders.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
  }
}
```

- `@Before`：該物件需要是 `public`，會在測試開始前自動執行 (有 `@Before` 當然就有 `@After`)

---

## 結語

如果都使用手動測試，那一定會很麻煩又很費時，所以將測試自動化是很重要的。如果未來有新需求，在撰寫新的程式碼時，可以利用測試來檢查新的程式碼是否會讓現行的程式碼出現錯誤，以避免往後出現許多問題。

我覺得最好用的還是，不需要讓程式運行就可以測試的這一點。