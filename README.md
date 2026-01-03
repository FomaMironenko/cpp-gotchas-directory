# C++ Advanced Primer

***What every C++ programmer must know about C++***

I came up with idea of this primer when I was conducting technical interviews in 2GIS. It turns out that many developers make (in their code or just mentally) same common mistakes. Some of them are trivial to catch, some of them are not. This primer is a compilation of such pitfalls and handful facts that on one hand can make you happy if you know about them, and on the other hand can turn your coding experience into nightmare if you don't. As an interviewer I'd expect that a candidate to a Senior position knows at least 90% of the underlying facts. As an interviewee I wish I had read this primer a couple of years ago.

The primer consists of several sections. Sections are grouped into paragraphs by higher-level topics. Some of the paragraphs and sections contain basic description of the core language ideas to provide context, which can be safely skipped by advanced readers.

This primer does not explain language basics. We assume that the reader has already studied a couple of courses about the Language and has at least a couple years of hands-on experience.

---

## 1. std::vector

`std::vector<T>` is dynamic array, which means the number of its elements (**size**) can change during lifetime. The *size*, however, does not equal the actual amount of allcated space (**capacity**). *Capacity* is the number of elements of type `T` that can fit into the currently allocated memory.

Vector *size* can never exceed *capacity*. If at any point of execution *capacity* of a vector equals its *size* and user attempts to insert a new element, a **reallocation** takes place. During the *reallocation*:
- a new memory chunk, bigger than the current capacity, is allocated,
- all the elements are transferred (usually, but not always, by move sematics) to the new location,
- the internal vector data pointer is overwritten with the address of the new location. 

In accordance to what was just said, vector implementation must store and manipulate the followinng members: 1. pointer to the allocated chunk, 2. vector size, 3.vectror capacity.

Alternatively, the implementation might store: 1. pointer to the allocated chunk, 2. pointer to the end of currently stored elements, 3. pointer to the end of the allocated chunk.

Both implementations store exactly three machine words, and thus it is the size of a vector object (unless a custom allocator is specified).

#### `std::vector` operational complexity
For a vector of size `N` and capacity `C` the complexity is as follows:

- **Insert an element at position `I <= N`: if `N < C`, then `O(N-I)`, else `O(N)`**
Specifically, the implementation would need to shift `N - I` elements towards the end (`O(N - I)`) and in case `N == C` allocate a new chunk and transefer **all** elements there (`O(N)`).

- **Erase an element at position `I < N`: `O(N-I)`**
Arguments are the same as in the case of insertion, except that now there's no need for reallocation. It is only necessary to shift `N - I` tail elements towards the begin.

- **Insert an element at the end (`push_back`): amortized `O(1)`**
There's no need to shift any elements. If `N < C` (most of the cases), there is enough capacity for yet another element, and adding new item is `O(1)`. In rare case when `N == C`, a reallocation takes place and the `push_back` would instantly cost `O(N)`. But the [amortized complexity](https://en.wikipedia.org/wiki/Amortized_analysis) when multiple `push_backs` are executed is still `O(1)`.

- **Erase the `back` element: `O(1)`**
There's no need for reallocation, and the complexity is always constant.


---


### 1.1 Iterators invalidation

The most common source of bugs in vector is iterators, references and pointers (to vector elements) invalidation. In common implementations vector iterator is nothing but pointer to a location within the allocated chunk. Thus, unlike in some other data structures, vector iterators, references and pointers are all invalidated simultaneously. Here and below we will omit "references and pointers" part and only speak about iterators.

Every reallocation or elements shift invalidate (some) iterators, but these are internally different scenarios. Elements shift would cause iterators to shifted elements point to unexpected elemnts within the allocated chunk and possibly within vector size. Reallocation would make every iterator dangling, with all ensuing consequences.

The scenarios of iterators invalidation are:

- **`reserve` and `shrink_to_fit`**: (might) cause a reallocation, thus invalidate all iterators.

- **`clear`**: invalidates all iterators. Invokes destructors for all the elements, but does not free the underlying memory. The memory locations can still be accessible by the iterators but the content is undefined.

- **`resize` to new size `M`**: if `M < N`, invalidates iterators to elements at positions `M <= I < N`. If `N <= M <= C`, no iterators are invalidated. Finally, if `M > C`, all iterators are invalidated due to reallocation.

- **Insert an element to position `I <= N` (`push_back` in particular)**: invalidates iterators to positions `J >= I` due to shift, and if `N == C` invalidates all iterators due to reallocation. Since reallocation is transparent to user, you must be particularly careful and make sure there was no reallocation before accessing iterators to positions `J < I`.

```C++
std::vector<int> v = {1, 2, 3};
v.reserve(4);
auto begin_it = v.begin();
v.push_back(4); // guaranteed no reallocation, begin_it IS VALID
v.push_back(5); // reallocation is possible,   begin_it IS NO MORE VALID
```

- **Erase an element at position `I < N`**:
invalidates iterators to positions `J >= I` due to shift.


---


### 1.2 Transfering elements on reallocation

During reallocation all existing elements need to be transferred from the previous memory chunk to the newly allocated one. The main question is how to do this efficiently and exception-safely. Reallocations must provide [strong exception guarantee](https://en.cppreference.com/w/cpp/language/exceptions.html#Exception_safety), meaning that if during reallocation an exception is thrown, the operation has no effect and the vector preserves initial state.

This turns out to be quite tricky if you don't want to copy each element, because move semantics mutates source elements and leaves them in unspecified state. If half of the elements have already been copied to the new destination, and the next call to move constructor throws an error, there's no way to restore the moved elements. You could try to move them back from destination to source. But if move constructor throws again, that is fatal. The same does not apply to copy constructor, that does not modify source elements.

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
    UnsafelyMovable(const UnsafelyMovable&) {  }
};
static_assert(!std::is_nothrow_move_constructible_v<SafelyMovable>, "Implicitly not noexcept");

// "GOOD": will be unsafely moved
struct NotCopyable {
    NotCopyable(NotCopyable&&) {  };
    NotCopyable(const NotCopyable&) = delete;
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


---


### 1.3 Clean up vector memory

Allocated vector memory is automatically freed on vector destruction. During reallocation old data chunk is also freed. Note, however, that memory consumption is temporarily doubled during reallocation.

In some cases, for example in real-time or memory bound applications, it is important to free the allocated memory long before vector destruction. A common mistake is to use `clear` method, wich, by the standard, does not  affect vector capacity, and does not free the buffer. A better approach would be to call `shrink_to_fit` after `clear`. In most implementations this would indeed free the buffer, but the standard does not guarantee that, [cppreference](https://en.cppreference.com/w/cpp/container/vector/shrink_to_fit.html): 

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
vec = {}; // BAD: no guarantee that vec memory is replaced
```
This would work with most compilers, but in general the result depends on implementation. Nothing prevents the implementation from optimizing the deallocation out and just calling `clear` instead.

Another imperfect approach would be to move the vector itself:
```c++
{
    auto tmp_vec = std::move(vec); // BAD: vec state is unspecified
}
```
the `tmp_vec` will be destructed, but the `vec` remains in a valid but unspecified state, meaning that it could potentially still hold its memory. Once again, your compiler will most likely do the desired thing, but there's no guarantee.


#### Conclusion

The best way to clear and free vector memory before its destruction is to swap it with a temporary vector that will be immediately destructed:
```c++
template <typename T>
void ClearAndFree(std::vector<T>& vec)
{
    auto destroyed_after_scope = std::vector<T>{};
    std::swap(vec, destroyed_after_scope);
}
```
A more elegant solution is to simply assign the vector with a temporary. This would work in ~~all~~ most implementations. The Standard, however, does not guarantee that the memory is actually freed, and the implementations are allowed to avoid memory manipulations
```c++
template <typename T>
void ClearAndFree(std::vector<T>& vec)
{
    vec = {};
}
```


---


### 1.4 Insert vector elements to the same vector

It is quite a common practice to insert copies of elements from vector to itself. Consider the following code:
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
Looks familiar? We've all seen dozens lines of code doing something like this. But does it seem valid? What if in any case 1-5 vector capacity is exceeded and reallocation takes place? This could invalidate the element being inserted, couldn't it? And what about cases 4 and 5, in which the elements cannot be copied to the front positions until all elements are moved, but moving the elements would invalidate the references to the front positions... There are basically three points of view:

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

Consider `a.push_back(t)` and `a.insert(p,t)` first. There are no explicit restrictions on `t`. *"Inserts a copy of t"* means that a copy must be inserted, no matter where `t` references to. **Cases 1, 2 and even 4 are valid**. If a reallocation occurs, the implementation must guarantee that the element `t` is copied before the old memory is freed or elements are transfered to the new location. The implementation is, thus, bound to the following algorithm : 1. allocate new memory, 2. copy the `t` element to back position, 3. transfer all elements, 4. free the old memory. In case of `a.insert(p,t)` the implementation is also forced to copy `t` to temporary location to avoid `t` being invalidated by elements shift.

Let us now move on to the case `a.insert(p,i,j)`, which is quite unambiguous. Iterators `i` and `j` must not point to the vector itself, and cases 3 and 5 are generally invalid... really? Let us try to apply some of the vector-specific properties to case 3. If no reallocation occurs, it is definetly valid to read the whole range and copy it to the end, since it cannot overwrite any of existing elements. Reallocations can be easily handled in same way as in `a.push_back(t)`, but the implementation is not obliged to take care of it. In fact, most stl implemetations would handle it correctly, and moreover many implementations would even successfully insert elelment somewhere into the middle of vector itself, like in case 5.

#### Conclusion

- Cases 1, 2, 4 are safe.
- Case 3 is safe if there's no reallocation, and in most implementations in case of reallocation too.
- Case 5 is not safe, but many implementaions handle it correctly.

Nevertheless, all of the code lines in cases 1-5 [smell](https://en.wikipedia.org/wiki/Code_smell). You should better avoid such constructions with `std::vector` and other containers. As you could have seen from this chapter, the reasoning behind those cases is quite entangled. Even if you are aware of all the nuances, don't complicate your colleagues' code review hours.

#### References

- The standard [C++20, N4868](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/n4868.pdf), section 22.2.3 (Sequence containers)

--- 


### 1.5 Boolean vector

Boolean values conceptually represent just a single bit (true or false). The `bool` type, however, occupies (at least) a single byte, and there is actually only a single significant bit out of 8. This causes x8 overhead in space, which can be especially harmful for large arrays of `bool`.

Fortunately, the Standard kindly enables `std::vector<bool>` to be space-efficient by packing boolean values into bits. Instead of storing each distinct element with one-byte offset, `std::vector<bool>` packs eight boolean values into single byte. This approach has great performance benefits in terms of memory utilization. However, this leads to very important and unobvious consequences.

Unlike with other vector template parameters, **`std::vector<bool>` element access methods `back`, `front`, `operator[]`, `at` and iterator dereferencing do not return `bool&` or `const bool&`**. Indeed, this would be impossible since reference cannot be bound to the middle of a byte, where the actual value is encoded. However, you can still write code like this:
```c++
auto bool_vec = std::vector<bool>{true, true};
assert(bool_vec.back());
bool_vec[0] = false;
assert(!bool_vec.front());
```
This is possible due to the following trick. These methods return a proxy object that implements an "interface" of a boolean value. Under the hood the proxy refers to the corresponding bit somewhere inside the vector guts. **The proxy is returned by value, but has a reference semantics**. Thus, the following code does not compile due to attempt to bind non-const lvalue reference `val` to rvalue of the proxy type:
```c++
for (auto& val : bool_vec) {  } // BAD, does not compile
```
This error can be fixed if using a universal reference, which in this case would become an rvalue-reference:
```c++
for (auto&& val : bool_vec) {  } // GOOD, compiles
```
However, `val` is not a reference to `bool`, it is a reference to temporary proxy object, which lifetime is extended to the loop scope.

The proxy object can be implicitly casted to bool. And the following code compiles:
```c++
for (bool val : bool_vec) {  } // GOOD, compiles
```

The proxy object has a reference semantics. Even when stored by value it can modify the underlying vector data and.
```c++
auto bool_vec = std::vector<bool>{true, true};
auto local_val = bool_vec[0];
local_val = false; // BAD, modifies bool_vec
```

The proxy is invalidated simultaneously to the corresponding pointer or iterator:
```c++
auto bool_vec = std::vector<bool>{true, true};
auto local_val = bool_vec[0];
bool_vec.reserve(100); // local_val is no more valid due to reallocation
```

One other consequence of the `std::vector<bool>` special implementation is that internal values representation is implementation-defined and raw `std::vector<bool>` data would make no sence to the user. For that reason the Standard does not provide a `data` method. In some implementations this method has a `void` type and does nothing.



#### Conclusion

- `std::vector<bool>` elements access methods return proxy objects intead of `bool` references. It is, thus, invalid to assume in template code that `std::vector<T>::operator[]` or other accessors return `T&` or `const T&`. If you need to utilize the return type of those methods, you should use `std::vector<T>::reference` (or `const_reference`).
- Even though the proxy is returned by value, it has a reference semantics. It can modify the underlying vector and is invalidated together with corresponding pointer and iterator.
- `std::vector<bool>` has no `data` method.

#### References

- Scott Meyers, [Effective modern C++](https://www.oreilly.com/library/view/effective-modern-c/9781491908419/), Item 6
- The Standard: [C++20, N4868](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/n4868.pdf), section 22.3.12 (Class vector\<bool\>)
