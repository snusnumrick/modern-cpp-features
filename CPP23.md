# C++23

## Overview
Many of these descriptions and examples are taken from various resources (see [Acknowledgements](#acknowledgements) section) and summarized in my own words.

C++23 includes the following new language features:
- [deducing this](#deducing-this)
- [if consteval](#if-consteval)
- [multidimensional subscript operator](#multidimensional-subscript-operator)
- [auto(x) and auto{x}](#autox-and-autox)
- [labels at the end of compound statements](#labels-at-the-end-of-compound-statements)
- [size_t literal suffix](#size_t-literal-suffix)
- [\[\[assume\]\] attribute](#assume-attribute)
- [constexpr function relaxations](#constexpr-function-relaxations)
- [static operator\[\] and static operator()](#static-operator-and-static-operator)
- [extended floating-point types](#extended-floating-point-types)
- [named universal character escapes](#named-universal-character-escapes)
- [#warning directive](#warning-directive)

C++23 includes the following new library features:
- [std::expected](#stdexpected)
- [std::print and std::println](#stdprint-and-stdprintln)
- [std::mdspan](#stdmdspan)
- [std::generator](#stdgenerator)
- [flat associative containers](#flat-associative-containers)
- [std::optional monadic operations](#stdoptional-monadic-operations)
- [std::stacktrace](#stdstacktrace)
- [ranges library enhancements](#ranges-library-enhancements)
- [std::spanstream](#stdspanstream)
- [std::byteswap](#stdbyteswap)
- [std::to_underlying](#stdto_underlying)
- [std::unreachable](#stdunreachable)
- [std::ranges::contains](#stdrangescontains)
- [std::ranges::fold](#stdrangesfold)

## C++23 Language Features

### Deducing this
_Deducing this_ (also known as _explicit object parameters_) allows member functions to explicitly name the object parameter (the implicit `this` pointer) and deduce its type, including cv-qualifiers and reference type. This eliminates the need to write multiple overloads for different value categories and const-qualifications.

A member function can declare an explicit object parameter by adding `this` before the first parameter. The explicit object parameter must be the first parameter and cannot be `static`.

```c++
struct Value {
  int value;

  // Before: needed four overloads
  // int& get() & { return value; }
  // const int& get() const& { return value; }
  // int&& get() && { return std::move(value); }
  // const int&& get() const&& { return std::move(value); }

  // After: one function template handles all cases
  template <typename Self>
  auto&& get(this Self&& self) {
    return std::forward<Self>(self).value;
  }
};

Value v{42};
int& r = v.get();           // calls with Self = Value&
int&& rr = Value{42}.get(); // calls with Self = Value&&
```

This feature is particularly useful for:
- Eliminating code duplication across cv/ref-qualified overloads
- Simplifying recursive lambdas
- Making CRTP patterns more straightforward

```c++
// Recursive lambda example
auto fib = [](this auto self, int n) -> int {
  if (n <= 1) return n;
  return self(n - 1) + self(n - 2);
};
fib(10); // == 55
```

### if consteval
`if consteval` allows conditional compilation based on whether code is being evaluated in a constant expression context. Unlike `if constexpr`, it checks if the code is running at compile-time vs runtime.

```c++
constexpr size_t strlen_cpp(const char* s) {
  if consteval {
    // Compile-time evaluation
    size_t len = 0;
    while (s[len] != '\0') ++len;
    return len;
  } else {
    // Runtime evaluation - can use optimized runtime function
    return std::strlen(s);
  }
}
```

The negated form `if not consteval` is also available:
```c++
constexpr void process() {
  if not consteval {
    // Only executed at runtime
    std::cout << "Runtime execution\n";
  }
}
```

### Multidimensional subscript operator
C++23 allows the subscript operator `operator[]` to accept multiple arguments, enabling more natural syntax for multidimensional data structures.

```c++
template <typename T>
class Matrix {
  std::vector<T> data;
  size_t rows, cols;
public:
  Matrix(size_t r, size_t c) : data(r * c), rows(r), cols(c) {}

  // Multidimensional subscript operator
  T& operator[](size_t row, size_t col) {
    return data[row * cols + col];
  }

  const T& operator[](size_t row, size_t col) const {
    return data[row * cols + col];
  }
};

Matrix<int> m(3, 3);
m[0, 0] = 1;  // Natural syntax for 2D access
m[1, 2] = 5;
```

### auto(x) and auto{x}
C++23 introduces decay-copy expressions using `auto(x)` and `auto{x}` syntax. These create a prvalue copy of the expression with decayed type (removing cv-qualifiers and references).

```c++
const std::string& get_string();

void example() {
  // Before: needed std::decay_t or similar
  std::decay_t<decltype(get_string())> copy1 = get_string();

  // After: concise decay-copy
  auto copy2 = auto(get_string()); // std::string, not const std::string&
  auto copy3 = auto{get_string()}; // same as above
}
```

This is particularly useful in template contexts:
```c++
template <typename T>
void forward_copy(T&& arg) {
  // Creates a copy with decayed type
  process(auto(std::forward<T>(arg)));
}
```

### Labels at the end of compound statements
C++23 allows labels to appear immediately before the closing brace of a compound statement, eliminating the need for null statements.

```c++
void process(int n) {
  if (n < 0) {
    goto end;
  }
  // ... processing ...

end: // Previously required: "end: ;"
}
```

### size_t literal suffix
The literal suffix `z` or `Z` creates a `std::size_t` literal, and `uz` or `UZ` creates a `std::size_t` unsigned literal. This makes it easier to avoid type mismatches and warnings when working with sizes.

```c++
auto size = 42z;  // std::size_t, not int
static_assert(std::same_as<decltype(size), std::size_t>);

std::vector<int> vec{1, 2, 3, 4, 5};
for (auto i = 0z; i < vec.size(); ++i) {  // No sign comparison warning
  std::cout << vec[i] << '\n';
}
```

### [[assume]] attribute
The `[[assume]]` attribute allows programmers to provide assumptions to the compiler for optimization purposes. If the assumption is violated at runtime, the behavior is undefined.

```c++
void process(int* ptr) {
  [[assume(ptr != nullptr)]];  // Tell compiler ptr is never null
  *ptr = 42;  // Compiler can optimize away null check
}

int divide(int a, int b) {
  [[assume(b != 0)]];  // Assume no division by zero
  return a / b;
}
```

Use with caution: incorrect assumptions lead to undefined behavior.

### constexpr function relaxations
C++23 relaxes some restrictions on `constexpr` functions:
- Non-literal variables are now allowed (variables that cannot be used in constant expressions)
- `static` and `thread_local` variables are allowed
- `goto` statements and labels are allowed

```c++
constexpr int compute(int n) {
  // goto allowed in C++23
  if (n < 0) goto negative;
  return n * 2;

negative:
  return 0;
}

constexpr int with_static() {
  static int cache = 42;  // Allowed in C++23
  return cache;
}
```

### static operator[] and static operator()
C++23 allows `operator[]` and `operator()` to be declared `static`, which is useful when these operators don't need access to instance data.

```c++
struct Configuration {
  static constexpr int operator[](std::string_view key) {
    if (key == "max_size") return 1024;
    if (key == "timeout") return 30;
    return 0;
  }
};

// Usage without instance
int max = Configuration::operator[]("max_size");
int timeout = Configuration{}["timeout"];  // Also works with instance
```

### Extended floating-point types
C++23 introduces optional extended floating-point type aliases in `<stdfloat>`:
- `std::float16_t` - 16-bit floating-point
- `std::float32_t` - 32-bit floating-point (typically same as `float`)
- `std::float64_t` - 64-bit floating-point (typically same as `double`)
- `std::float128_t` - 128-bit floating-point
- `std::bfloat16_t` - Brain floating-point (16-bit, used in ML)

```c++
#include <stdfloat>

void compute() {
  std::float32_t f32 = 1.5f;
  std::float64_t f64 = 2.5;

  #ifdef __STDCPP_FLOAT16_T__
  std::float16_t f16 = 1.0f16;  // If supported
  #endif
}
```

These types may not be available on all platforms. Check for support with feature test macros.

### Named universal character escapes
C++23 allows using character names in escape sequences with `\N{NAME}` syntax, making Unicode characters more readable.

```c++
const char* smiley = "\N{SMILING FACE WITH SMILING EYES}";  // U+1F60A
const char* lambda = "\N{GREEK SMALL LETTER LAMDA}";        // λ
const char* arrow = "\N{RIGHTWARDS ARROW}";                  // →

// More readable than:
// const char* smiley = "\U0001F60A";
```

### #warning directive
The `#warning` preprocessor directive generates a compiler warning with a custom message, similar to `#error` but without stopping compilation.

```c++
#ifndef OPTIMIZED
  #warning "Building without optimization enabled"
#endif

#ifdef DEPRECATED_API
  #warning "Using deprecated API, please migrate to new version"
#endif
```

## C++23 Library Features

### std::expected
`std::expected<T, E>` is a vocabulary type that represents a value that might be either a successful result of type `T` or an error of type `E`. It provides an alternative to exceptions for error handling.

```c++
#include <expected>
#include <string>

std::expected<int, std::string> divide(int a, int b) {
  if (b == 0) {
    return std::unexpected("Division by zero");
  }
  return a / b;
}

void use() {
  auto result = divide(10, 2);
  if (result.has_value()) {
    std::cout << "Result: " << result.value() << '\n';  // 5
  } else {
    std::cerr << "Error: " << result.error() << '\n';
  }

  // Or use value_or for a default
  int value = divide(10, 0).value_or(-1);  // -1
}
```

`std::expected` can be used with monadic operations:
```c++
auto safe_sqrt(double x) -> std::expected<double, std::string> {
  if (x < 0) return std::unexpected("Negative value");
  return std::sqrt(x);
}

auto result = safe_sqrt(16.0)
  .and_then([](double x) { return safe_sqrt(x); })  // Chain operations
  .transform([](double x) { return x * 2; });        // Transform success value
```

### std::print and std::println
C++23 introduces `std::print` and `std::println` for formatted output using the formatting syntax from C++20's `std::format`.

```c++
#include <print>

void example() {
  std::print("Hello, {}!\n", "World");
  std::println("The answer is {}", 42);

  // Type-safe formatting
  std::println("Hex: {:x}, Octal: {:o}, Binary: {:b}", 255, 255, 255);
  // Hex: ff, Octal: 377, Binary: 11111111

  // Formatting with alignment and width
  std::println("{:>10} | {:<10}", "right", "left");
  //      right | left
}
```

Unlike `std::cout`, these functions are simpler, more efficient, and provide better formatting capabilities.

### std::mdspan
`std::mdspan` is a multidimensional span providing a view over a contiguous sequence of objects with multidimensional indexing.

```c++
#include <mdspan>
#include <vector>

void example() {
  std::vector<int> data(24);

  // Create 3D view: 2x3x4
  std::mdspan<int, std::dextents<size_t, 3>> view3d(
    data.data(), 2, 3, 4
  );

  // Access with multidimensional indices
  view3d[1, 2, 3] = 42;

  // Iterate over dimensions
  for (size_t i = 0; i < view3d.extent(0); ++i) {
    for (size_t j = 0; j < view3d.extent(1); ++j) {
      for (size_t k = 0; k < view3d.extent(2); ++k) {
        std::println("view3d[{}, {}, {}] = {}", i, j, k, view3d[i, j, k]);
      }
    }
  }
}
```

Different layout policies can be specified:
```c++
// Row-major (C-style)
std::mdspan<int, std::dextents<size_t, 2>, std::layout_right> mat_row(
  data.data(), 3, 3
);

// Column-major (Fortran-style)
std::mdspan<int, std::dextents<size_t, 2>, std::layout_left> mat_col(
  data.data(), 3, 3
);
```

### std::generator
`std::generator` is a view (range) that represents a synchronous coroutine generator. It simplifies writing generator coroutines that yield values.

```c++
#include <generator>
#include <ranges>

std::generator<int> fibonacci() {
  int a = 0, b = 1;
  while (true) {
    co_yield a;
    auto next = a + b;
    a = b;
    b = next;
  }
}

void example() {
  for (int n : fibonacci() | std::views::take(10)) {
    std::println("{}", n);  // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
  }
}
```

Generators can be used with ranges:
```c++
std::generator<std::string> generate_words() {
  co_yield "Hello";
  co_yield "C++23";
  co_yield "Generators";
}

auto words = generate_words()
  | std::views::transform([](auto s) { return s.size(); });
```

### Flat associative containers
C++23 introduces flat container adaptors that store elements in contiguous memory for better cache locality and performance:
- `std::flat_set` and `std::flat_multiset`
- `std::flat_map` and `std::flat_multimap`

```c++
#include <flat_map>

void example() {
  std::flat_map<std::string, int> scores;

  scores["Alice"] = 95;
  scores["Bob"] = 87;
  scores["Charlie"] = 92;

  // Elements stored contiguously, better cache performance
  for (const auto& [name, score] : scores) {
    std::println("{}: {}", name, score);
  }

  // Lookup is still logarithmic, but faster due to cache locality
  if (auto it = scores.find("Bob"); it != scores.end()) {
    std::println("Bob's score: {}", it->second);
  }
}
```

Flat containers trade insertion/deletion performance for better iteration and lookup performance due to memory layout.

### std::optional monadic operations
C++23 adds monadic operations to `std::optional` (and `std::expected`), enabling functional-style chaining without explicit checks:
- `and_then` - chains operations that return `std::optional`
- `transform` - applies a function to the contained value
- `or_else` - provides an alternative when empty

```c++
#include <optional>

std::optional<int> parse_int(std::string_view s);
std::optional<double> safe_sqrt(int x);

void example() {
  std::optional<std::string> input = get_input();

  auto result = input
    .and_then([](auto s) { return parse_int(s); })
    .and_then([](int x) { return safe_sqrt(x); })
    .transform([](double x) { return x * 2; })
    .or_else([]() { return std::optional{0.0}; });

  // Without monadic operations:
  // std::optional<double> result;
  // if (input) {
  //   if (auto x = parse_int(*input)) {
  //     if (auto y = safe_sqrt(*x)) {
  //       result = *y * 2;
  //     }
  //   }
  // }
  // if (!result) result = 0.0;
}
```

### std::stacktrace
The `<stacktrace>` library provides a portable way to capture and examine the call stack, useful for debugging and error reporting.

```c++
#include <stacktrace>
#include <iostream>

void third() {
  std::println("Current stacktrace:");
  auto trace = std::stacktrace::current();
  std::println("{}", std::to_string(trace));
}

void second() { third(); }
void first() { second(); }

int main() {
  first();
}
```

Stacktraces can be captured and examined programmatically:
```c++
void handle_error() {
  auto trace = std::stacktrace::current();

  std::println("Error occurred at:");
  for (const auto& entry : trace) {
    std::println("  {} at {}:{}",
      entry.description(),
      entry.source_file(),
      entry.source_line());
  }
}
```

### Ranges library enhancements
C++23 significantly expands the ranges library with new views and algorithms:

New views:
```c++
#include <ranges>

void example() {
  std::vector v{1, 2, 3, 4, 5, 6};

  // views::chunk - divides range into chunks
  for (auto chunk : v | std::views::chunk(2)) {
    // {1,2}, {3,4}, {5,6}
  }

  // views::slide - sliding window
  for (auto window : v | std::views::slide(3)) {
    // {1,2,3}, {2,3,4}, {3,4,5}, {4,5,6}
  }

  // views::stride - every nth element
  for (int n : v | std::views::stride(2)) {
    // 1, 3, 5
  }

  // views::cartesian_product - Cartesian product
  std::vector a{1, 2};
  std::vector b{'a', 'b'};
  for (auto [n, c] : std::views::cartesian_product(a, b)) {
    // (1,'a'), (1,'b'), (2,'a'), (2,'b')
  }

  // views::zip - zips multiple ranges
  std::vector names{"Alice", "Bob"};
  std::vector ages{25, 30};
  for (auto [name, age] : std::views::zip(names, ages)) {
    std::println("{} is {} years old", name, age);
  }

  // views::enumerate - elements with indices
  for (auto [i, val] : v | std::views::enumerate) {
    std::println("[{}] = {}", i, val);
  }

  // views::adjacent - adjacent elements
  for (auto [a, b] : v | std::views::adjacent<2>) {
    std::println("{}, {}", a, b);
  }
}
```

### std::spanstream
`std::spanstream` provides stream operations over a fixed-size buffer using `std::span`, similar to `std::stringstream` but without allocation.

```c++
#include <spanstream>
#include <span>

void example() {
  char buffer[128];
  std::span<char> buf_span(buffer);

  // Output span stream
  std::ospanstream oss(buf_span);
  oss << "The answer is " << 42;

  // Get written data
  std::span<char> written = oss.span();

  // Input span stream
  std::ispanstream iss(written);
  std::string text;
  int number;
  iss >> text >> text >> number;  // "The answer is 42"
}
```

### std::byteswap
`std::byteswap` reverses the bytes of an integer, useful for endianness conversion.

```c++
#include <bit>

void example() {
  uint32_t value = 0x12345678;
  uint32_t swapped = std::byteswap(value);  // 0x78563412

  // Useful for network byte order conversion
  uint16_t host_port = 8080;
  uint16_t network_port = std::byteswap(host_port);
}
```

### std::to_underlying
`std::to_underlying` converts an enumeration to its underlying integer type.

```c++
#include <utility>

enum class Color : uint8_t { Red, Green, Blue };

void example() {
  Color c = Color::Green;
  auto value = std::to_underlying(c);  // uint8_t with value 1
  static_assert(std::same_as<decltype(value), uint8_t>);
}
```

### std::unreachable
`std::unreachable()` indicates that a code path should never be reached. If executed, the behavior is undefined, but it helps the compiler optimize.

```c++
#include <utility>

int handle_value(int x) {
  switch (x) {
    case 0: return 10;
    case 1: return 20;
    case 2: return 30;
    default: std::unreachable();  // Tell compiler x is always 0, 1, or 2
  }
}
```

### std::ranges::contains
Convenience algorithms to check if a range contains a value or satisfies a predicate.

```c++
#include <algorithm>
#include <vector>

void example() {
  std::vector v{1, 2, 3, 4, 5};

  bool has_3 = std::ranges::contains(v, 3);  // true
  bool has_even = std::ranges::contains(v, [](int x) {
    return x % 2 == 0;
  });  // true
}
```

### std::ranges::fold
Fold algorithms provide a functional approach to accumulation:
- `std::ranges::fold_left`
- `std::ranges::fold_right`
- `std::ranges::fold_left_first`
- `std::ranges::fold_right_last`

```c++
#include <algorithm>
#include <vector>

void example() {
  std::vector v{1, 2, 3, 4, 5};

  // Sum with initial value
  int sum = std::ranges::fold_left(v, 0, std::plus{});  // 15

  // Product using first element as initial value
  auto product = std::ranges::fold_left_first(v, std::multiplies{});
  // product is std::optional<int> with value 120

  // Build string from right
  std::vector<std::string> words{"Hello", "C++", "23"};
  auto sentence = std::ranges::fold_right(words, std::string{},
    [](const auto& word, const auto& acc) {
      return acc.empty() ? word : word + " " + acc;
    });
  // "Hello C++ 23"
}
```

## Acknowledgements
* [cppreference](http://en.cppreference.com/w/cpp) - especially useful for finding examples and documentation of new library features.
* [clang](http://clang.llvm.org/cxx_status.html) and [gcc](https://gcc.gnu.org/projects/cxx-status.html)'s standards support pages. Also included here are the proposals for language/library features that I used to help find a description of, what it's meant to fix, and some examples.
* [Compiler explorer](https://godbolt.org/)
* [C++ Stories](https://www.cppstories.com/) - excellent blog covering modern C++ features with practical examples.
* [C++23 - Wikipedia](https://en.wikipedia.org/wiki/C++23)
* [Modern C++](https://www.modernescpp.com/) - Rainer Grimm's blog on Modern C++.

## Author
Anthony Calandra

## Content Contributors
See: https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

## License
MIT
