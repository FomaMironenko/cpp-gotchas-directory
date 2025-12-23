# C++ Advanced Primer

// TODO



---

## 1. std::vector

`std::vector<T>` is dynamic array, which means the number of its elements (**size**) can change during lifetime. The *size*, however, does not equal the actual amount of allcated space (**capacity**). *Capacity* is the number of elements of type `T` that can fit into the currently allocated memory.

Vector *size* can never exceed *capacity*. If at any point of execution *capacity* of a vector equals its *size* and user attempts to insert a new element, a **reallocation** takes place. During the *reallocation*:
- a new memory chunk, bigger than the current capacity, is allocated,
- all the elements are transferred (usually, but not always, by move sematics) to the new location,
- the internal vector data pointer ids overwritten with the address of the new location. 

In accordance to what was just said, vector implementation must store and manipulate the followinng members: 1. pointer to the allocated chunk, 2. vector size, 3.vectror capacity.

Alternatively, the implementation might store: 1. pointer to the allocated chunk, 2. pointer to the end of currently stored elements, 3. pointer to the end of the allocated chunk.

Both implementations store exactly three machine words, and thus it is the size of a vector object (unless a custom allocator is specified).

#### `std::vector` operational complexity
For a vector of size `N` and capacity `C` the complexity is as follows:

- **Insert an element at position `I <= N`: if `N < C`, then `O(N-I)`, else `O(N)`**
Specifically, the implementation would need to shift `N - I` elements towards the end (`O(N - I)`) and in case `N == C` allocate a new chunk and transefer **all** elements there (`O(N)`).

- **Erase an element at position `I < N`: `O(N-I)`**
Arguments are the same as in the case of insertion, except that now there's no need for reallocation. Only action necessary is to shift `N - I` tail elements towards the begin.

- **Insert an element at the end (`push_back`): amortized `O(1)`**
There's no need to shift any elements. If `N < C` (most of the cases), there is enough capacity for yet another element, and adding new item is `O(1)`. In rare case when `N == C`, a reallocation takes place and the `push_back` would instantly cost `O(N)`. But the [amortized complexity](https://en.wikipedia.org/wiki/Amortized_analysis) when multiple `push_backs` are executed is still `O(1)`.

- **Erase the `back` element: `O(1)`**
There's no need for reallocation, and the complexity is always constant.


### 1.1 Iterators invalidation

The most common source of bugs in vector is iterators and pointers (to vector elements) invalidation. In common implementations vector iterator is nothing but pointer to a location within the previously allocated chunk. Thus, unlike in some other data structures, vector iterators and pointers are invalidated simultaneously.

Every reallocation or elements shift invalidate (some) iterators, but these are different types of UB. Elements shift would cause iterators to shifted elements point to unexpected elemnts within the allocated chunk and possibly within vector size. Reallocation would make every iterator and pointer dandgling, with all ensuing consequences.

The scenarios of iterator invalidation are:

- **`reserve` and `shrink_to_fit`**:
(might) cause a reallocation, thus invalidate all the iterators and pointers.

- **`clear`**:
invalidates all iterators and pointers. Actually, most implementations would only reset the vector size, keeping elements unaffected and addressable by the iterators and pointers. But it is still UB to access any previously stored elements.

- **`resize` to new size `M`**:
if `M < N`, invalidates iterators and pointers to elements at positions `M <= I < N`. If `N <= M <= C`, no iterators are invalidated. Finally, if `M > C`, all iterators and pointers are invalidated.

- **Insert an element at position `I <= N` (`push_back` in particular)**:
invalidates iterators to positions `J >= I` due to shift, and if `N == C` invalidates all iterators due to reallocation. Since reallocation is transparent to user, you must be particularly careful and make sure there was no reallocation before accessing iterators to positions `J < I`.

```C++
std::vector<int> v = {1, 2, 3};
v.reserve(4);
auto begin_it = v.begin();
v.push_back(4); // guaranteed no reallocation, begin_it IS VALID
v.push_back(5); // reallocation is possible,   begin_it IS NO MORE VALID
```

- **Erase an element at position `I < N`**:
invalidates iterators to positions `J >= I` due to shift.


### 1.2 Transfering elements on reallocation

During reallocation all existing elements need to be transferred from the previous memory chunk to the newly allocated one. The main question is how to do this efficiently and exception-safely. Reallocations must provide [strong exception guarantee](https://en.cppreference.com/w/cpp/language/exceptions.html#Exception_safety), meaning that if during reallocation an exception is thrown, the operation will have no effect and the vector will preserve initial state.

This turns out to be quite tricky if you don't want to copy each element, because move semantics mutates source elements and leaves them in unspecified state. If you've already moved half of the elements to the new destination, and the next call to move constructor throws an error, there's no way to restore the moved elements. You could try to move them back from destination to source. But if move constructor throws again, that will be fatal. The same does not apply to copy constructor, that does not modify source elements.

This leads to the following constraint: **if vector element type move constructor is not `noexcept`, elements are copied during reallocation, not moved**. Even if copy constructor is also not `noexcet`. The condition can be checked via [`std::is_nothrow_move_constructible`](https://en.cppreference.com/w/cpp/types/is_move_constructible.html) type trait. The implementation is somewhat like calling [`std::move_if_noexcept`](https://en.cppreference.com/w/cpp/utility/move_if_noexcept.html) on each element. If move constructor is not `noexcept` and copy constructor is not provided, objects are, nevertheless, moved, likewise `std::move_if_noexcept`.

```C++
// GOOD: will be moved
struct SafelyMovable {
    SafelyMovable(SafelyMovable&&) noexcept {  }
    SafelyMovable(const SafelyMovable&) noexcept {  }
};
static_assert(std::is_nothrow_move_constructible_v<SafelyMovable>, "Explicitly noexcept");

// BAD: will be copied
struct UnsafelyMovable {
    UnsafelyMovable(UnsafelyMovable&&) {  }
    SafelyMovable(const SafelyMovable&) {  }
};
static_assert(!std::is_nothrow_move_constructible_v<SafelyMovable>, "Implicitly not noexcept");

// "GOOD": will be unsafely moved
struct NotCopyable {
    NotCopyable(const NotCopyable&) = delete
    NotCopyable(NotCopyable&&) {  };
};
```

The move-constructor generated by compiler is safely movable if and only if all members are safely movable.
```C++
// GOOD: will be moved
struct ImplicitlySafelyMovable {
    SafelyMovable m_good;
};
static_assert(std::is_nothrow_move_constructible_v<ImplicitlySafelyMovable>, "Implicitly noexcept by members");

// BAD: will be copied
struct ImplicitlyUnsafelyMovable {
    UnsafelyMovable m_bad;
};
static_assert(!std::is_nothrow_move_constructible_v<ImplicitlyUnsafelyMovable>, "Implicitly not noexcept by members");
```

In user-defined classes copy-constructors are ususally either omitted or declared `default`. Good news is that even if defaulted move-constructor is not labeled `noexcept`, the compiler will make it `noexcept` if all members are safely movable.
```C++
// GOOD: will be moved
struct DefaultSafelyMovable {
    SafelyMovable m_good;
    DefaultSafelyMovable(DefaultSafelyMovable&&) = default;
    DefaultSafelyMovable(const DefaultSafelyMovable&) = default;
};
static_assert(std::is_nothrow_move_constructible_v<DefaultSafelyMovable>, "Implicitly noexcept by members");
```

Now let's move on to the positive scenario when it is possible to not copy all elements. In some cases moving each distinct element is not the most efficient solution. Assume you have a `std::vector<int>`. In this case moving means copying. Copying each distinct element would be much slower than calling `memcpy` for the whole range. In general, a type can be `memcpy`-ed if it satisfies [`std::is_trivially_copyable`](https://en.cppreference.com/w/cpp/types/is_trivially_copyable.html) type trait. Thus, **trivially copyable types can be transfered via `memcpy` during reallocation**. This is not guaranteed by the standard, but most implementations do so.

#### Conclusion
- if element type does not satisfy `std::is_nothrow_move_constructible`, elements are **copied** during reallocation;
- else if element type satisfies `std::is_trivially_copyable`, elements can be (will almost certainly be) **`memcpy`-ed**;
- else elements are **moved**.


### 1.3 Clean up vector memory

Allocated vector memory is automatically freed on vector destruction. During reallocation old data chunk is also freed. Note, however, that memory consumption is temporarily doubled during reallocation.

In some cases it is important to free the allocated memory long before vector destruction. For example in real-time or memory bound applications.

A common mistake is to use `clear` method, wich, by the standard, does not  affect vector capacity, and does not free the buffer. A better approach would be to call `shrink_to_fit` after `clear`. In most implementations this would indeed free the buffer, but the standard does not guarantee that, [cppreference](https://en.cppreference.com/w/cpp/container/vector/shrink_to_fit.html): 

> [*shtink_to_fit*] is a non-binding request to reduce capacity() to size(). It depends on the implementation whether the request is fulfilled

**The best way to deterministically free vector memory** is to somehow invoke vector destructor. For example by swapping the vector with a temporary that will be immediately destructed:
```c++
{
    auto tmp_vec = decltype(vec){};
    vec.swap(tmp_vec); // GOOD: old buffer is freed on tmp_vec destruction
}
```

Another approach is to invoke vector move assigment operator:
```c++
vec = decltype(vec){}; // BAD: no guarantee that vec memory is replaced
```
This would work with most compilers, but in general the result depends on implementation. Nothing prevents the implementation from optimizing the deallocation out and just calling `clear` instead.

Another imperfect approach would be to move the vector itself:
```c++
{
    auto tmp_vec = std::move(vec); // BAD: vec state is unspecified
}
```
the `tmp_vec` will be destructed, but the `vec` remains in a valid but unspecified state, meaning that it could potentially still hold its memory. Once again, your compiler will most likely do the desired thing, but there's no guarantee.


### 1.4 Insert vector elements to the same vector

It is a very common practice to insert copies of the elements from vector to itself. Consider the following code:
```c++
auto vec = std::vector<int>{1};
// 1. duplicate back element at the end
vec.push_back(vec.back());
// 2. duplicate element by certain index at the end
vec.push_back(vec[0]);
// 3. duplicate the whole vector at the end
vec.insert(/* dst */ vec.end(), /* src */ vec.begin(), vec.end());
// 4. duplicate front element at the beginning
vec.insert(vec.begin(), vec.front());
// 5. duplicate the whole vector at the beginning
vec.insert(/* dst */ vec.begin(), /* src */ vec.begin(), vec.end())
```
Looks familiar? We've all seen dozens lines of code doing something like this. But does it seem valid? What if in any case 1-4 vector capacity is exceeded and reallocation takes place? This could invalidate the element being inserted, couldn't it? And what about cases 4 and 5, in which the elements cannot be copied to the front positions until all elements are moved, but moving the elements would invalidate the references to the front positions... There are basically three points of view:

1. Of course it **is valid**. The code is too simple to be wrong.
2. Hmm, these lines **are invalid**. Elements being inserted are passed either by reference or by iterator, and `back`/`front` also return a reference. If a reallocation takes place, it invalidates the reference (or iterator) and we bump into UB.
3. Well, I believe the code **is valid**. Yes, it kind of smells, but I trust my language, the Standards Commetee and library implementers. They must have foreseen and prevented these issues.

All of a sudden, neither of the three statements is true. But fortunately, 3 is the right intuition. The only option to seek the truth is to dig into the Standard.

Here is what you can find in [C++20, N4868](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/n4868.pdf), section 22.2.3 Sequence containers, Tables 77 and 78. And there is nothing specific said about vector: only these generic Sequence containers requirements.

- `a.push_back(t)` ...  Appends a copy of t
- `a.insert(p,t)` ...  Inserts a copy of t before p
- `a.insert(p,i,j)` ... **Neither i nor j are
iterators into a** ... Inserts copies of elements in [i, j)
before p. Each iterator in the range [i, j)
shall be dereferenced exactly once.

Consider `a.push_back(t)` and `a.insert(p,t)` first. There are no explicit restrictions on `t`. *"Inserts a copy of t"* means that a copy must be inserted, no matter where it references to. **Cases 1, 2 and even 4 are valid**. If a reallocation occurs, the implementation must guarantee that the element `t` is copied before the old memory is freed or elements are transfered to the new location. The implementation is, thus, bound to the following algorithm : 1. allocate new memory, 2. copy the `t` element to back position, 3. transfer all elements, 4. free the old memory. In case of `a.insert(p,t)` the implementation is also forced to copy `t` to temporary location to avoid `t` being invalidated by elements shift.

Let us now move on to the case `a.insert(p,i,j)`, which is quite unambiguous. Iterators `i` and `j` must not point to the vector itself, and cases 3 and 5 are generally invalid... really? Let us try to apply some of the vector-specific properties to case 3. If no reallocation occurs, it is definetly valid to read the whole range and copy it to the end, since it cannot overwrite any of existing elements. Reallocations can be easily handled in same way as in `a.push_back(t)`, but the implementation is not obliged to take care of it. In fact, most stl implemetations would handle it correctly, and moreover many implementations would even successfully insert elelment somewhere into the middle of vector itself, like in case 5.

#### Conclusion

- Cases 1, 2, 4 are safe.
- Case 3 is safe if there's no reallocation, and in most implementations in case of reallocation too.
- Case 5 is not safe, but many implementaions handle it correctly.

Nevertheless, all of the code lines in cases 1-5 [smell](https://en.wikipedia.org/wiki/Code_smell). You should better avoid such constructions with `std::vector` and other containers. As you could have seen from this chapter, the reasoning behind those cases is quite entangled. Even if you are aware of all the nuances, don't complicate your colleagues' code review hours.


--- 


### 1.5 Boolean vector

// TODO: packs bits into bytes
ref: Scott Meyers


---


## Strings

// TODO: intro


### Small String Optimization

// TODO


### Constructing from `std::nullptr`

// TODO: UB


### C-style string as template parameter

// TODO

---


## `std::unordered_map` / `std::unordered_set`

// TODO: implementation, hash table, rehashing, load factor


### Iterators and pointer invalidation

// TODO: iterators are invalidated on insert, pointers are not!
ref: https://stackoverflow.com/questions/39868640/stdunordered-map-pointers-reference-invalidation


---


## Smart pointers

// TODO: implementation of shared and uniqie pts, control block, strong and weak counters
ref: Scott Meyers


### std::make_shared VS std::shared_ptr(obj_ptr)

// TODO


### Smart pointers require access to destructor

// TODO


### std::make_unique / std::make_shared requires access to constructor

// TODO

---


## Core language

// TODO: intro


### Range-based for over temporaries

// TODO


### Redundant copy with `auto`

// TODO: auto a = f(); for (auto a : vec) {}


### Deleted destructor

// TODO: only cast


### Deleted constructor

// TODO: prohibit objects on stack


### Static initialization order fiasco

// TODO:
ref: https://en.cppreference.com/w/cpp/language/siof.html


### Deleting a pointer to incomplete type

// TODO: UB, can be detected at compile time
ref: https://www.modernescpp.com/index.php/small-safety-improvements-in-the-c-26-core-language/#:~:text=Deleting%20a%20Pointer%20to%20an%20Incomplete%20Type%20should%20be%20ill%2Dformed


### RVO, NRVO

// TODO


### Initialization of unions

// TODO
ref: https://www.youtube.com/watch?v=kaI4R0Ng4E8



--- 


## Inheritance 

// TODO: intro


### Virtual destructors

// TODO


### Virtual function calls in constructors and destructors

// TODO


---


## Exceptions

// TODO: intro


### Exceptions in constructor

// TODO


### Exceptions in destructor

// TODO


### Exceptions inside new

// TODO: memory is deallocated automatically


### Never catch by value

// TODO


---


## Move semantics

// TODO


### std::move has no effect

https://stackoverflow.com/questions/2734884/why-does-iso-iec-charge-for-the-c-and-c-standards-instead-of-providing-them-fo
