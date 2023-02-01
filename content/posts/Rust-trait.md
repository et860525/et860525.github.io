---
title: "Rust: 特徵( Trait )"
date: 2023-02-01
draft: false
author: "Chen Yu Fan"
tags: ["Rust"]
---

特徵( trait )，是定義特定型別與其他型別共享的功能。可以使用**特徵界限 ( trait bounds )** 來指定泛型型別為擁有特定行為的任意型別。

> 特徵類似於其他語言常稱作**介面 ( interfaces )** 的功能，但還是有些差異。

<!--more-->

## 定義特徵

舉例，現在我們有兩個結構體各自擁有不同種類與不同數量的文字：

- `NewsArticle` 儲存特定地點的新聞故事
- `Tweet` 則有最多 280 字元的內容，且有個欄位來判斷是全新的推文、轉推或其他推文的回覆。

我們想要建立一個多媒體資料庫，來顯示可能存在於 `NewsArticle` 或 `Tweet` 實例的總結 ( summary )。要達成這個目的，我們會呼叫該實例的 `summarize` 方法來獲得實例裡的 summary。

新建立一個檔案為 `src/lib.rs`，並在此使用 `trait` 關鍵字定義一個 `Summary` 特徵：

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

- `fn summarize(&self) -> String` 宣告方法簽名來描述有實作此特徵的型別行為

> 特徵本體中可以有多個方法，每行會有一個方法簽名並都以分號做結尾。

**每個有實作此特徵的型別，都必須提供其自訂行為的方法本體**。編譯器會強制要求任何有 `Summary` 特徵的型別都要有定義相同簽名的 `summarize` 方法。

## 型別實作特徵

在 `src/lib.rs` 中定義 `Summary` 特徵完成後，就可以在資料庫裡實作它：

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{} {} 著 ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

在 `impl` 之後我們加上想要實作的特徵，然後使用 `for` 關鍵字加上想要實作的型別名稱。並在 `impl` 區塊裡定義特徵所指定的方法。

這裡使用的 `rust_aggregator` 是一開始 `cargo new <name>` 的名字，我們已經在函式庫裡加入對 `NewsArticle` 和 `Tweet` 實作 `Summary` 特徵，如果是 `crate` 的使用者只要直接呼叫即可。唯一的不同就是，使用的特徵也必須加入到作用域中。

以下是我的 `rust_aggregator` 函式庫的 `main.rs`：

```rust
use rust_aggregator::{self, Summary, Tweet};

fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 則新推文：{}", tweet.summarize());
}
```

此程式碼會印出：`1 則新推文：horse_ebooks: of course, as you probably alreadyknow, people`。

## 預設實作

將特徵內的一些方法預先定義實作，就不用要求型別實作這些方法了。如果特定型別想要更改特徵內方法的實作，直接寫上方法就能覆蓋預設的實作。以下是定義預設實作：

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(閱讀更多...)")
    }
}
```

當 `summarize` 方法此時有預設實作，就不會強制要求型別需要設定此方法：

```rust
fn main() {
    let article = NewsArticle {
        headline: String::from("Penguins win the Stanley Cup Championship!"),
        location: String::from("Pittsburgh, PA, USA"),
        author: String::from("Iceburgh"),
        content: String::from(
            "The Pittsburgh Penguins once again are the best \
             hockey team in the NHL.",
        ),
    };

    println!("有新文章發佈！{}", article.summarize());
}
```

如果 `NewsArticle` 沒有實作 `summarize` 方法，此程式碼就會印出 `有新文章發佈！(閱讀更多...)`。

## 特徵作為參數

使用特徵來定義函式所接受的參數。在 `Summary` 特徵定義一個新的 `notify` 函式，它會使用自己的參數 `item` 來呼叫 `summarize` 方法，所以此參數的型別會預期有 `Summary` 特徵。

```rust
pub fn notify(item: &impl Summary) {
    println!("頭條新聞！{}", item.summarize());
}
```

`item` 參數指定實際型別用的是 `impl` 關鍵字加上特徵名稱，表示此參數會接受任何有實作指定特徵的型別 ( `NewsArticle` 或 `Tweet` ) 。但如果用其他型別像是 `String` 或 `i32` 來呼叫此程式碼則會無法編譯，因為那些型別沒有實作 `Summary`。

> 這能讓函式的參數只接受特定特徵的型別

### 特徵界限語法

`impl Trait` 是一個更長格式的語法糖，而這個格式稱為**特徵界限 ( trait bound )**，它長得像：

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("頭條新聞！{}", item.summarize());
}
```

`impl Trait` 語法比較方便，且在簡單的案例中可以讓程式碼比較簡潔，而特徵界限適合其他複雜的案例。

舉例參數是兩個實作 `Summary` 參數，使用 `impl Trait`：

```rust
pub fn notify(item1: &impl Summary, item2: &impl Summary) {
```

如果函式允許 `item1` 與 `item2` 是不同型別，就可使用 `impl Trait`。但如果兩個參數是同一型別，就要改成特徵界限 ( trait bound ) 的方式：

```rust
pub fn notify<T: Summary>(item1: &T, item2: &T) {
```

這樣 `item1` 與 `item2` 的型別就必須要相同了。

### 使用「+」指定多個特徵界限

在上一章泛型有提到用兩個特徵的函式：

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
```

這兩個特徵分別為：

- `PartialOrd` 判斷邏輯
- `Copy` 將變數指派給另一個變數

這個方法也一樣可以用在參數裡。假設 `notify` 中的 `item` 不只能呼叫 `summarize` 方法，還能顯示格式化訊息的話，也可以使用 `+`：

- `impl Trait`
	```rust
	pub fn notify(item: &(impl Summary + Display)) {
	```
- 特徵界限
	```rust
	pub fn notify<T: Summary + Display>(item: &T)
	```

### 使用「where」使特徵界限更清楚

如果今天每個泛型都有自己的特徵界限，會造成函式簽名難以閱讀：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
```

使用 `where` 讓一切看起來不那麼複雜：

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
where {
    T: Display + Clone,
    U: Clone + Debug,
}
```

## 回傳有實作特徵的型別

在回傳型別的位置使用 `impl Trait` 語法，來回傳某個有實作特徵的型別：

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```

但是如果使用 `impl Trait` 的話就只能回傳單一型別。舉例來說，雖然程式碼指定回傳的型別為 `impl Summary`，但是將程式碼寫成可能會回傳 `NewsArticle` 或 `Tweet` 就會無法執行：

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
            reply: false,
            retweet: false,
        }
    }
}
```

對於可能返回 `NewsArticle` 或 `Tweet` 的話是不被允許的，因為 `impl Trait` 語法會限制在編譯器中最終決定的型別。

## 透過特徵界限來選擇性實作方法

在有使用泛型型別參數 `impl` 區塊中使用特徵界限，就可以有選擇性的實作特定型別的實作方法：

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("最大的是 x = {}", self.x);
        } else {
            println!("最大的是 y = {}", self.y);
        }
    }
}
```

簡單來說，第一個 `impl<T>` 因為它沒有任何特徵，所有型別的 `Pair` 都可以使用這個方法；第二個 `impl` 則有限定型別，如果型別的特徵是 `<T: Display + PartialOrd>` 那它就可以使用，如果沒有則無法使用這個方法。

而這種滿足特徵界限的型別實作特徵，則稱之為**全面實作 ( blanket implementations )**，這被廣泛的使用在 Rust 標準函式中。舉個例子，標準函式庫會對任何有實作 `Display` 特徵的型別實作 `ToString`，以下是類似於標準函式庫中的 `impl` 區塊：

```rust
impl<T: Display> ToString for T {
    // --省略--
}
```

標準函式庫有此全面實作，這樣就能將整數與字元轉為 `String`：

```rust
let s1 = 3.to_string();
let s2 = 'c'.to_string();
```

## 結論

上述有提到 `Trait`與其他程式語言的 `Interface` 有些不同，其中最大的不同在於：`Interface` 主要是根據特定的 object 來設定接口的；`Trait` 則可以對現有的型別來實現實作：

```rust
trait Hash {
    fn hash(&self) -> u64;
}

impl Hash for bool {
    fn hash(&self) -> u64 {
        if *self { 0 } else { 1 }
    }
}

impl Hash for i64 {
    fn hash(&self) -> u64 {
        *self as u64
    }
}
```

在 `Hash` 的作用域內就可以使用 `true.hash()` 這樣的寫法。

---

## 引用

- [特徵：定義共同行為](https://rust-lang.tw/book-tw/ch10-02-traits.html)
- [Traits: Defining Shared Behavior](https://doc.rust-lang.org/book/ch10-02-traits.html)
- [Abstraction without overhead: traits in Rust](https://blog.rust-lang.org/2015/05/11/traits.html)