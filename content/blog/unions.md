+++
title = "Designing an efficient memory layout in Rust with unions, or, an overlong guide in avoiding dynamic dispatch"
date = 2024-11-27
+++

## Introduction

This is the first blog post in a series of how to build a CLI spreadsheet program, mostly because I'm too tired of all other spreadsheets' deficiencies. In this blog post, I will be designing the memory layout of all spreadsheet cells value, so we should start with the question: What does a spreadsheet cell contain?
* A number? Perhaps!
* A string of characters? Perhaps!
* A formula, which is itself a domain-specific-language? Perhaps!

However, that is not just it. I am not aware if that is the case in Excel, but in Google Docs, a cell can get its value overridden by a matrix displayed on another cell that covers it. Matrices and iterators will be the core design of this spreadsheet engine, but that is for another blog post. However, that means that a value is either one of the listed before, or an iterator that yields these values.

## A first attempt: dynamic dispatch

A naive Rust programmer, probably someone who's just read the book and is likely to come from an OOP language, will raise their hands and shout: *"enums, teacher, enums!"*, and that would not be untrue. However, for educational purposes, we will start modeling this with dynamic dispatch; because as we will see in just a moment, its memory layout is very efficient (always two words/`usize`s). For starters, let's define a trait that models the behavior for any cell value:
```rust
trait CellValue: Display + Any {}
```

I have decided not to include any methods, and instead require that a) it implements `Display` for printing on the screen, and b) It implements `Any` so that we can "downcast" the value out of dynamic dispatch if necessary. Downcasting is a practice in which you can save an "ID" of a type inside its dynamic function table (what the keyword *dyn* generates); and if you statically assert that its ID is of a known type, it is safe to convert it back to its non-dynamic type.

Thus, we can now define each individual type:
```rust
type DynCellValue = Box<dyn CellValue>;
struct Num(f64);
struct Str(String);
struct Formula; // To-do
struct Iter(Box<dyn Iterator<Item = DynCellValue>);

// Display & CellValue impls left as an exercise for the reader.
```

The most important conclusion we can draw is that `Iter` has two dynamic dispatch indirections, for itself (since we want to allow arbitrary matrices), and for its values, as they themselves could be anything. Before discussing why this is a bad approach, let's take a look at `DynCellValue`'s layout:
```txt
[ 64bits               | 64bits                ]
|______________________|_______________________|
 Pointer to the heap,   Pointer to a VTable,
 where the data lives.  where we can find the
                        pointers to the
                        functions it implements.
```

However, the `Num` case is as big as a pointer, so perhaps we could include it inline, instead of allocating it! If only we had a way of distinguishing between those two cases^[1]... Not only that, but maybe we could use a `ThinVec` to store the `String`' contents, which would store all of its data (other than the pointer to the heap) inside its allocation (surprise! a `String` is just a `Vec<u8>`). That would only leave the `Formula` as a variant possibly bigger than a word, but I plan on it being inside a reference-counted allocation anyway (an `Rc` container), so in the future, it'll be just a pointer. That means that all allocations of `Box<dyn CellValue>` would just be... slow and needless indirections. There is a point why I started the article with `dyn`, even though it's the slowest and least idiomatic: it will always be two words, regardless of what we throw at it. From now on, our objective is to keep that while removing all of `dyn`'s unneecessary indirections.

## Enum dispatch
This is what you should always try before resorting to dynamic dispatch. `enum_dispatch` is a useful crate in situations which you know all the types of a `dyn Trait`, as it automagically does the following:
```rust
enum CellValue {
    Num(f64),
    Str(ThinVec<u8>),
    Formula(Rc<Formula>),
    Iter(Box<dyn Iterator<Item = CellValue>>),
}

impl Display for CellValue {
    fn fmt(&self, f: ...) -> ... {
        match self {
            Self::Num(x) => x.fmt(f),
            Self::Str(s) => s.fmt(f),
            Self::Formula(o) => o.fmt(f),
            Self::Iter(i) => todo!(),
        }
    }
}
```

The `todo!()` there is beacuse `Iter` has spooky action at a distance (meaning, it affects more cells than itself); so in the future, we'll engineer a way to allow it to do its thing, but, right now, that is not a priority. Layout:
```txt
[ 64bits               | 64bits                ]
|______________________|_______________________|
 Tag                    An f64, a ThinVec<u8>
                        or an Rc<Formula>.
```

As you can see, we managed to keep the enum at two words (one for the value, one for the tag that tells which variant is inside), and by removing the heap allocations & pointers and the opaque function calls, it now is orders of magnitude faster. All the `Display::fmt()` functions are now static, and in the `enum`'s, we dispatch it to the implementation of the inner values using the tag (what `match` is doing). However, I lied before, our number is not a single word, it's going to be two (just like an u128), and for now we will call it `Long` to piss off all C programmers (IYKYK). That breaks our previous goal of keeping it at two words, since enums are as big as their bigger variant plus the tag, so it'd be three words:
```txt
[ 64bits              | 64bits                | 64bits                ]
|_____________________|_______________________|_______________________|
 Tag                    Half a Long, or a full  The other half of a                                    
                        ThinVec<u8> or          `Long`, or nothing
                        Rc<Formula>.            otherwise.
```

That is a whole wasted word when the value is not a `Long`, we can do better than that! However, we will have to rely on a giant hack: our `Long` does not use its entire bit pattern. Otherwise, if it being just two words long was really necessary, or there was a very big variant in contrast to very small ones, the best approach would be to box said variant, which is still way more performant than `dyn` (so, as you can see, avoid it if you can).

## The decimal number type, ft tagged pointers.
That is the correct name for our `Long`, `Decimal`. It is a number like a floating-point one, but much more precise and suitable for financial computations. Its layout is as follows:
```txt
[ 15bits | 113 bits                                             ]
|________|______________________________________________________|
 Unused.   The actual decimal number, irrelevant.
```

Although these first bits are unused, they're always zero, and the moment they're not, we may hit UB in the internal implementations of the number.

If you've ever played with tagged pointers, perhaps you already know what we're getting into here. For any pointer, if the pointee has an alignment bigger than one, we can store as much data in the lower part of the pointer as its alignment. That is because in Rust, all reads are aligned, so data never exists outside its alignment. Note that this data must be removed before the pointee is read (for obvious reasons). From now on, all our cell values will be of alignment of two, so:
```txt
[ 2bits | 62 bits              ... ]
|_______|__________________________|
 Always   The rest of the pointer.
 zeroed.
 ```

Thus, we can exploit these common unused bits to store the `enum`'s tag, and this is what the `tagged_pointer` crate will do for us. Explaining tagged pointers beyond the conceptual points is out of scope for this post, so I recommend you read `tagged_pointer`'s documentation and source if you're interested.

## Now with unions, also known as C's untagged enums or Friedrich Transmute.

Unions are the backing data for enums, these allow you to define a space of data which may be used by a set of types; but only one at a time, and without taking note of which one it is. The Friederich Transmute pun comes from the fact that these effectively allow you to do the same as `std::mem::transmute`, since by accessing using a different type than what was written, you're reinterpreting the bytes of a type as a different one.

For starters, let's copy our previous `enum`:
```rust
union CellValue {
    num: Decimal,
    str: ThinVec<u8>,
    formula: Rc<Formula>,
    iter: Box<dyn Iterator<Item = CellValue>>,
}
```

The first thing we'll get is a screaming message from rustc telling is to wrap everything that's not `Copy` in a `ManuallyDrop<T>`. As it turns out, unions are one of the many reasons why destructors are just a suggestion in Rust, and that is what `ManuallyDrop<T>` does, remove the drop implementation from its inner. This is needed because there is no info for the compiler to codegen the drop of the union since it can't possibly know what's inside, so it nicely asks us not to ask it to do so. The next thing we'll do is add a field with a `TaggedPtr<Aligned, 2>` where `Aligned`'s align is two; so that we can use what's inside the union as a tagged pointer to get, set, and remove the tag. As it stands, we get the following:

```rust
union CellValue {
    tag: TaggedPtr<Aligned, 2>,
    num: Decimal,
    str: ManuallyDrop<ThinVec<u8>>,
    formula: ManuallyDrop<Rc<Formula>>,
    iter: ManuallyDrop<Box<dyn Iterator<Item = CellValue>>>,
}
```

### Manually implementing iter's `dyn`.
Before continuing, there's something we should do, desugar `Box<dyn ...>`. This allows us to remove a pointer of indirection: the whole vtable. It is true that `Iterator` has like eighty methods we will want to use, but these all have default implementations which we will not override, so we won't need a pointer to a vtable of functions, only to one function (I bet you know where this is going). The only required function is `Iterator::next()`, so that is the only one we will keep as an opaque one, and we will statically generate the rest. First, let's create a struct with the data:
```rust
struct DynIter {
    data: TaggedPtr<Aligned, 2>,
    next: NonNull<()>, // All function pointers are guaranteed not to be null
}
```

Then let's tell the compiler to codegen the rest of the iterator's functions with our opaque `next()` for us:

```rust
impl Iterator for DynIter {
    type Item = CellValue;

    fn next(&mut self) -> Option<Self::Item> {
        type NextFn = for<'a> fn(&'a mut Aligned) -> CellValue;
        unsafe { transmute::<_, NextFn>(self.next)(self.data.ptr().as_mut()) }
    }
}
```

Taking advantage of this, I will be refactoring the unions so as to limit when a value is or is not allowed to be an iterator (this will make our life easier in the coming posts):
```rust
const MASK_BITS: usize = 2;

union Value {
    tag: TaggedPtr<Aligned, MASK_BITS>,
    num: Decimal,
    str: ManuallyDrop<ThinVec<u8>>,
    formula: ManuallyDrop<Rc<Formula>>,
}

struct DynIter {
    data: TaggedPtr<Aligned, MASK_BITS>,
    next: NonNull<()>,
}

union CellValue {
    tag: TaggedPtr<Aligned, MASK_BITS>,
    value: ManuallyDrop<Value>,
    iter: ManuallyDrop<DynIter>,
}
```

### Messing with TaggedPtr

Now we have to choose the bitmasks for the tags of each value. The `0b00` and `0b01` tags will be the fastest ones, so I will be giving them to `num` and `iter`, being the no-op one (`0b00`) for `num` because I'd rather not mess with it:
```rust
const DEC_MASK: usize = 0b00;
const ITER_MASK: usize = 0b01;
const STR_MASK: usize = 0b10;
const FORMULA_MASK: usize = 0b11;
```

Because `TaggedPtr::new(ptr, tag)`'s is the way you construct a new one, we will have to figure out a way to have the `ptr` it gets to be the same as the low bits of our CellValue. The easiest way out of here would just be to use arguably the most unsafe function in all Rust, the chaotic sibling of Friedrich Transmute: `std::mem::transmute_copy`. This allows us to copy the CellValue's low 64 bits into `TaggedPtr::new()`'s first argument, which is likely to get optimized away after inlining:
```rust
impl Value {
    unsafe fn tag(mut self, tag: usize) -> Self {
        self.tag = TaggedPtr::new(transmute_copy(&self), tag);
        self
    }

    fn dec(dec: Dec) -> Self { Self { num: dec } }

    fn str(str: ThinVec<u8>) -> Self {
        unsafe { Self { str: ManuallyDrop::new(str) }.tag(STR_MASK) }
    }

    fn formula(f: Rc<Aligned>) -> Self {
        unsafe { Self { formula: ManuallyDrop::new(f) }.tag(FORMULA_MASK) }
    }
}
```
The last thing I will show on this blog post is how to get back a value from our union, which can be done by checking whether the tag matches the value you want back:

```rust
impl CellValue {
    pub fn get_value(&mut self) -> Option<&mut Value> {
        if self.downcast_iter().is_none() {
            Some(unsafe { transmute(&mut self.value) })
        } else {
            None
        }
    }

    pub fn get_iter(&mut self) -> Option<&mut DynIter> {
        if unsafe { self.tag }.tag() == ITER_MASK {
            Some(unsafe { transmute(&mut self.iter) })
        } else {
            None
        }
    }
}

impl Value {
    fn get_num(&mut self) -> Option<&mut Dec> {
        if unsafe { self.tag }.tag() == NUM_MASK {
            Some(unsafe { transmute(&mut *self) })
        } else {
            None
        }
    }

    // the rest are almost the same, so I leave as an
    // exercise to the reader creating a macro for them.
}
```

There's only one little thing that we are forgetting, the drop code. Because I left as an exercise for you creating a macro for accessing each field, I will do the same with the drop code that uses it (this is long enough, I'm tired).

## Conclusion
Unions are not just an archaic tool from the forgotten era of Daniel Ritchie, they are still a very useful tool which can yield amazing results in the right han**Segmentation fault, (core dumped)**.

---
[1]: Note that this is what Niko Matsakis proposed with its [dyn* blog post](https://smallcultfollowing.com/babysteps/blog/2022/03/29/dyn-can-we-make-dyn-sized/), and we will be exploiting this later.
