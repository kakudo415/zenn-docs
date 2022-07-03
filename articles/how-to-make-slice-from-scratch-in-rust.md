---
title: "RustのSliceを自作する"
emoji: "🥩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust"]
published: false
---

# はじめに

話を簡単にするために、この記事で作る`MyVec`・`MySlice`はかなり限定的な機能を持ったものにしています。
定義は以下の通りです。

```rust
struct MyVec {
    ptr: *mut char,
    len: usize,
    cap: usize,
}

struct MySlice {
    ptr: *mut char,
    len: usize,
}
```

:::details サンプルデータを含むプログラム全体

```rust
use std::alloc;
use std::ops;

struct MyVec {
    ptr: *mut char,
    len: usize,
    cap: usize,
}

struct MySlice {
    ptr: *mut char,
    len: usize,
}

fn new_sample_myvec() -> MyVec {
    let layout = alloc::Layout::array::<char>(10).unwrap();
    let ptr = unsafe { alloc::alloc(layout) } as *mut char;
    for i in 0..10 {
        unsafe {
            ptr.add(i).write(char::from_u32(65 + i as u32).unwrap());
        }
    }
    MyVec {
        ptr: ptr,
        len: 0,
        cap: 10,
    }
}

fn main() {
    let myvec = new_sample_myvec(); // [A, B, C, D, E, F, G, H, I, J]
}
```

:::

`Vec<T>`は`Deref`トレイトを実装していて、`&Vec<T>`は自動で`&[T]`に変換されます。
この処理を`MyVec`・`MySlice`に素直に実装してみると以下のようになります。

```rust
impl ops::Deref for MyVec {
    type Target = MySlice;
    
    fn deref(&self) -> &MySlice {
        let temporary_myslice = MySlice {
            ptr: self.ptr,
            len: self.len,
        };
        &temporary_myslice
    }
}
```

しかし、MySliceのインスタンスは`deref()`内で作られたものであるため、上記のように参照を返そうとすると、以下のように怒られてしまいます。

```
error[E0515]: cannot return reference to local variable `temporary_myslice`
```
