---
title: "Spring Boot 專案(一) - 介紹與開發環境"
date: 2023-05-23
draft: false
author: "Chen Yu Fan"
tags: ["Java", "Spring", "Projects", "API"]
---

畢業後開始找工作，在高雄開有關於 Express 的職缺很少，就因為這樣我決定多學一個後端框架。選擇 Spring 是因為大二的時候學過 Java，看了半天的書記憶就回復得差不多了。學到現在其實不難，也多虧有學習其他框架，在概念上很快就可以理解了。

<!--more-->

## Spring Framework

[Spring Framework](https://spring.io/)，利用控制反轉容器 ( 依賴注入 ) 核心概念實現的 Web 應用開源框架。大幅簡化過去 Java EE Web 應用程式開發，目前該框架許多核心功能也都可以用於大部份 Java 應用。

- [控制反轉容器( 依賴注入 )](https://zh.wikipedia.org/zh-tw/Spring_Framework)：**控制反轉(IOC，Inverse Of Control)**，就是把**建立物件的權利交給框架**，也就是指將物件的建立、儲存與管理交給了 **Spring 容器** ( Spring 容器是Spring 中的一個核心模組，用於管理物件，底層可以理解為是一個Map集合 )。

Spring 擁有以下特性：

-   輕量級、鬆散耦合的
-   Spring Web 是一個設計良好的 Web MVC 框架
-   模組化組織
-   易於測試

> **鬆散耦合**：耦合指的是系統中元件互相依賴的程度，越少依賴則重複使用的彈性就越高。如果 A 模組與 B、C 和 D 模組有關係，那麼 A 就有很高的依賴，需要花更多時間維護。

## Spring Boot

Spring Boot 就是 Spring 的簡化版，是基於 Java 的開源框架，可以用於創建微服務(MicroService)。

-   易於理解開發應用
-   使配置變得簡單並彈性
-   提供有力的管理Rest端點
-   自動化配置
-   依賴套件管理簡單
-   內建Servlet容器
-   獨立打包直接運行

## VSCode 建立 Spring 專案

先在 VSCode 上下載 Java Extensions：

1. Language Support for Java(TM) by Red Hat 
2. Java Extension Pack 
3. Debugger for Java 
4. Maven for Java

再安裝 Spring 需要以下三個 Extensions：

1. Spring Initializr Java Support 
2. Spring Boot Tools
3. Spring Boot Dashboard

### 新建專案

跟著[官方文件](https://code.visualstudio.com/docs/java/java-spring-boot#_create-the-project) 來建立 Spring 專案。

在 VSCode 按下 F1，輸入 `Spring initializr: Create a Maven Project`，將 `Group Id` 與 `Artifact Id` 改成自己想要的名字，其他就下一步即可。

當然也可以預先安裝幾個套件，後面會解釋：

- Spring Web
- Spring Data JPA
- H2 Database
- Lombook

## 進入專案

在新的專案裡找到 `src/main` 裡面唯一的那個檔案 `BlogApplication.java` (這裡的專案名稱為 blog)，以下為程式碼：

```java
package demo.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BlogApplication {

  public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
  }
}
```

`@SpringBootApplication` 是一個復合的 Annotation，裡面有：

- `@SpringBootConfiguration`：繼承自 `@Configuration`，標註當前類別是配置類，並會將當前的類別標記為 `@Bean` 的實例加入到 Spring 容器中
- `@EnableAutoConfiguration`：啟動自動加入配置，導入你所需要的 jar 包
- `@ComponentScan`：掃描當前包與底下所有 
	- `@Controller`
	- `@Service`
	- `@Compoment`
	- `@Repository`

按下 Run > Start Debugging 來啟動 Spring Boot 專案 ( 快捷鍵 F5 )。

看到以下畫面就表示啟動成功了：

```text
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.1.0)
```

## 資料夾結構

![spring-boot-dir.png](/images/Spring-boot/spring-boot-dir.png)

- `main`：程式資料夾
- `BlogApplication.java`：程式進入點
- `static`：用於存放 css、js、images 靜態檔案
- `templates`：提供默認配置的模板引擎
- `application.properties`：設定所有的配置的地方，可參考[官方文件](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html)
- `test`：測試資料夾
- `BlogApplicationTests.java`：測試檔案

我會把 `application.properties` 改成 `application.yaml`，並同時更改 port：

```yaml
server:
  port: 8080
```

## 結語

很多網路上的教學都是預設的 Demo，一開始我也使用 Demo，但是寫著寫著就完成了整個 RESTful-API 了，忘了改也懶得改了 😂。順道一提，這個專案已經完成並放在我的 GitHub 了 [et860525/spring-restful](https://github.com/et860525/spring-restful)。

明天要去面試一個遠方的工作，以上。