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

## Declare data members `private`
- Declare data members private. It gives clients syntactically uniform access to data, affords fine-grained access control, allows invariants to be enforced, and offers class authors implementation flexibility. 
- `protected` is no more encapsulated than public.

## Prefer non-member non-friend functions to member functions
Doing so increases encapsulation, packaging flexibility, and functional extensibility.
### Examples
Bad:
```cpp
class WebBrowser { 
public: 
	... 
	void clearCache(); 
	void clearHistory(); 
	void removeCookies();
	
	void clearEverything(); // BAD!
};
```

Good:
```cpp
void clearBrowser(WebBrowser& wb) 
{ 
	wb.clearCache(); 
	wb.clearHistory(); 
	wb.removeCookies(); 
}
```

## Declare non-member functions when type conversions should apply to all parameters
If you need type conversions on all parameters to a function (including the one that would otherwise be pointed to by the this pointer), the function must be a non-member.
### Examples
Bad:
```cpp
class Rational { 
public: 
	... 
	const Rational operator*(const Rational& rhs) const; 
};

result = oneHalf * 2; // fine 

result = 2 * oneHalf; // error!
```

Good:
```cpp
class Rational { 
	...     // contains no operator* 
}; 

const Rational operator*(const Rational& lhs, // now a non-member 
                         const Rational& rhs) // function 
{ 
	return Rational(lhs.numerator() * rhs.numerator(), 
	                lhs.denominator() * rhs.denominator()); 
} 

Rational oneFourth(1, 4); 
Rational result; 

result = oneFourth * 2; // fine 
result = 2 * oneFourth; // also fine now
```

## Postpone variable definitions as long as possible
It increases program clarity and improves program efficiency.

### Examples
Bad:
```cpp
std::string encryptPassword(const std::string& password) 
{ 
	using namespace std; 
	string encrypted; 
	
	if (password.length() < MinimumPasswordLength) { 
		throw logic_error("Password is too short"); 
	} 
	
	...
	return encrypted; 
}
```

Good:
```cpp
std::string encryptPassword(const std::string& password) 
{ 
	using namespace std; 
	if (password.length() < MinimumPasswordLength) { 
		throw logic_error("Password is too short"); 
	} 
	
	string encrypted; 
	... 
	return encrypted; 
}
```

## Minimize casting

Types of casting:
- C-style casts: `(T) expression` - old-school casts.
- `const_cast<T>(expression)` - used to cast away the constness of objects.
- `dynamic_cast<T>(expression)` - it performs "safe downcasting" i.e., determines whether an object is of a particular type in an inheritance hierarchy. It's also the only cast that may have significant runtime cost.
- `reinterpret_cast<T>(expression)`- used for low level casts that yield implementation-dependent results, e.g. casting a pointer to an int. It should be used rarely.
- `static_cast(expression)` - used to force implicit conversions (e.g. int to double or non-const to const object).

Using new casts is preferred because:
- They're much easier to find in the code.
- Since each cast has a specific purpose compilers may diagnose usage errors. For example, trying to cast away the constness using new-style cast other that `const_cast` will provide compiler errors.

## Avoid returning "handles" to object internals
Avoid returning handles (references, pointers, or iterators) to object internals. Not returning handles increases encapsulation, helps const member functions act const, and minimizes the creation of dangling handles.

In an example like this returning a reference from `boundingBox` will be undefined behavior because that reference is deleted after the call:
```cpp
GUIObject *pgo; 
const Point *pUpperLeft = &(boundingBox(*pgo).upperLeft());
```

## Understanding inline functions
Use inline only on very short functions where code generated for function body would be smaller than code generated for function call.

Two ways to declare inline functions are:
- Implicit:
```cpp
class Person { 
public: 
	... 
	int age() const { return theAge; } // an implicit inline request: age is 
	...                                // defined in a class definition 
private: 
	int theAge; 
};
```
- Explicit:
```cpp
template<typenameT>                              // an explicit inline 
inline const T& std::max(const T& a, const T& b) // request: std::max is 
{ return a < b ? b : a; }                        // preceded by “inline”
```

Constructors and destructors should never be `inline` because compilers generate code for those.

Some debuggers prevent from setting breakpoints in inline functions.

## Make sure public inheritance models "is-a"
Public inheritance means “is-a.” Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object.

Example of this claim:
```cpp
class Person { ... }; 
class Student: public Person { ... };
```

## Never redefine an inherited non-virtual function
Non-virtual functions are statically bound and virtual functions are dynamically bound so calling the same functions on different types of pointers, even if those pointers point to the same underlying object, will call different functions (when those functions are non-virtual and redefined).

```cpp
class B { 
public: 
	void mf(); 
	... 
};

class D: public B { 
public: 
	void mf(); // hides B::mf;
	... 
};

D x;
B *pB = &x;
D *pD = &x;

pB->mf(); // calls B::mf 
pD->mf(); // calls D::mf
```

## Never redefine a function's inherited default parameter value
Redefining an inherited function's default parameter leads to undefined behavior as inherited (virtual) functions are dynamically bound but the default parameters are statically bound so redefining it in derived classes can lead to strange behavior based on the kind of pointer we call the function from.

```cpp
class Shape { 
public: 
	enum ShapeColor { Red, Green, Blue }; 
	virtual void draw(ShapeColor color = Red) const = 0; 
};

class Rectangle: public Shape { 
public: 
	// Defined different default param - bad!
	virtual void draw(ShapeColor color = Green) const;
};

// Static type - Shape
// Dynamic type - Rectangle
Shape *pr = new Rectangle;

pr->draw(); // calls Rectangle::draw(Shape::Red)
```