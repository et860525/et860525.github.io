---
title: "Spring Boot 專案(三) - 分層式架構"
date: 2023-05-25
draft: false
author: "Chen Yu Fan"
tags: ["Java", "Spring", "Projects", "API"]
---

這一篇會整理分層式架構的資訊，並實際運用於 RESTful-API 專案裡。為了不要讓整篇文章太長，我只會放幾個 Method 來展示，更完整的程式碼可以參考 [et860525/spring-restful](https://github.com/et860525/spring-restful)。

<!--more-->

## 分層架構模式 ( Layered Architecture )

![layered-architecture.png](/images/Spring-boot/layered-architecture.png)

看一下就會發現，這不就是很多框架會用到的 MVC 嗎？確實，不過 MVC 是簡化的分層式架構。

每一層各自代表：

1. **表示層 (Presentation Layer)**：處理所有用戶介面和瀏覽器通訊 (Controller & View)
2. **業務層 (Business Layer)**：處理軟體本身特定的業務邏輯 (Service)
3. **持久層 (Persistence Layer)**：專注於資料的儲取活動 (DAO)
4. **資料庫層 (Database Layer)**：訪問資料 (Database)

以上都只是抽象的概念，所以可以根據專案的大小與複雜程度來分類。有些比較小的應用可能會把 Service 與 DAO 寫在一起。

優點為：

- **易於理解與開發**
- **良好的安全性**
- **可測試性**：因為分層明確，每一層都可以單獨進行測試。

缺點為：

- 因為每一層都是上下互相關聯的，只要有一層需要修改，它的上下層就要修改
	- 修改業務層，那表示層與持久層就要一併修改
- 當程式越大，分層架構可能會擴大到不可思議，對程式的維護與耦合性會有影響。所以分層架構經常會被用在**單體應用 (Monolithic Applications)** 當中。不過，其實也能在其他架構裡發現分層式架構的身影，在微服務架構裡，還是可以選擇使用分層式架構，作為某個軟體的架構，由此可見，分層式架構可以與其他架構互相結合

## 實作 Controller

終於要進到實作的部分了，首先，如果沒有預先安裝 Spring Web 套件，請先安裝：

- VSCode 安裝
- 或是 `pom.xml`：
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

先從 Controller 開始，首先建立 Controller 資料夾

```text
|-- src
    |-- main
        |-- java
             |-- com.xxx.xxxxx
	            |-- Controller
                    |-- TodoController.java
                |-- Entity
                    |-- Todo.java
                |-- Dao
                    |-- TodoDao.java
                |-- TodoListApplication.java # 程式進入點
```

在 `TodoController.java` 放入程式碼：

```java
import com.restful.restful.Entity.Todo;
import com.restful.restful.Service.TodoService;
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;


@RestController
@RequestMapping("/api")
public class TodoController {
	@Autowired
	TodoService todoService;

	@GetMapping("/todos")
	public ResponseEntity<Iterable<Todo>> getTodos() {
		Iterable<Todo> todoList = todoService.getTodos();
		return ResponseEntity.status(HttpStatus.OK).body(todoList);
	}

	@GetMapping("/todos/{id}")
	public Optional<Todo> getTodo(@PathVariable Integer id) {
		Optional<Todo> todo = todoService.findById(id);
		return todo;
	}

	@PostMapping("/todos")
	public ResponseEntity<Integer> createTodo(@RequestBody Todo todo) {
		Integer result_todo = todoService.createTodo(todo);
		return ResponseEntity.status(HttpStatus.CREATED).body(result_todo);
	}
}
```

**標註(Annotation)** 的各個功用：
- `@RestController`：表示這個類別是 RESTful Web 服
- `@RequestMapping("/api")`：映射一個路徑，下面的 metods 都會從這個路徑開始
- `@Autowired`：依賴注入物件，會依注入對象(這裡是 `TodoService`) 的類別型態，來選擇容器中相符的物件來注入
- `@GetMapping("/todos")`：GET 請求
- `@RequestBody`：接收 POST 發送過來的 request Body
- `@PathVariable`：獲得 URL 裡面的參數

> `@PathVariable` 與 `@RequestParam` 做的事情一樣，但是 URL 呈現的本質是不一樣的。以上面的 id 為例子：
> - `@PathVariable`: `http://host:port/path/3`
> - `@RequestParam`: `http://host:port/path?id=3`

**ResponseEntity**：是用來處理 HTTP 的 Response ([Class ResponseEntity\<T>](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html))
- `status`：回傳的 Code (這裡使用 [HttpStatus](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/HttpStatus.html)
- `body`：回傳的內容

> 更多詳細的資訊，可以到 [Spring Framework API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/index.html) 查詢。

還沒完成喔！還要完成 Service 來獲得資料。

## 實作 Service

建立 Service 資料夾

```text
|-- src
    |-- main
        |-- java
             |-- com.xxx.xxxxx
	            |-- Controller
                    |-- TodoController.java
                |-- Entity
                    |-- Todo.java
                |-- Dao
                    |-- TodoDao.java
                |-- Service
	                |-- TodoService.java
                |-- TodoListApplication.java # 程式進入點
```

在 `TodoService.java` 放入程式碼：

```java
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.restful.restful.Dao.TodoDao;
import com.restful.restful.Entity.Todo;

@Service
public class TodoService {
	@Autowired
	TodoDao todoDao;

	public Iterable<Todo> getTodos() {
		return todoDao.findAll();
	}
	
	public Integer createTodo(Todo todo) {
		Todo result_todo = todoDao.save(todo);
		return result_todo.getId();
	}
	
	public Optional<Todo> findById(Integer id) {
		Optional<Todo> todo = todoDao.findById(id);
		return todo;
	}
}
```

- `@Service` 表示這是一個 Service
- `@Autowired` 注入 `TodoDao`

恭喜，這樣就建立好了，可以開始執行專案了。

## 執行結果

建立：

![spring-boot-POST.png](/images/Spring-boot/spring-boot-POST.png)

查詢全部：

![spring-boot-GET-all.png](/images/Spring-boot/spring-boot-GET-all.png)

查詢指定 Id：

![spring-boot-GET-one.png](/images/Spring-boot/spring-boot-GET-one.png)

---

## 結語

標註(Annotation) 讓一切都很清楚，說到標註，如果今天是前後端混和的話 `@RestController`  就會變成 `@Controller`，這個就可以讓人一眼就分辨出來，這個後端是不是單純傳送 API 了。

下一篇我會改變一下 Todo Entity，並把資料放到資料庫裡，再用 Spring 去連結它。