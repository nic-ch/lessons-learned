[← Back](README.md)

# C++

## `decltype(auto)` and `auto`

As much as possible, except for copy and move assignment operators, function return types shall be one of:

* `void`,
* `bool`,
* `decltype(auto)`.

Initialized variables shall be declared as `auto`.

## Exceptions

`assert` shall never be called and exceptions shall be thrown on ***all*** unrecoverable errors, in case the caller wants to recover, save some states (e.g. the current weights), or cleanup the stack before propagating the (by definition exceptional) failure. `static_assert()` shall be used to validate conditions at compile time.

Exceptions shall ***never*** happen only from user inputs and shall always mark a bug or an issue internal to the program.

Although exceptions are technically very low cost at run time, functions that have to be optimized shall be declared `noexcept` as much as possible.

## Move Semantics

In functions, *rvalue references* may need to be passed via `std::move()` when passed down to other functions, e.g.:

```c++
// f() is called with an rvalue reference.
void f(Type&& x)
{
  // x is an lvalue reference here.
  g(x);             // x is still needed.
  h(std::move(x));  // x is not needed anymore.
}
```

Also in functions, *forwarding references* may need to be passed via `std::forward<>()` when passed down to other functions, e.g.:

```c++
// Both f(Type&) and f(Type&&) may be instantiated and called.
template<typename Type>
void f(Type&& x)
{
  // x is an lvalue reference here.
  g(x);                      // x is still needed.
  h(std::forward<Type>(x));  // x is not needed anymore.
}
```

## Pointers

The only two occasions where *raw pointers* shall be used is for utilizing `*this` and `main()`'s `argv` argument. Everywhere else, `std::unique_ptr` or `std::shared_ptr` shall be used. `std::unique_ptr` at is pretty much zero-cost at run time while `std::shared_ptr` is very low cost. They both solve the problem of shared objects deallocation and a collection of them can be gathered in a `std::vector`. As much as possible, smart pointers will only be created via `std::make_unique` and `std::make_shared`.

## RAII (Resource Acquisition is Initialization)

RAII shall be used to lock mutexes so to guarantee they get unlocked.

```c++
  std::mutex myMutex;
  ...
  // Lock guard context.
  {
    std::lock_guard const lockGuard(myMutex);
    ...
  }
```

Also, a local `struct` can be declared in a function to guarantee the deallocation of some resources, irrespective of how the functions returns or if exceptions are thrown (and caught downstream). Because of its ephemeral nature, such a `struct` does not need to follow the [rule of five](https://en.cppreference.com/w/cpp/language/rule_of_three). For example, the following will **always** print 3:

```c++
void f(int& x)
{
  struct Deallocator {
    int& deallocator_x;
    Deallocator(int& constructor_x) : deallocator_x(constructor_x) { }
    ~Deallocator() { deallocator_x = 3; }
  } deallocator(x);

  srand(time(nullptr));
  if (rand() % 2) { x = 1; return; }
  if (rand() % 2) { throw true; }
  x = 2;
}
int main()
{
  int a;
  try { f(a); }
  catch (...) { }
  std::cout << a << '\n';
}
```

## Range-based For Loop

If the range's values need to be modified, then a *forwarding reference* shall be used:

```c++
  for (auto&& element : collection)
    ++element;
```

Otherwise, if the range's values need only be read, a *const reference* shall be used as it can almost be viewed as a read-only "universal reference":

```c++
  for (auto const& element : collection)
    std::cout << element;
```

## Templated Code Proliferation

Templated coded may, under some circumstances, be instantiated *en masse* if left unchecked. The four following habits shall therefore be implemented (knowing that compilers may nonetheless inline some code).

### 1. De-templatize common code, e.g.:

```c++
void common()
{
  ...
}

template<typename Type>
void f(Type&& x)
{
  g(std::forward<Type>(x));
  common();
}
```

### 2. When calling templated functions, *decay* C strings to type `char*` by prefixing them with `+`:

All C string types can be passed to functions by implicitly converting them on the fly to a `std::strings` as follows:

```c++
void f(std::string const& name)
{
  ...
}

f("Hello");
char s[100]{ "Ciao" };
f(s);
```

This has the disadvantage of creating and destroying on the stack a `std::string` each and every time `f()` is called. To save this run time cost, one may be tempted to templatize `f()` thus, so to forward C strings and `std::strings` as is:

```c++
template<typename String>
void f(String const& name)
{
  ...
}
```

Unfortunately this will lead to templated code proliferating as, according to [C++ Insights](https://cppinsights.io/), the following will instantiate `f()` **six** times to handle C strings only:

```c++
template<typename String>
void f(String const& name)
{
  ...
}

int main()
{
  f("Hello");             // (1) Instantiates 'f<char [6]>'.
  f("Hola");              // (2) Instantiates 'f<char [5]>'.
  f("Bonjour");           // (3) Instantiates 'f<char [8]>'.
  char s1[10]{ "Ciao" };
  f(s1);                  // (4) Instantiates 'f<char [100]>'.
  auto s2{ "Buna ziua" };
  f(s2);                  // (5) Instantiates 'f<const char *>'.
  char* s3{ s1 };
  f(s3);                  // (6) Instantiates 'f<char *>'.

  f(std::string("Olá"));  // Instantiates 'f<std::basic_string<char>>'.

  f(+"Hello");            // Uses 'f<const char *>'.
  f(+"Hola");             // Uses 'f<const char *>'.
  f(+"Bonjour");          // Uses 'f<const char *>'.
  f(+s1);                 // Uses 'f<char *>'.
}
```

Therefore, when calling templated functions, *decay* C strings to type `char*` by prefixing them with `+`.

### 3. When this can not be controlled and C strings and `std::string` need only to be passed through, define the three following functions:

```c++
template<typename String>
void original_f(String const& name)
{
  ...
}

void f(char const* const name)
{
  original_f(name);  // Instantiates 'original_f<const char *>'.
}
void f(char* const name)
{
  // Cast name to 'char const*' to use 'original_f<const char *>'.
  original_f(static_cast<char const*>(name)); 
}
template<typename String>
void f(String const& name)
{
  original_f(name);
}
```

This will instantiate `original_f()` only **once** to handle all C strings.

### 4. When a `std::string` needs to be retained could be moved from, define the following two functions:

```c++
void f(std::string&& s)
{
  my_s = std::move(s);
}
void f(std::string const& s)
{
  f(std::string(s));
}
```

The first `f()` will catch all C strings, that will be implicitly converted on the stack to r-value `std::strings` that will be moved from. The second `f()` will catch actual l-value `std::strings`, duplicate them to r-value `std::strings` and pass them to the first `f()`. The first `f()` will also catch anything at all that can be implicitly converted to a `std::string`.