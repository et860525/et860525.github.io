---
title: "Spring Boot 專案(四) - 連接到真實資料庫與 Swagger"
date: 2023-05-27
draft: false
author: "Chen Yu Fan"
tags: ["Java", "Spring", "Projects", "API"]
---

使用 H2 資料庫的好處就是，測試資料的時候很方便，不需要實際有一個資料庫。但這也代表你所存放的資料，在專案關閉後就會消失。

<!--more-->

## 修改 Entity

首先，先來加一點東西到 Todo Entity，打開 `./Entity/Todo.java` 並修改成：

```java
import lombok.Data;
import lombok.NoArgsConstructor;

import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;

import jakarta.persistence.*;
import java.util.Date;

@Entity
@Table
@Data
public class Todo {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  Integer id;

  @Column
  String task = "";

  @Column(insertable = false, columnDefinition = "int default 1")
  Integer status = 1;
  
  @CreatedDate
  @Column(updatable = false, nullable = false)
  Date createTime = new Date();
  
  @LastModifiedDate
  @Column(nullable = false)
  Date updateTime = new Date();
}
```

解釋一下新出現的 Annotation：
- `@Table`：表示在資料庫裡的 Table 名字，預設為 class 的名字 ( 你也可以 `@Table(name="Todos")` )
- `@Data`：一個懶人包，加上這個 Annotation 等於加上：
	- `@Getter` & `@Setter`
	- `@ToString`
	- `@EqualsAndHashCode`
	- `@RequiredArgsConstructor`
- `@Column` 裡面的各式參數：
	- `insertable`：在使用 "INSERT" 數據時，是否需要輸入該欄位的值
	- `columnDefinition`：直接在 SQL 創立一個默認值
	- `updatable`：該欄位是否可以更新，如果 false，就不能進行修改
	- `nullable`：可否使用空值

你可以執行來試試看，是不是會新增幾個欄位。

## 連接到 MySQL Database

先安裝 MySQL 需要的依賴項：

- VSCode 安裝
- `pom.xml`
```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>8.0.33</version>
</dependency>
```

接著到 `application.yaml` 來做設定：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
  datasource:
    url: jdbc:mysql://localhost/todo
    username: username
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
```

更改 `username` 和 `password` 成自己需要的，如果跟我一樣是使用 VM 來架設資料庫的，要更改 `localhost` 成 VM 的 IP。

完成後就可以運行專案了。

## Swagger API

後端開發 API 時，通常都會產生相對應的文件，讓前端人員或其他使用者能知道這些 API 該如何使用。

最有名的 API 工具應該就是 [Swagger](https://swagger.io/) 了，Swagger 可以根據寫出來的程式碼，自動轉換成 Open API 文件，使這些 API 用視覺化的方式呈現，並且也支援直接測試的介面。由 Swagger 產生出來的統一文件與測試介面，可以大幅減少溝通與維護的成本。

我這裡會使用 [springdoc-openapi](https://springdoc.org/v2/) 來做設定。

### 設定

首先，先安裝 `springdoc-openapi`：

- VSCode 安裝
- `pom.xml`
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.1.0</version>
</dependency>
```

好了，可以直接重啟專案了，沒有錯就是這麼簡單。

- `http://server:port/context-path/swagger-ui.html` 可以看到 Swagger 的 UI 畫面
- `http://server:port/context-path/v3/api-docs` 可以看到 OpenAPI Json 格式
	- `server`：伺服器名稱或是 IP
	- `port`：伺服器的 port
	- `context-path`：路徑

此專案為 `http://localhost:8080/swagger-ui/index.html` 或是 `http://localhost:8080/v3/api-docs`。

![spring-boot-swagger-ui.png](/images/Spring-boot/spring-boot-swagger-ui.png)

還可以再多顯示一些東西，例如詳細表明這個 API 會做什麼事情，到 `TodoController.java`：

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;

//...

@RestController
@RequestMapping("/api")
@Tag(name="Todo List!")
public class TodoController {
  @Autowired
  TodoService todoService;

  @Operation(summary = "Get all todo")
  @GetMapping("/todos")
  public ResponseEntity<Iterable<Todo>> getTodos() {
    Iterable<Todo> todoList = todoService.getTodos();
    return ResponseEntity.status(HttpStatus.OK).body(todoList);
  }

  @Operation(summary = "Get a todo")
  @GetMapping("/todos/{id}")
  public Optional<Todo> getTodo(@PathVariable Integer id) {
    Optional<Todo> todo = todoService.findById(id);
    return todo;
  }

  @Operation(summary = "Create todo")
  @PostMapping("/todos")
  public ResponseEntity<Integer> createTodo(@RequestBody Todo todo) {
    Integer result_todo = todoService.createTodo(todo);
    return ResponseEntity.status(HttpStatus.CREATED).body(result_todo);
  }

  @Operation(summary = "Update todo")
  @PutMapping("/todos/{id}")
  public ResponseEntity<String> updateTodo(@PathVariable Integer id, @RequestBody Todo todo) {
    Boolean result_todo = todoService.updateTodo(id, todo);
    if (!result_todo) {
      return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Status fields required!");
    }
    return ResponseEntity.status(HttpStatus.OK).body("");
  }

  @Operation(summary = "Delete todo")
  @DeleteMapping("/todos/{id}")
  public ResponseEntity<String> deletTodo(@PathVariable Integer id) {
    Boolean result_todo = todoService.deleteTodo(id);
    if (!result_todo) {
      return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Id not exist");
    }
    return ResponseEntity.status(HttpStatus.NO_CONTENT).body("");
  }
}
```

- `@Tag`：增加敘述
- `@Operation(summary, description)`：對單一個端點加入敘述
- `@Hidden`：隱藏一個端點
- `@ApiResponse(responseCode, description)`：該端點所回傳的 statusCode 與敘述

![spring-boot-swagger-ui-annotation.png](/images/Spring-boot/spring-boot-swagger-ui-annotation.png)

不只更改單個控制器的敘述，整篇文件的一些敘述也可以更改。打開進入點 `RestfulApplication.java`：

```java
//...
import org.springframework.context.annotation.Bean;

import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;

@SpringBootApplication
public class RestfulApplication {

  public static void main(String[] args) {
    SpringApplication.run(RestfulApplication.class, args);
  }

  @Bean
  public OpenAPI todoOpenAPI() {
    return new OpenAPI()
                .info(new Info().title("Todo API")
                .description("Spring todo application")
                .version("v0.0.1")
                .license(new License().name("Apache 2.0").url("http://springdoc.org")))
                .externalDocs(new ExternalDocumentation()
                .description("Todo API Documentation")
                .url("https://springshop.wiki.github.org/docs"));
  }
}
```

![spring-boot-swagger-ui-openAPI.png](/images/Spring-boot/spring-boot-swagger-ui-openAPI.png)

如果你曾經有使用 SpringFox，那可以參考 [Migrating from SpringFox](https://springdoc.org/v2/#migrating-from-springfox) 來查看更多資訊。

---

## 結語

使用 Swagger 真的很方便，以前曾經後端完成後，開發前端在串接 API 時，有時候就會忘記版本是幾版，或是參數該丟甚麼，有了 Swagger 就可以馬上查詢，我就不必在每一個 methods 上加註解。

