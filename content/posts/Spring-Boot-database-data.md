---
title: "Spring Boot 專案(二) - 資料庫與互動"
date: 2023-05-24
draft: false
author: "Chen Yu Fan"
tags: ["Java", "Spring", "Projects", "API"]
---

基本的設定完成後，現在就要來設計資料的存放問題。這裡會先使用 H2 Database，後面我會在 Docker 上部屬一個 MySQL 來存放資料。

<!--more-->

---

## Spring Boot Database - H2

H2 是一個由 Java 撰寫的關聯型資料庫，也是一個記憶體資料庫(In memory database)，將內容存放在**記憶體(內存)** 當中，而非傳統型資料庫存放在外部記憶體中。

> 所以一但專案關閉再重啟，資料就會不見

如果在前一篇沒有預先安裝，這裡提供兩種安裝的方式：

1. 在 VSCode 安裝方式：`F1 -> Maven: Add a dependency -> 輸入套件名稱`。( 必須要注意 `groupId` 與 `artifactId` 是否與自己需要的套件一致 )
2. 直接在 `pom.xml` 新增以下
```xml
<dependencies>
	<dependency>
	    <groupId>com.h2database</groupId>
	    <artifactId>h2</artifactId>
	    <scope>runtime</scope>
	</dependency>
	
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-data-jpa</artifactId>
	</dependency>
	
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-devtools</artifactId>
	    <optional>true</optional>
	</dependency>
</dependencies>
```

以上可以看到多新增一個 JPA 與 devtools，這裡來解釋一下。

- 簡單介紹一下 JPA，主要的功能是用來實現 ORM 的**開發介面**，透過使用簡單的**標註(Annotation)** 將指定的對象與關係表的關係映射
- devtools 是熱部屬，當有檔案改變時，會自動重啟伺服器

首先，先到 `application.yaml` 檔案裡設定 DB 的環境：

```yaml
server:
  port: 8000 # 伺服器的port號

spring:
  h2:
    console:
      enabled: true
    datasource:
    url: jdbc:h2:mem:todolist # h2 database 連接位址
    driver-class-name: org.h2.Driver # 配置driver
    username: sa # database 用戶名
    password: # database 密碼

  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
```

設定完後啟動應用，進入 `http://localhost:8000/h2-console/` 會看到一個 h2 DB 的介面，按下 Connect 就可以查看資料庫了。這樣就完成建立 todolist  資料庫。

![Spring-h2-connect.png](/images/Spring-boot/Spring-h2-connect.png)

> 如果 JDBC URL 欄位裡不是像上圖一樣的網址，那就要更改

## 建立實體

實體就是資料類別

```text
|-- src
    |-- main
        |-- java
             |-- com.xxx.xxxxx
                  |-- Entity
                    |-- Todo.java
                  |-- TodoListApplication.java # 程式進入點
```

並在 `Todo.java` 裡加入：

```java
package com.demo.demo.Models;

import jakarta.persistence.*; // 引入 JPA

@Entity
@Table
public class Todo {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Integer id;
    
    @Column
    String task;
    
    @Column
    Integer status;
    
    public Integer getId() {
        return id;
    }
    
    public void setId(Integer id) {
        this.id = id;
    }
    
    public String getTask() {
        return task;
    }
    
    public void setTask(String task) {
        this.task = task;
    }
    
    public Integer getStatus() {
        return status;
    }
    
    public void setStatus(Integer status) {
        this.status = status;
    }
}
```

- `@Entity`：宣告此類是一個實體，並映射至一個數據表，**Entity** 名稱預設為 class 名稱
- `@Table`：映射資料表的名稱，預設為Entity名稱
- `@Id`：聲明為一個主鍵
- `@GeneratedValue(strategy = GenerationType.IDENTITY)`：主鍵生成的策略
	- GenerationType.TABLE用一個特定數據庫保存主鍵
	- GenerationType.SEQUENCE 用序列的方式
	- GenerationType.IDENTITY 自動增長生成
	- GenerationType.AUTO 程式指定主鍵
- `@Column`：聲明是個欄位，可以針對此欄位設置參數，如name, unique, length等

再次進入到 `http://localhost:9100/h2-console/`，Connect 之後就可以看到 Todo table 了。

## Lombok 套件

當建立實體(Entity) 時，必須要新增 getter & setter 方法，當 class 裡面的成員一多，程式碼就會存在太多的 getter & setter 方法。所以這裡可以使用 [Lombok](https://projectlombok.org/) 來解決這個問題，只要在類別上使用 `@Getter` `@Setter` Annotation 標註即可。

```xml
<dependencies>
	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<version>1.18.26</version>
	</dependency>
</dependencies>
```

### @Getter & @Setter

將程式碼改成：

```java
package com.caili.todolist.entity;

import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;

import javax.persistence.*;

@Entity
@Table
@Getter @Setter
@NoArgsConstructor
public class Todo {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Integer id;
    
    @Column
    String task;
    
    @Column
    Integer status;
}
```

沒幾下就完成上面所寫的一大堆 getter & setter 方法。

Lombok 還有其他的功能，可以參考 [Java - 五分鐘學會 Lombok 用法](https://kucw.github.io/blog/2020/3/java-lombok/)。

## JPA (Jakarta Persistence API)

JPA 全名為 [Jakarta Persistence API](https://jakarta.ee/specifications/persistence/3.0/)，以前稱為 [Java Persistence API](https://www.oracle.com/java/technologies/persistence-jsp.html)。
是官方所提出的 ORM 規範，利用 **標註(Annotation)** 將指定的對象與關係表的關係映射，就是上面所看到的 `@Entity`、`@Table`、`@Column`。

> JPA 只是一個介面( interface )，Hibernate 就是實現了 JPA 介面的 ORM 框架

### Spring Data JPA

[Spring Data JPA](https://spring.io/projects/spring-data-jpa) 是 Spring 基於 ORM 框架與 JPA 規範的一套應用框架，底層則是使用 [Hibernate](https://www.javatpoint.com/hibernate-tutorial)，可以讓開發者用程式碼實現對資料庫溝通與操作。

### Spring Data JPA interface

- Repository 最頂層
- `CrudRepository implements Repository`，提供 CRUD 功能
- `PagingAndSortingRepository implements CrudRepository`，添加分頁和排序的功能
- `JpaRepository implements PagingAndSortingRepository`

## 實作一個 DAO

上面有提到，JPA 只是一個 interface，所以建立一個 DAO 讓他實作：

```text
|-- src
    |-- main
        |-- java
             |-- com.xxx.xxxxx
                  |-- Entity
                    |-- Todo.java
                  |-- Dao
                    |-- TodoDao.java
                  |-- TodoListApplication.java # 程式進入點
```

`TodoDao.java` 的程式碼為：

```java
import com.caili.todolist.entity.Todo;
import org.springframework.data.repository.CrudRepository;

// 第一個參數為訪問的實體，第二參數是這個Entity ID的資料型態
public interface TodoDao extends CrudRepository<Todo, Integer> {

}
```

這樣就可以開始使用 JPA 所提供的 CRUD了。

---

## 結語

寫下來印象最深刻的應該就是 Lombok 套件，一次可以簡寫很多東西，這讓我想到在使用 Mongoose 時，還會再使用 [typegoose](/posts/jwt-authentication-models-jwt/#建立資料庫欄位) 來簡寫欄位裡相關的設定。

下一篇會從**三層式架構**開始說起，這一個架構在我使用 Express 就一直在使用了。