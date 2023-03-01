---
title: "Rust: 生命週期( Lifetimes )"
date: 2023-02-10
draft: false
author: "Chen Yu Fan"
tags: ["Rust"]
---

**生命週期 ( lifetimes )** 會確保我們在需要引用的時候，它們都是有效的。

在 Rust 中，每個引用都是有生命週期的，簡單來說就是它的有效範圍。在大多情況下，生命週期都是隱藏且可以推導出來的，如同型別一樣也都是可以推導出來的。當型別有很多種可能的情況下，就要**詮釋型別**，同樣在生命週期下，引用以不同方式關聯的話，就要**詮釋生命週期**。

<!--more-->

---

## 透過生命週期預防迷途引用

生命週期最主要的目的就是要預防**迷途引用 ( dangling references )**：

```rust
fn main() {
    let r;
    
    {
        let x = 5;
        r = &x;
    }
    
    println!("r: {}", r);
}
```

在這裡嘗試使用已經離開作用域的引用，就會得到錯誤訊息，這是因為 `r` 所指向的數值已經離開作用域。這也表示變數 `x` **存在的不夠久**。那 Rust 要如何決定程式碼無效？它使用了**借用檢查器 ( borrow checker )** 來做檢查。

## 借用檢查器 ( Borrow checker )

Rust 編譯器有一個**借用檢查器 ( borrow checker )** 會比較作用域來檢測所有的借用是否有效。

依上面的程式碼為例：

```rust
fn main() {
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

`r` 的生命週期為 `'a`，`x` 為 `'b`，可以看到 `'b` 的生命週期區塊比 `'a` 還要小。而 `r` 引用了一個生命週期比他自己還短的變數 `x`，所以程式會報錯：**引用的對象比引用者存在的時間還短**。

以下為修正版本：

```rust
fn main() {
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```

此時 `x` 的生命週期 `'b` 比 `r` 的生命週期 `'a` 還長，Rust 就能知道 `r` 引用 `x` 是永遠有效的。

## 函式中的泛型生命週期

寫一個比較兩個字串切片誰比較長的函式。該函式會接收兩個字串切片並回傳一個字串切片：

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";
    
    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}
```

該函式接收的是字串切片的**引用**，而不是**字串**，因此我們不希望 `longest` 拿到參數的所有權。

> 字串切片本來就是引用，所以作為參數時可以不用加入 `&`

實作 `longest` 函式：

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

編譯後會出現生命週期的錯誤：

```bash
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/hello)
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++

For more information about this error, try `rustc --explain E0106`.
error: could not compile `chapter10` due to previous error
```

錯誤訊息表示回傳型別必須要有一個**泛型生命週期參數**，因為 Rust 不知道回傳的引用是 `x` 還是 `y`。事實上我們也不知道，因為這裡是交由 `if else` 區塊判定 `x` 和 `y` 哪個字串大就回傳哪個。

當我們不知道哪個字串會被回傳，也不知道傳遞進來的引用實際的生命週期為何，就會造成以上的問題。為了解決這個錯誤，必須要加上泛型生命週期參數來定義引用的關係，讓借用檢查器能夠分析。

## 生命週期詮釋語法

**生命週期詮釋 ( Lifetime Annotation )** 不會改變引用能存活多久，只是描述引用之間的相互關係，不會影響引用的生命週期。這就像函式簽名指定了一個泛型型別參數，該函式就能接受任意型別一樣。函式可以指定一個泛型生命週期參數，該函式就能接受任何生命週期。

生命週期參數的名稱必須以 ( `'` ) 作為開頭，通常是小寫且很短。大多數人都使用 `'a` 作為第一個生命週期詮釋。將生命週期詮釋放在 `&` 後面，再使用空格來詮釋引用的型別：

```rust
&i32        // 一個引用
&'a i32     // 一個有顯式生命週期的引用
&'a mut i32 // 一個有顯式生命週期的可變引用
```

## 在函式簽名中的生命週期詮釋

在此段簽名想要表達：當所有傳遞進來的參數都是有效的，那回傳的引用才會是有效的。以下會將生命週期命名為 `'a` 並加進每個引用：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

這樣更改完 `longest` 就能執行了。

此函式簽名告訴 Rust 有個生命週期 `'a`，函式的兩個參數都是字串切片，並且都有生命週期 `'a`，而回傳的字串切片也會和生命週期 `'a` 存活一樣久。

注意：當此函式簽名指定生命週期參數時，就不能變更任何傳入或傳出數值的生命週期。這個目的只是為了告訴借用檢查器一件事，拒絕任何沒有服從約束的數值。`longest` 函式並不需要知道 `x` 與 `y` 會存活多久，它只需要知道有某個作用域會被 `'a` 所取代。

當 `longest` 傳入實際的引用時，泛型生命週期 `'a` 取得的生命週期，會等於 `x` 與 `y` 的生命週期中較短的，而回傳引用的生命週期也會相同。

傳入不同實際生命週期的引用，來使生命週期詮釋能約束 `longest` 函式：

```rust
fn main() {
    let string1 = String::from("Long long string");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

- `string1` 在外部作用域結束前都有效；`string2` 在內部作用域結束前都有效。
- `result` 會取得某個引用直到內部作用域結束 ( _依照參數最短的生命週期_ )。

接下來做一點改變，如果將 `result` 移動到外部作用域做宣告，並保留 `result` 的賦值與 `string2` 在內部作用域。並將使用 `result` 的 `println!` 移動到外部作用域，編譯並且觀看結果：

```rust
fn main() {
    let string1 = String::from("Long long string");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

編譯後會看到以下的錯誤訊息：

```bash
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/hello)
error[E0597]: `string2` does not live long enough
 --> src/main.rs:6:44
  |
6 |         result = longest(string1.as_str(), string2.as_str());
  |                                            ^^^^^^^^^^^^^^^^ borrowed value does not live long enough
7 |     }
  |     - `string2` dropped here while still borrowed
8 |     println!("The longest string is {}", result);
  |                                          ------ borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `chapter10` due to previous error
```

錯誤訊息表示，需要讓 `result` 與 `println!` 有效的話，`string2` 必須要在外部作用域有效，因為 Rust 知道這個函式的參數與回傳值，都使用著相同的生命週期 `'a`。

當然我們都知道 `string1` 字串長度比較長，`result` 的結果會是 `string1` 的引用，而且 `string1` 還在作用域裡，照理來說應該是不會有問題才對。但是編譯器不懂，所以我們才會告訴 Rust 此函式所回傳引用的生命週期，會等於較短的生命週期。所以編譯器才會報錯，因為回傳可能會包含無效的引用。

## 深入理解生命週期

再來做一點改變，如果現在 `longest` 回傳的條件是永遠回傳第一個字串切片，那參數 `y` 就可以不需要指定生命週期：

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

這是因為 `x` 與回傳型別的生命週期都是 `'a`，但現在永遠只回傳 `x`，`y` 有沒有生命週期根本沒有任何差別。這也表示當函式回傳引用時，**回傳型別的生命週期必須符合其中一個參數的生命週期**。所以下面的程式碼是不會編譯成功的：

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("Long long string");
    result.as_str()
}
```

因為回傳值的生命週期與參數的生命週期無關。錯誤訊息為：

```bash
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/hello)
error[E0515]: cannot return reference to local variable `result`
  --> src/main.rs:11:5
   |
11 |     result.as_str()
   |     ^^^^^^^^^^^^^^^ returns a reference to data owned by the current function

For more information about this error, try `rustc --explain E0515`.
error: could not compile `chapter10` due to previous error
```

此時我們嘗試從函式中回傳 `result` 引用， 但 `result` 已經離開作用域，已經是迷途引用了，Rust 是不會允許使用迷途引用。解決這個問題最好的方法就是回傳有所有權的型別，而不是引用。

> 由上可知，生命週期語法是用來連接函式中不同參數與回傳值的生命週期。只要能連結，Rust 就能有足夠的資訊防止產生迷途指標或違反記憶體安全的操作。

## 在結構體中使用生命週期詮釋

結構體的所有定義都持有型別的所有權，然而結構體也可以持有引用。在這裡使用 `ImportantExcerpt` 來當作例子：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Can not find '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

- 如同泛型資料型別，在結構體名稱後加上泛型生命週期參數 ( `<'a>` )
- 在 `main` 裡產生一個結構體 `ImportantExcerpt` 實例，變數 `novel` 擁有 `String` 資料，而 `novel` 在 `ImportantExcerpt` 實例之前建立的，這表示 `novel` 不會比 `ImportantExcerpt` 早離開作用域，所以 `ImportantExcerpt` 裡的引用就能成立

## 省略生命週期

現在我們已經知道每個引用都有生命週期，那是否每一次都要寫出來呢？以下的函式是沒有詮釋生命週期還可以編譯成功的：

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    
    &s[..]
}
```

以上的程式曾經出現在[Rust: 所有權 ( Ownership )](/posts/rust-ownership/#%E5%88%87%E7%89%87-slice)，那為什麼它可以不用寫出生命週期？

如果是在早期版本 Rust ( 1.0 之前 )，上面的程式碼就真的無法編譯了，因為那個時候每個引用都必須要顯示生命週期，在當時此函式就會是：

```rust
fn first_word<'a>(s: &'a str) -> &'a str{
```

在寫了大量 Rust 程式碼後，Rust 團隊發現開發者會在特定情況反覆輸入同樣的生命週期詮釋，而這些情況都是可預期的，並且遵循一些明確的模式。所以 Rust 團隊將這些模式加入編譯器的程式碼中，讓借用檢查器可以依據這些規則自行推導生命生命週期。

這個讓 Rust 分析的模式稱為**生命週期省略規則 ( lifetime elision rules )** ，當你的程式碼符合該情形時，就可以不用寫出生命週期。

> 生命週期省略規則並不是完美的，還是有一些模稜兩可的生命週期，當編譯器無法猜出生命週期時，就會回傳錯誤給你，說明你必須自己指定生命週期

### 生命週期省略規則 ( Lifetime elision rules )

首先要先知道生命週期有兩種：

1. 在**函式**或**方法參數**上的生命週期稱為**輸入生命週期 ( input lifetimes )**
2. 在**回傳值**的生命週期則稱為**輸出生命週期 ( output lifetimes )**

編譯器會根據**三項規則**來推導沒有顯式生命週期的型別。第一個規則適用於輸入生命週期，而第二與第三個規則適用於輸出生命週期。如果三個規則跑完，還是沒有推斷出生命週期，編譯器就會停止並回傳錯誤。

第一個規則：**編譯器會給予每個引用參數一個生命週期參數**：

```rust
fn foo<'a>(x: &'a i32)                 // 1個參數；1個生命週期
fn foo<'a, 'b>(x: &'a i32, y: &'b i32) // 2個參數；2個生命週期
```

第二個規則：**如果只有一個輸入生命週期參數，所有輸出生命週期參數就等於此參數**：

```rust
fn foo<'a>(x: &'a i32) -> &'a i32
```

第三個規則：**如果輸入的生命週期參數裡有 `&self` 或 `&mut self`，很明顯這就是**方法 ( methods )**，那 `self` 的生命週期參數就等同於所有輸出生命週期參數**。

現在就可以根據上面的三個規則，來解釋 [函式中的泛型生命週期](#函式中的泛型生命週期) 裡的 `longest` 函式錯誤的原因了：

```rust
fn longest(x: &str, y: &str) -> &str {
```

先套用第一個規則，每個參數都有自己的生命週期。而這次有兩個參數：

```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {
```

第二個規則就不適用了，因為這裡不只有一個生命週期參數。第三個規則也不適用，`longest` 是函式而不是方法。經過三個規則後，編譯器還是無法推斷出型別的生命週期，這也就是程式會出現錯誤的原因。

## 屬於方法定義中的生命週期詮釋

> 那為什麼第三個規則只適用於**方法**呢？

結構體欄位的生命週期**永遠**需要宣告在 `impl` 關鍵字後方以及結構體名稱後方，因為**這些生命週期是結構體型別的一部分**。

在 `impl` 區塊中，**方法**所簽名的引用很可能會與**結構體欄位的引用**生命週期綁定，當然它們也可能是獨立的。

這裡使用 `ImportantExcerpt` 來作範例，首先使用一個方法名為 `level`，其參數就只有 `&self` 的引用，而回傳值是 `i32` 並不是引用：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

以下為第三個生命週期省略規則適用的範例：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Notice：{}", announcement);
        self.part
    }
}
```

首先，根據第一個生命週期省略規則，給予 `&self` 與 `announcement` 各自的生命週期。然後因為其中一個參數是 `&self`，適用於第三個生命週期省略規則，所以該回傳型別會取得 `&self` 的生命週期。

## 靜態生命週期

有個特殊的生命週期 `'static`，這表示該引用可以存活在整個程式期間。字串有 `'static` 生命週期：

```rust
let s: &'static str = "I have static lifetime";
```

此字串的文字會直接儲存在程式的執行檔中，永遠有效。

## 泛型型別參數、特徵界限與生命週期的組合

這邊使用一個函式來組合**泛型型別參數**、**特徵界限**與**生命週期**：

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement！{}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

- 這個函式會比較兩個字串切片，並回傳比較長的那一個
- `ann` 使用泛型型別 `T`，後面的 `where` 可以指定 `T` 有 `Display` 特徵。這表示這個 `ann` 參數可以使用 `{}` 方式印出來
- 生命週期也是一種泛型，所以生命週期參數 `'a` 與 泛型型別參數 `T` 都會一起寫在 `<>` 裡

---

## 結論

生命週期與所有權都是 Rust 管理記憶體很重要的機制，而透過借用檢查器將引用的作用域做比較，可以讓無效的引用在編譯期間就被發現。在三個生命週期省略規則下，Rust 大多的生命週期都可以自動被推導出來，這就跟這就跟型別一樣。

以上這些分析都是在編譯期間進行的，完全不會影響執行的效能😆！