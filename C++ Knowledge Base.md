## Prefer `const`, `enum` and `inline` to `#define`
Prefer the compiler to the preprocessor. Preprocessor replaces the variables with their defined types before passing code to compiler. This might cause bugs during compilation, debugging or runtime. Replacing function calls to avoid incurring function call overhead also leads to bugs. 
### Examples
Bad:
```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

Good:
```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b) {
	f(a > b ? a : b);
}
```

## Use `const` whenever possible
This helps compilers detect usage errors. There's two types of constness:
- Bitwise constness - compiler enforced. Forbids from changing any member value. This is bad when we want the const method to do some logging or behind-the-scenes caching.
- Logical constness - this should be preferred by the programmer. Const member functions can modify some data in class but only in ways that client cannot detect.
### Examples
Bad:
```cpp
char *p = greeting;
```

Good:
```cpp
const char * const p = greeting;
```

## Make sure that objects are initialized before they're used
Always initialize class members. The most efficient way to do so is using initialization lists. Note that members in the initialization list should be listed in the same order as they are declared in the class itself.
### Examples
Bad:
```cpp
int x;

Widget::Widget(const std::string& title) {
	mainTitle = title;
}

extern Widget w;
```

Good:
```cpp
/* Manually initialize objects of built-in type */
int x = 0;

/* Prefer initialization lists in ctors (list data members in the initialization list in the same order they're declared in the class) */
Widget::Widget(const std::string& title)
: mainTitle(title) {}

/* Replace non-local static objects with local static objects */
Widget& w() {
	static Widget w;
	return w; 
}
w();
```