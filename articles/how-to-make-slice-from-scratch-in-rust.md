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

:::details サンプルデータ生成を含むプログラム全体

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

本家`Vec<T>`は`Deref`トレイトを実装していて、`&Vec<T>`は自動で`&[T]`に変換されます。
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

# 本家の挙動を追う

解決策を知るため、本家の実装を追ってみます。
まずは、`Vec<T>`に対する`Deref`トレイトの実装です。

`alloc/vec/mod.rs`を読むと、`slice::from_raw_parts()`が呼ばれているようです。

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T, A: Allocator> ops::Deref for Vec<T, A> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        unsafe { slice::from_raw_parts(self.as_ptr(), self.len) }
    }
}
```

`core/slice/raw.rs`

```rust
#[inline]
#[stable(feature = "rust1", since = "1.0.0")]
#[rustc_const_unstable(feature = "const_slice_from_raw_parts", issue = "67456")]
#[must_use]
pub const unsafe fn from_raw_parts<'a, T>(data: *const T, len: usize) -> &'a [T] {
    // SAFETY: the caller must uphold the safety contract for `from_raw_parts`.
    unsafe {
        assert_unsafe_precondition!(
            is_aligned_and_not_null(data)
                && crate::mem::size_of::<T>().saturating_mul(len) <= isize::MAX as usize
        );
        &*ptr::slice_from_raw_parts(data, len)
    }
}
```

`&*`なるものがついていますが、とりあえずさらに`ptr::slice_from_raw_parts()`を確認してみます。

`core/ptr/mod.rs`

```rust
#[inline]
#[stable(feature = "slice_from_raw_parts", since = "1.42.0")]
#[rustc_const_unstable(feature = "const_slice_from_raw_parts", issue = "67456")]
pub const fn slice_from_raw_parts<T>(data: *const T, len: usize) -> *const [T] {
    from_raw_parts(data.cast(), len)
}
```

`core/ptr/metadata.rs`

```rust
#[unstable(feature = "ptr_metadata", issue = "81513")]
#[rustc_const_unstable(feature = "ptr_metadata", issue = "81513")]
#[inline]
pub const fn from_raw_parts<T: ?Sized>(
    data_address: *const (),
    metadata: <T as Pointee>::Metadata,
) -> *const T {
    // SAFETY: Accessing the value from the `PtrRepr` union is safe since *const T
    // and PtrComponents<T> have the same memory layouts. Only std can make this
    // guarantee.
    unsafe { PtrRepr { components: PtrComponents { data_address, metadata } }.const_ptr }
}
```
