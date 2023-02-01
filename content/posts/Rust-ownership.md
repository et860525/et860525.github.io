---
title: "Rust: 所有權( Ownership )"
date: 2023-01-30
draft: false
author: "Chen Yu Fan"
tags: ["Rust"]
---

在一開始撰寫文章時，本來想以寫比較久的 TypeScript 來做部落格的開頭文章，但在過年前接觸到 Rust 這個程式語言，就順勢把最近學到的東西放上來。等到把在 TypeScript 遇到的問題整理一下再寫成一個系列放上來。

**所有權 ( ownership )** 是 Rust 用來 **管理程式記憶體的一系列規則**，讓 Rust 不需要[垃圾回收 ( Garbage collection )](https://zh.wikipedia.org/zh-tw/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6_(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%B8)) 就可以保障記憶體的安全。

<!--more-->

所有程式都需要在執行時管理它們使用記憶體的方式，這裡有常見的兩種：

1.  語言本身就有垃圾回收機制，在程式執行時不斷尋找不再使用的記憶體
2.  開發者必須親自分配和釋放記憶體

而 Rust 選擇第三種方式：記憶體由**所有權**系統管理，編譯器會在編譯時加上一些規則檢查，如果有違規，程式就無法編譯。

> 所有權的規則完全不會降低執行程式的速度

---

## 堆疊 ( Stack ) 與堆積 ( Heap )

堆疊與堆積都是提供程式碼在執行時能夠使用的記憶體部分，但組成的方式不一樣。

**堆疊 ( Stack )** 會按照順序依序排列它們，並以相反順序移除，這也稱之為 **後進先出 ( last in, first out )**。所有在堆疊上的資料都必須是已知的固定大小，在編譯期間屬於未知或可能變更大小的資料則必須儲存在於堆積。

> 想像堆疊是盤子，當加入盤子時只能疊在最上方，想要拿走盤子也只能拿最上面的盤子，想從中間或最下面插入或拿走盤子都不行。

**堆積 ( Heap )** 相比堆疊就沒有組織，當資料放進堆積時，記憶體分配器 ( memory allocator )會找到一塊夠大的空位，並標記已占用，然後回傳一個指標 ( pointer ) 指向該位址。這一整個過程稱之為 **堆積上分配 ( allocating on the heap )** 或簡稱為分配。也因為指標是固定的大小，它可以被存在堆疊上，當需要存取實際的資料時，就透過指標去獲得即可。

資料在堆疊與堆積的比較：

- 將資料推入堆疊會比堆積分配還快，因為分配器不用去尋找空位，其位置永遠在堆疊的最上方。堆積就需要比較多的步驟，分配器必須要先找到一個足夠的空位，並做紀錄為下一次分配做準備。

- 獲得資料的時間也是堆疊最快，因為堆積必須要透過指標才能找到。如果處理器與記憶體間跳轉的時間越少，則速度就越快。

> 理解所有權主要就是為了管理堆積。

## 所有權規則

-   Rust 中每個數值都有個**擁有者 ( owner )**。
-   同時間只能有一個擁有者。
-   當擁有者離開**作用域 ( scope )** 時，數值就會被丟棄。

## 作用域 ( scope )

```rust
fn main() {
    {                      // s 在此處無效，因為它還沒宣告
        let s = "hello";   // s 在此開始視為有效

        // 使用 s
    }                      // 此作用域結束， s 不再有效
}
```

兩個重要的時間點：

-   當 `s` **進入**作用域時，它是有效的。
-   它持續被視為有效直到它**離開**作用域為止。

## 記憶體與分配

而對於 `String` 型別來說，為了要能夠支援**可變性** (_改變文字長度大小_)，我們需要在堆積 ( Heap ) 上分配一塊編譯時未知大小的記憶體來儲存這樣的內容：

-   記憶體分配器必須在執行時請求記憶體
-   我們不再需要這個 `String` 時，我們需要以某種方法將此記憶體還給分配器

第一部分，當呼叫 `String::from` ，他會請求分配一塊它需要的記憶體，這在其他程式語言都一樣。

第二部分，在擁有**垃圾回收機制(garbage collector, GC)** 的語言中，GC 會追蹤並清理不再使用的記憶體。沒有 GC 的話，就必須自己去識別哪些記憶體不再使用，並且釋放它們。如果忘記釋放會造成記憶體的浪費，太早釋放則會拿到無效的變數，如果釋放了兩次，就會造成程式錯誤。

Rust 的方法是，**當記憶體在擁有它的變數離開作用域時就會自動釋放**：

```rust
fn main() {
    {
        let s = String::from("hello"); // s 在此開始視為有效

        // 使用 s
    }                                  // 此作用域結束
                                       // s 不再有效
}
```

當 `s` 離開作用域，`String` 所需要的記憶體釋放回分配器。當離開作用域，Rust 會幫我們呼叫一個特殊函式 `drop` 來釋放記憶體。

### 變數與資料互動的方式：移動（Move）

```rust
fn main() {
    let x = 5;
    let y = x;
}
```

一般來說，`x` 取得數值 5，然後 copy 一份給 `y`。

但在 `String` 的版本，就不只是拷貝那麼簡單

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;

    println!("{}, world!", s1);
}
```

這就要先了解 `String` 的架構，一個 `String` 由三個部分組成：
1. 指向儲存字串內容記憶體的指標
2. 它的長度：是 `String` 內容在記憶體以位元組為單位所佔用的大小
3. 它的容量：是 `String` 從分配器以位元組為單位取得的總記憶體量

![rust-string.png](/images/Rust-ownership/rust-string.png)

所以將 `s1` 賦值給 `s2`，`String` 的資料會被拷貝，不過這裡指的是拷貝堆疊上的**指標**、長度和容量。

![rust-same-string.png](/images/Rust-ownership/rust-same-string.png)

> 如果 Rust 直接拷貝堆積的資料，`s2 = s1` 的動作花費會變得非常昂貴，當堆積上的資料非常龐大時，是十分影響效能的。

先前有提到當變數離開作用域時，Rust 會自動呼叫 `drop` 函式來清理堆積上的資料。而當 `s2` 與 `s1` 離開作用域時，它們都會嘗試釋放相同的記憶體，這被稱為 **雙重釋放** ( double free )，釋放記憶體兩次可能會導致記憶體損壞，進而造成安全漏洞。

所以為了保障記憶體安全，`let s2 = s1;` 後 `s1` 就不再有效，所以在 `s2` 建立後再使用 `s1` 就會無法執行：

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

Rust 會跳出錯誤防止你執行：

```bash
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:28
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 |
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0382`.
error: could not compile `ownership` due to previous error
```

### 變數與資料互動的方式：克隆（Clone）

如果需要深拷貝 ( deep copy )的話，使用 `clone`：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();

    println!("s1 = {}, s2 = {}", s1, s2);
}
```

這樣 `s1` 與 `s2` 都能使用。

### 只在堆疊上的資料：拷貝（Copy）

```rust
fn main() {
    let x = 5;
    let y = x;

    println!("x = {}, y = {}", x, y);
}
```

>Q: 那為什麼上面的程式碼會成立？沒有呼叫 `clone`，但 `x` 卻仍是有效的，沒有移動到 `y`。
>
>A: 因為像整數這樣的型別在編譯時是已知大小，所以只會存在在堆疊上。

Rust 有個特別的標記叫做 `Copy` 特徵（trait）可以用在標記像整數這樣存在堆疊上的型別。如果一個型別有實作 ( implement ) `Drop` 特徵的話，Rust 不會允許我們讓此型別擁有 `Copy` 特徵。

哪些型別有實作 `Copy` 特徵呢？基本原則是任何簡單地純量數值都可以實作 `Copy`

-   所有整數型別像是 `u32`。
-   布林型別 `bool`，它只有數值 `true` 與 `false`。
-   所有浮點數型別像是 `f64`。
-   字元型別 `char`。
-   元組，不過包含的型別也都要有實作 `Copy` 才行。比如 `(i32, i32)` 就有實作 `Copy`，但 `(i32, String)` 則無。

### 所有權與函式

傳遞數值給函式的方式和賦值給變數是類似的。

```rust
fn main() {
    let s = String::from("hello");  // s 進入作用域
    takes_ownership(s);             // s 的值進入函式
                                    // 所以 s 也在此無效
    let x = 5; // x 進入作用域 makes_copy(x); 
			   // x 本該移動進函式裡 // 但 i32 有 Copy，所以 x 可繼續使用
}
fn takes_ownership(some_string: String) { // some_string 進入作用域 
	println!("{}", some_string); 
} // some_string 在此離開作用域並呼叫 `drop` 
  // 佔用的記憶體被釋放
  
fn makes_copy(some_integer: i32) { // some_integer 進入作用域 
	println!("{}", some_integer); 
} // some_integer 在此離開作用域，沒有任何動作發生
```

如果呼叫 `takes_ownership` 後使用 `s`，Rust 會拋出編譯時期錯誤。

### 回傳值與作用域

變數的所有權每次都會遵照相同的模式，**只要賦值給其他變數就會移動**。當有堆積的變數離開作用域，該值就會被 `drop` 清除，除非資料的所有權被轉移到其他變數。

使用以下方法來回傳參數的所有值：

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1); // s1 移入 calculate_length
                                          // 將所有權透過回傳給 s2

    println!("'{}' 的長度為 {}。", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() 回傳 String 的長度

    (s, length)
}
```

以上是正確的做法，但如果要重複使用這個值，每一次都要傳進傳出就很麻煩。所以 Rust 還有提供一個在不移轉所有權的情況下使用數值，稱為 **引用 ( references )**。

## 引用與借用

**引用 ( references )** 就像是指向某個地址的指標，我們可以追蹤存取到該處儲存的資訊，而讓該地址被其他變數所擁有，與指標不同的是，引用保證所指向的特定型別的數值一定是有效的。

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("'{}' 的長度為 {}。", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

`&s1` 語法讓我們可以建立一個指向 `s1` 數值的引用，但不會擁有它。也因為它沒有所有權，它所指向的資料在引用不再使用後並**不會被丟棄**。

建立引用這樣的動作叫做**借用（borrowing）**。就像現實世界一樣，如果有人擁有一個東西，他可以借用給你。當你使用完後，你就還給他，你並不擁有它。

以下程式碼能不能執行？

```rust
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world");
}
```

答案是不行，因為它只是**借用**，所以不能改變引用的值。

### 可變引用

如果要讓上方的程式碼，改變引用的值：

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

- 先將 `s` 加入 `mut` 讓他能被改變
- `change` 函式的地方建立了一個可變引用 `&mut s`
- `change` 函式的新簽章為 `some_string: &mut String` 來接收這個可變引用

可變引用有一個大限制：**對相同的變數可變引用只能有一個**。如果嘗試建立兩個可變引用就會失敗：

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
}
```

錯誤資訊：

```bash
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
5 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
6 |
7 |     println!("{}, {}", r1, r2);
  |                        -- first borrow later used here

For more information about this error, try `rustc --explain E0499`.
error: could not compile `ownership` due to previous error

```

這項限制的好處是 Rust 可以在編譯時期就防止**資料競爭 ( data races )**。它會由以下三種行為引發：

-   同時有兩個以上的指標存取同個資料。
-   至少有一個指標在寫入資料。
-   沒有針對資料的同步存取機制。

資料競爭會造成**未定義行為 ( undefined behavior )**，而且在執行時你通常是很難診斷並修正的，而 Rust 能阻止這樣的問題，它不會讓有資料競爭的程式碼編譯。

簡單來說，不要同時擁有同一個引用就可執行：

```rust
fn main() {
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    } // r1 離開作用域，所以建立新的引用也不會有問題

    let r2 = &mut s;
}
```

可變引用和不可變引用，不能同時使用：

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // 沒問題
    let r2 = &s; // 沒問題
    let r3 = &mut s; // 有問題！

    println!("{}, {}, and {}", r1, r2, r3);
}
```

這是因為，不可變引用的使用者不希望有人改變了值，並造成錯誤。不過多個不可變引用是沒問題的，因為大家都不能變更值。

引用的作用域始於它被宣告的地方，一直到它最後一次引用被使用為止：

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // 沒問題
    let r2 = &s; // 沒問題
    println!("{} and {}", r1, r2);
    // 變數 r1 和 r2 將不再使用
    
    let r3 = &mut s; // 沒問題
    println!("{}", r3);
}
```

### 迷途引用 ( Dangling pointer )

有指標的程式語言，就會不小心產生 **迷途指標 ( dangling pointer )**。當資源已經被釋放但指標卻還留著，這樣的指標指向的地方很可能就已經被別人所有了。在 Rust 中編譯器會保證引用絕不會是迷途引用。

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String { // 回傳 String 的迷途引用

    let s = String::from("hello"); // s 是個新 String

    &s // 我們回傳 String s 的引用
} // s 在此會離開作用域並釋放，它的記憶體就不見了。
  // 危險！

```

`s` 是在 `dangle` 裡面產生的，當 `dangle` 結束 `s` 會被釋放。如果嘗試回傳 `s`，這個引用會指向一個無效的 `String`，所以 Rust 不會讓它發生。

讓他回傳的值不是引用就可以了，這邊直接回傳 `String`就好：

```rust
fn main() {
    let string = no_dangle();
}

fn no_dangle() -> String {
    let s = String::from("hello");

    s
}
```

## 切片 (Slice)

**切片 (Slice)** 可以引用一串集合的元素列，並非引用整個集合。切片也是一種引用，所以它沒有所有權。

寫一個函式接收一串用空格分開單字的字串，如：`Hello world`、`Good job`，並回傳第一個找到的單字；如果沒有找到任何空格，就代表整個字串就是一個單字，並回傳整個字串。

```rust
fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return i;
        }
    }

    s.len()
}

fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s); // word 取得數值 5

    s.clear(); // 這會清空 String，這就等於 ""

    // word 仍然是數值 5 ，但是我們已經沒有相等意義的字串了
    // 擁有 5 的變數 word 現在完全沒意義！
}
```

- `let bytes = s.as_bytes();`：將 `String` 轉換成一個位元組陣列
- `for (i, &item) in bytes.iter().enumerate()`：使用 `iter` 方法對位元建立一個疊代器 (iterator)
	- `iter()`：是一個回傳集合中的每個元素方法
	- `enumerate()`：回傳的元組中第一個是索引(`i`)，第二個是元素的引用(`&item`)
- `if item == b' ' {return i}` ：找到空格後回傳該位置，如果沒有就回傳整個字串長度

程式雖然可以成功編譯，可以看到 `s` 的內容與 `word` 是沒有直接關係的，所以當 `s` 改變後，直接使用 `word` 去獲得 `s` 的單字，就會造成錯誤，這也導致需要留意 `word` 是不是與 `s` 脫鉤，而這個麻煩可以使用 Rust 的 **字串切片(tring slice)**。

### 字串切片

```rust
fn main() {
    let s = String::from("hello world");

    let hello = &s[0..5];
    let world = &s[6..11];
}
```

> 基本上就是 Python 的 slice，只是用引用的方式

與其引用整個 `String`，透過 `[0..5]` 來引用了一部分的 `String`。

更改上方回傳字串的程式：

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

> 可以將 `fn first_word(s: &String) -> &String` 寫成 `fn first_word(s: &str) -> &str` 

現在編譯器會確保 `String` 的引用是有效的，所以當使用以下程式碼進行編譯，就會直接跳出錯誤：

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s); // word 取得數值 5

    s.clear(); // 錯誤
	println!("第一個單字為：{}", word);
}
```

```bash
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
  --> src/main.rs:18:5
   |
16 |     let word = first_word(&s);
   |                           -- immutable borrow occurs here
17 |
18 |     s.clear(); // 錯誤！
   |     ^^^^^^^^^ mutable borrow occurs here
19 |
20 |     println!("第一個單字為：{}", word);
   |                                ---- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
error: could not compile `ownership` due to previous error
```

出現錯誤是因為借用的規則：當呼叫 `clear` 會清除 `String`，這表示它必須是可變引用。在 `clear` 後呼叫 `println!` ，這時就會用到 `word` 的引用。Rust **不允許同時**存在 `clear` 的可變引用與 `word` 的不可變引用，所以會編譯失敗。這就能讓這些錯誤在編譯期間就被發現，進而修改。

## 結論

在高中時期有使用 Unity 製作過遊戲，當時所使用的就是 C# 程式語言，對於 GC 這個機制是不陌生的，而到了大學轉為寫網頁時，JavaScript 也有 GC 的機制來回收記憶體。第一次接觸 Rust 管理記憶體的方法，所有權的規則看似限制很多其實用起來很直覺，他可以預防很多問題，例如: 指標是空的、迷途指標等等。

---

## 引用

- [理解所有權](https://rust-lang.tw/book-tw/ch04-01-what-is-ownership.html)