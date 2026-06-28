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

## Declare a virtual destructor in a class if and only if that class contains at least one virtual function
Not having a virtual destructor on a base class that is being inherited can lead to undefined behavior. It happens when trying to delete a pointer that is of base type but actually has other dynamic type.
### Examples
Bad:
```cpp
class TimeKeeper { 
public: 
    TimeKeeper();
    ~TimeKeeper();
};

class AtomicClock: public TimeKeeper { ... };

TimeKeeper *ptk = getTimeKeeper();
delete ptk;
```

Good:
```cpp
class TimeKeeper { 
public:
    TimeKeeper(); 
    virtual ~TimeKeeper();
}; 
TimeKeeper *ptk = getTimeKeeper();
delete ptk;
```

## Prevent exceptions from leaving destructors
Destructors should never throw errors because they are left unhandled and program may have undefined behavior. Two main ways of dealing with exceptions in destructors are:
- Terminating the program
- Swallowing the exception
If cleanup requires exception handling it's better to move the cleanup logic into a separate cleanup function that clients may call themselves and handle exceptions as they see fit.

## Assignment operators should return a reference to `*this`
Is it good convention to follow because all standard library types follow it as well. This allows chaining of assignments like so: `x = y = z = 15`
### Example
```cpp
class Widget { 
public: 
	...
	Widget& operator+=(const Widget& rhs) // the convention applies to
	{                                     // +=, -=, *=, etc. 
		... 
		return *this; 
	} 
	
	Widget& operator=(int rhs) // it applies even if the
	{                          // operator’s parameter type 
		...                    // is unconventional 
		return *this; 
	} 
	... 
};
```

## Handle assignment to self in `operator=`
Assignment to self can be quite common in code like this `a[i] = a[j]` or this `*px = *py`.

There are a couple of ways to deal with it:
1. Explicitly handle assignment to self
```cpp
Widget& Widget::operator=(const Widget& rhs) 
{ 
	if (this == &rhs) return *this; // identity test: if a self-assignment, 
	                                // do nothing 
	delete pb; 
	pb = new Bitmap(*rhs.pb); 
	return *this; 
}
```

2. Making assignment exception-safe also makes it self-assignment safe
```cpp
Widget& Widget::operator=(const Widget& rhs) 
{ 
	Bitmap *pOrig = pb;         // remember original pb 
	pb = new Bitmap(*rhs.pb);   // point pb to a copy of rhs’s bitmap 
	delete pOrig;               // delete the original pb 
	return *this; 
}
```

3. Using the "copy-and-swap" technique makes the assignment exception-safe and self-assignment safe
```cpp
Widget& Widget::operator=(const Widget& rhs) 
{ 
    Widget temp(rhs);    // make a copy of rhs’s data 
    swap(temp);          // swap *this’s data with the copy’s 
    return *this; 
}
```

## Use the same form in corresponding uses of `new` and `delete`
If you use `[]` in a `new` expression, you must use `[]` in the corresponding `delete` expression. If you don’t use `[]` in a `new` expression, don’t use `[]` in the matching `delete` expression.
```cpp
std::string *stringPtr1 = new std::string; 
std::string *stringPtr2 = new std::string[100]; 
... 
delete stringPtr1;     // delete an object 
delete [] stringPtr2;  // delete an array of objects
```

## Store `new`ed objects in smart pointers in standalone statements
In a method call like this
```cpp
int priority(); 
void processWidget(std::shared_ptr pw, int priority);
...
processWidget(std::shared_ptr(new Widget), priority());
```

Compilers are given a lot of freedom in which order they want to execute the calls. If they decide to do the following:
- Execute `new Widget`
- Call `priority`
- Call the `std::shared_ptr` constructor
And calling `priority` throws an exception, then memory will leak because nobody deletes the newly created widget.

It's better to create the shared pointer before calling `processWidget`:
```cpp
std::tr1::shared_ptr pw(new Widget);
...
processWidget(pw, priority());
```