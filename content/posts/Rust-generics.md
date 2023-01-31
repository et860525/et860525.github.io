---
title: "Rust: 泛型( Generics )"
date: 2023-01-31
draft: false
author: "Chen Yu Fan"
tags: ["Rust"]
---

泛型 ( generics )，實際型別或屬性的抽象表示。舉例來說，`String` 和 `i32` 這兩個不同型別的資料都可以被存到 [Vec 結構體](https://doc.rust-lang.org/std/vec/struct.Vec.html)建立的實例中，不需要針對型別來做分別，只要使用 `Vec<String>` 或 `Vec<i32>`，這是因為 `Vec` 結構體使用了泛型。

泛型就是 **參數多型 ( parametric polymorphism )**，在定義型別或函數的時候不去明確指定具體的型別，而是以參數的形式來傳入型別，這可以讓程式設計更為彈性。

以下先來看泛型在各個地方中如何定義。

<!--more-->

## 函式中定義

用一個找出陣列最大元素值的程式來實作泛型，首先，先來看它原本的樣子：

```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];
 
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];
 
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}
 
fn main() {
    let int_list = vec![34, 50, 25, 100, 65];
 
    let result = largest_i32(&int_list);
    println!("The largest integer number is {}", result);
    
    let char_list = vec!['y', 'm', 'a', 'q'];
    
    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
}
```

兩個 `largest_i32`、`largest_char`，可以分別從 32位元的有號整數陣列切片、字元陣列切片中找出最大的元素並回傳出來。基本上函式裡面的程式碼都是相同的，只是因為我們要處理不同型別的資料所以寫了三次，如果要連 `i8`、`i16`、`i64`、`u8`、`u16`、`u32`、`u64`、`f32` 等型別的陣列切片都寫，那不就要再多寫8次，所以 Rust 提供泛型來解決這個問題，以下是改為泛型示範：

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];
    
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}
 
fn main() {
    let int_list = vec![34, 50, 25, 100, 65];
    
    let result = largest(&int_list);
    println!("The largest integer number is {}", result);
    
    let char_list = vec!['y', 'm', 'a', 'q'];
    
    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

在呼叫 `largest<T>` 函式時，編譯器會在編譯階段時自動判斷泛型的第一個參數型別，來決定 `T` 會是什麼型別。如果不想要讓編譯器自己判定，那就在呼叫函式時直接指定泛型的型別：

```rust
fn largest<T>(list: &[T]) -> T {
    // 略...
}
 
fn main() {
    let int_list = vec![34, 50, 25, 100, 65];
    
    let result = largest::<i32>(&int_list);
    println!("The largest integer number is {}", result);
    
    let char_list = vec!['y', 'm', 'a', 'q'];
    
    let result = largest::<char>(&char_list);
    println!("The largest char is {}", result);
}
```

不管如何，上述程式碼執行都會出現錯誤。這是因為要將值進行大於小於的判定之類的邏輯判斷，該值就必須要有 `PartialOrd` 的**特徵 ( trait )** 。綜上所知，泛型的 `T` 可以是任何型別，但它不一定會是 `PartialOrd` 特徵，所以程式碼才會編譯失敗。為了要讓編譯器確定 `T` 要有這個 `PartialOrd` 特徵，我們必須事先明確定義，就像定義一般函式參數的型別：

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T { // 加上 PartialOrd 特徵
    let mut largest = list[0];
    
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}
 
fn main() {
    // 略...
}
```

再編譯一次還是會有錯誤，這是因為第 2行有將陣列的元素指派給 `largest` 變數，這種 **把變數指派給另一個變數**就表示這個型別有 `Copy` 的特徵。所以還要再讓 `T` 知道該型別還會有 `Copy` 特徵：

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T { // 再加上 Copy 特徵
    let mut largest = list[0];
    
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}
 
fn main() {
    // 略...
}
```

這樣程式就可以執行了：

```bash
❯ cargo run
   Compiling hello_cargo v0.1.0 (file:///projects//hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 0.23s
     Running `target/debug/hello_cargo`
The largest integer number is 100
The largest char is y
```

## 結構體中定義

在結構體的名稱右邊加上 `<>` 語法來定義泛型：

```rust
struct Point<T> {
	x: T,
	y: T
}

fn main() {
	let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

編譯器在編譯期間也會自動判斷泛型第一個接觸的值，來決定型別。因此 `integer` 的泛型為 `i32`；`float` 的泛型為 `f64`。

另一個例子：

```rust
struct Point<T> {
    x: T,
    y: T
}
 
fn main() {
    let wont_work = Point { x: 5, y: 4.0 };
}
```

以上程式就會發生錯誤，因為泛型第一個接觸的值是 5 也就是 `i32` 型別，此時 `Point` 的 `x` 與 `y` 的值都一定要是 `i32`，自然也就不能存取 `4.0` 這個 `f64` 的型別。

要解決上面的問題，可以使用兩個泛型參數：

```rust
struct Point<T, U> {
    x: T,
    y: U
}
 
fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

## 枚舉中定義

像是 [Option 枚舉](https://doc.rust-lang.org/stable/std/option/enum.Option.html) 與 [Result 枚舉](https://doc.rust-lang.org/std/result/enum.Result.html)：

```rust
enum Option<T> {
    Some(T),
    None
}

enum Result<T, E> {
    Ok(T),
    Err(E)
}
```

## 方法中定義

`impl` 關鍵字右邊也可以加上 `<>` 來定義泛型要使用的參數：

```rust
struct Point<T> {
    x: T,
    y: T,
}
 
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
 
fn main() {
    let p = Point { x: 5, y: 10 };
 
    println!("p.x = {}", p.x());
}
```

`impl` 也可以只針對的特定的型別，來實作關聯函式和方法：

```rust
struct Point<T> {
    x: T,
    y: T,
}
 
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
 
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
 
impl Point<i32> {
    fn distance_from_origin(&self) -> f64 {
        ((self.x.pow(2) + self.y.pow(2)) as f64).sqrt()
    }
}
 
fn main() {
    let p = Point { x: 3.0, y: 4.0 };
 
    println!("distance = {}", p.distance_from_origin());
 
    let p = Point { x: 5, y: 12 };
 
    println!("distance = {}", p.distance_from_origin());
}
```

## 使用泛型的程式碼效能

> Rust 的泛型不會有任何額外的運算效能的耗損。

Rust 在編譯時對使用泛型的程式碼進行**單態化 ( monomorphization )** 。單態化能讓泛型轉換成特定程式碼的過程，並在編譯時填入實際型別。簡單來說，它會根據填入的實際型別，自動產生相應的程式碼。

以下示範標準函式庫的泛型枚舉 `Option<T>` 是如何做到的：

```rust
let integer = Some(5);
let float = Some(5.0);
```

Rust 在編譯上面的程式碼時，就會進行單態化。上面的型態分別是 `i32` 與 `f64`，而編譯器會自動產生 `Option_i32` 和 `Option_f64` 的結構體 ( 編譯器實際的名稱與這邊的不同 )：

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

因此，使用泛型的時候，程式在執行階段完全不需要使用額外的運算資源去進行型別的檢查，因為這些工作都在編譯期間自動完成了。

## 結論

在 TypeScript 裡也有泛型的機制，所以學習起來並不那麼吃力。泛型就是為了解決這種不同型別但是程式碼重複的情況，加上下個章節會提到的**特徵 ( trait )** 來限定哪種型別能使用。

---

## 引用

- [泛型資料型別](https://rust-lang.tw/book-tw/ch10-01-syntax.html)