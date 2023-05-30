---
title: "Spring Boot 專案(六) - 資料庫關聯: One-to-Many & Many-to-One"
date: 2023-05-30
draft: false
author: "Chen Yu Fan"
tags: ["Java", "Spring", "Projects", "API"]
---

實現資料一對多與多對一的關聯。讓一個 Todo 只能有一個 Category，一個 Category 會有多個 Todo。

<!--more-->

在一對多或多對一的情況下，為了避免雙向引用導致的無限遞歸(Infinite Recursion) 的問題，還必須要使用 `@JsonManagedReference` 與 `@JsonBackReference` 註解來解決這個問題，這兩個註解通常會配對使用，通常是父子關係。

- `@JsonManagedReference` 標註的屬性會被序列化
- `@JsonBackReference` 標註的屬性不會被序列化

Todo Entity 使用 `@ManyToOne` 來執行多對一，這裡有幾個 `@ManyToOne` 的相關設定：

- `CascadeType.REMOVE`：當關聯實體被**刪除**時，有關係的實體也會被刪除
- `CascadeType.MERGE`：當關聯實體更新，有關聯的實體也會被更新
- `CascadeType.REFRESH`：關聯刷新，資料庫也會更新
- `CascadeType.ALL`：以上所有關聯操作的權限

其用法為 `@ManyToOne(cascade = CascadeType.ALL)`。

## 實體修改與建立

接下來，來改變 `Todo.java` 實體：

```java
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
  
  @JsonBackReference
  @ManyToOne(cascade = CascadeType.ALL)
  @JoinColumn(name="category_id")
  private Category category;
}
```

- `@JoinColumn(name="category_id")`：代表這個欄位將會關聯到 Category 的 Id

接下來建立一個 `Category.java` 的實體：

```java
@Entity
@Table
@Data
public class Category {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  Integer id;

  @Column
  public String name;

  @JsonManagedReference
  @OneToMany(cascade = CascadeType.ALL, mappedBy = "category")
  @EqualsAndHashCode.Exclude
  private Set<Todo> todos;
}
```

## Category 相關檔案

- `CategoryDao.java`
```java
package com.restful.restful.Dao;

import org.springframework.data.repository.CrudRepository;

import com.restful.restful.Entity.Category;


public interface CategoryDao extends CrudRepository<Category, Integer> {
	
}
```

- `CategoryService.java`
```java
@Service
public class CategoryService {
  @Autowired
  CategoryDao categoryDao;
  
  public Optional<Category> getTodosByCategoryId(Integer id) {
  	Optional<Category> category = categoryDao.findById(id);
  	return category;
  }
}
```

- `CategoryController.java`
```java
@RestController
@RequestMapping("/api")
public class CategoryController {
  @Autowired
  CategoryService categoryService;
  
  @GetMapping("/categories/{id}/todos")
  public ResponseEntity<Optional<Category>> getTodosByCategoryId(@PathVariable Integer id) {
    Optional<Category> category = categoryService.getTodosByCategoryId(id);
    return ResponseEntity.status(HttpStatus.OK).body(category);
  }
}
```

## 測試結果

到 `http://localhost:8080/h2-console` 輸入測試資料：

```java
Insert into CATEGORY (NAME) values ('居家');

INSERT INTO TODO (TASK, STATUS, UPDATE_TIME, CREATE_TIME, CATEGORY_ID) values ('打電腦', 1, '2023-05-30 11:00:00', '2023-05-30 11:00:00', 1);
```

到 `http://localhost:8080/api/categories/1/todos` 就可以看到結果。

---

## 結語

資料庫關聯是資料庫很重要的一環，使用主鍵(primary key)與外鍵(foreign key) 的方式串起兩個 table 之間的關係，讓電腦來判斷相互關聯的依據，總能比人為新增更加嚴謹。