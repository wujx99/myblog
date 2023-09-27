---
title: effective c++
date: 2023-09-05 15:26:18
tags:
- cpp
- effective c++
---
《effective c++》读书笔记
<!-- more -->
# 简介

## 术语
- declaration：
- `object`:包括内置类型和用户自定义类型
- signature：函数的参数和返回类型
- definition：提供编译器declaratin没有的实现细节；对于对象来说，定义就是编译器为对象保留内存。
- initialization：给对象第一次的值
- default constructor：不用提供参数就能调用，要么没有参数，要么参数有默认值
- copy constructor：参数传值很有用
- copy assignment operator：
- STL
- function object：像函数一样的对象
- undefined behavior：
- interface：
- client：
- ctor，dtor：

```cpp
extern int x   //declaration

int x  			// definition
```
## 命名习惯
pt意味着指向t对象的指针
rt意味着指向t对象的引用

## 对多线程的考虑

## TR1 and Boost

# chap01：accustoming yourself  to c++

## item1：view C++ as a federation of languages
c++是一系列的语言特性的集合，你最好在不同的使用方式下切换。识别出主要的子语言：
- c
- OO c++
- Template c++
- STL

举例来说，内置类型pass-by-value比pass-by-reference更高效，OO中pass-by-reference-to-const更有效，在TMP中你甚至不知道你的类型是什么。STL中更像是c，所以又回到pass-by-value。所以**没有统一的规则**，记住这四种子语言。

>**Things to Remember**
✦ Rules for effective C++ programming vary, depending on the part of C++ you are using

## Item2:perfer consts,enums,and inlines to #defines
其实应该说“相较于预处理器应该更喜欢编译器”

宏定义不会出现在符号表，且经过预处理器后会有多处替代，**不如**声明const常量！

有两点需要注意
1. const指针：因为常量一般放在头文件里面，所以不能被更改。
```cpp
//需要两个const保证指针的值和字符串不会改变
const char * const authorName = "Scott Meyers";

//一个更好的实践是
const std::string authorName("Scott Meyers"); 
```
2. 类依赖的常量

```cpp
class GamePlayer {
private:
	static const int NumTurns = 5; 	// constant declaration
	int scores[NumTurns]; 			// use of constant
	...
};
```
注意上面要声明`static`,上面的`NumTurns`是declaration，不是definition。

通常，c++要求您为使用的任何东西提供定义，但是类依赖的静态的和整型(例如，整数、字符、bool)的常量是**一个例外**。

如果不是整型的话可以这样：

```cpp
class CostEstimate {
private:
	static const double FudgeFactor;// declaration of static class
	... 							// constant; goes in header file
};

const double 						// definition of static class
CostEstimate::FudgeFactor = 1.35; 	// constant; goes in impl. file
```

老的编译器不支持声明时赋值，但是类的其他成员需要常量值，这时要使用"the enum hack"。

```cpp
class GamePlayer {
private:
	enum { NumTurns = 5 }; 	// “the enum hack” — makes
							// NumTurns a symbolic name for 5
	int scores[NumTurns]; 	// fine
	...
}
```
**enum**更像#define，不能获取其地址，也就能防止访问和修改。
>好的编译器不会为整型的const对象留出存储空间(除非创建指向该对象的指针或引用),为什么？

不要用宏替代函数，有更好的模板函数的实现。
>**Things to Remember**:
✦ For simple constants, prefer const objects or enums to #defines.
✦ For function-like macros, prefer inline functions to #defines.

## Item3：use const whenever possible.
stl的iterator使用指针建模的，分为`iterator`和`const_iterator`，对iterator的const限制是对指针的限制而不是对指针指向的值的限制：

```cpp
std::vector<int> vec;
...
const std::vector<int>::iterator iter = 	// iter acts like a T* const
vec.begin();
*iter = 10; 								// OK, changes what iter points to
++iter; 									// error! iter is const
std::vector<int>::const_iterator cIter = 	// cIter acts like a const T* vec.begin();
*cIter = 10; 								// error! *cIter is const
++cIter; 									// fine, changes cIter
```

const在函数上的使用就更多了。

return value声明成const的原因，为了不让返回的值作为左值：

```cpp
class Rational { ... };
const Rational operator*(const Rational& lhs, const Rational& rhs);
```
上面如果不声明const就会有下面的语句

```cpp
Rational a, b, c;
...
(a * b) = c; 	// invoke operator= on the
				// result of a*b!
```

**----const Member Functions**
一个成员函数可以有const和constness的重载：

```cpp
class TextBlock {
public:
	...
	const char& operator[](std::size_t position) const 	// operator[] for
	{ return text[position]; } 							// const objects
	char& operator[](std::size_t position) 				// operator[] for
	{ return text[position]; } 							// non-const objects
private:
 	std::string text;
};
```
上面的重载的**返回类型也不同**，这是合理的。

```cpp
TextBlock tb("Hello");
std::cout << tb[0]; 	// calls non-const
						// TextBlock::operator[]
const TextBlock ctb("World");
std::cout << ctb[0]; 	// calls const TextBlock::operator[]

std::cout << tb[0];	 	// fine — reading a
						// non-const TextBlock
tb[0] = ’x’; 			// fine — writing a
						// non-const TextBlock
std::cout << ctb[0]; 	// fine — reading a
						// const TextBlock
ctb[0] = ’x’; 			// error! — writing a
						// const TextBlock
```
注意上面`[]`返回的都是**引用**，而**不是**内置类型`char`，否则就不能被赋值（就算能赋值，也是传值的，并不是传地址）。
>是不是const类对象只能访问const成员函数吗？是对的，const对象的`this`指针加入了顶层const，只能传入到const成员函数里。

现在考虑一下成员函数设置成const的哲学，有两种观点：bitwise constness和logical constness。

bitwise constness**有漏洞**，当cosnt成员函数返回非常量的引用还是可以修改const类对象的值。

引入了logical constness的理念和mutable

```cpp
class CTextBlock {
public:
	...
	std::size_t length() const;
private:
	char *pText;
	mutable std::size_t textLength; // these data members may
	mutable bool lengthIsValid; 	// always be modified, even in
}; 									// const member functions

//在const成员函数内也修改了
std::size_t CTextBlock::length() const
{
if (!lengthIsValid) {
textLength = std::strlen(pText); 	// now fine
lengthIsValid = true; 				// also fine
}
return textLength;
}
```
**-----Avoiding Duplication in const and Non-const Member Functions**

mutable是一个避免bitwise-constness-is-not-what-i-had-in-mind解决方案，但是不能解决所有const-related的困难，例如：

```cpp
class TextBlock {
public:
...
const char& operator[](std::size_t position) const
{
	... // do bounds checking
	... // log access data
	... // verify data integrity
	return text[position];
}
char& operator[](std::size_t position)
{
	... // do bounds checking
	... // log access data
	... // verify data integrity
	return text[position];
}
private:
 	std::string text;
};
```
一般来说cast是一个不好的选择，但在这里却还不错：

```cpp
class TextBlock {
public:
	...
	const char& operator[](std::size_t position) const // same as before
	{
	...
	...
	...
		return text[position];
	}
	char& operator[](std::size_t position) 			// now just calls const op[]
	{
		return
			const_cast<char&>( 						// cast away const on
													// op[]’s return type;
			static_cast<const TextBlock&>(*this) 	// add const to *this’s type;
			[position] 								// call const version of op[]
			);
		}
		...
};
```
必须先cast成const对象，否则无限递归调用，
>item27介绍static-cast/const-cast

注意上面的反面的调用时不合适的，即用const成员函数调用nonconst的成员函数。

>**Things to Remember**:
✦ Declaring something const helps compilers detect usage errors. const
can be applied to objects at any scope, to function parameters and
return types, and to member functions as a whole.
✦ Compilers enforce bitwise constness, but you should program using
logical constness.
✦ When const and non-const member functions have essentially identical implementations, code duplication can be avoided by having the
non-const version call the const version.

## item4:make sure that object are initialized before they're used

c++的c部分可能由于运行时开销就不保证初始化，但是非c部分可能就不同了。

所以**内置类型**就手动初始化，**其他的类型**构造器初始每个类数据成员。

要注意初始化和赋值的区别：

```cpp
class PhoneNumber { ... };
class ABEntry { 				// ABEntry = “Address Book Entry”
public:
	ABEntry(const std::string& name, const std::string& address,
			const std::list<PhoneNumber>& phones);
private:
	std::string theName;
	std::string theAddress;
	std::list<PhoneNumber> thePhones;
	int numTimesConsulted;
};
ABEntry::ABEntry(const std::string& name, const std::string& address,
const std::list<PhoneNumber>& phones)
{
	theName = name; 			// these are all assignments,
	theAddress = address; 		// not initializations
	thePhones = phones;
	numTimesConsulted = 0;
}
```
最好改成（成员初始化列表内**拷贝初始化或默认初始化**），

```cpp
ABEntry::ABEntry(const std::string& name, const std::string& address,
const std::list<PhoneNumber>& phones)
: theName(name),
theAddress(address), 		// these are now all initializations
thePhones(phones),
numTimesConsulted(0)
{} 							// the ctor body is now empty
```

上面的**内置类型**`numTimesConsulted`放在ctor函数体内赋值还是在成员初始化列表内初始化都无所谓。

有时候成员初始化列表**是必须**的，比如const成员和引用不能被赋值（见item5）。

如果有多个构造器和很多成员，这时候可以声明一个函数进行赋值**也是**合理的。一般来说，真正的成员初始化(通过初始化列表)**优于**通过赋值进行的伪初始化。

成员初始化的顺序：

当一切都按上面的做好了后，只有一件事情需要担心：**在不同的translation units中定义的non-local static对象的初始化顺序**。

static object不同于stack和heap中的对象，lifetime是整个程序。**包含**：全局对象，namespace中的对象，类中声明static的对象，函数中声明static的对象，**文件域**中声明static的对象。
其中**函数中**的static对象是局部的（只对函数可见），其他是非local的。

>专题作用域和可见域1<https://www.cnblogs.com/cygalaxy/p/7103674.html>
2<http://17de.com/library/CPP/ls15.htm>

translation unit是单个object file的源代码，其中的`#include`file都加入其中了。

一个translation unit的static对象用另一个translation unit的static对象初始化会导致问题。如：

```cpp
class FileSystem { 					// from your library’s header file
public:
	...
	std::size_t numDisks() const; 	// one of many member functions ...
};
extern FileSystem tfs; 				// declare object for clients to use
									// (“tfs” = “the file system” ); definition
									// is in some .cpp file in your library
```
FileSystem对象显然是non-trivial的，因此在构造tfs对象之前使用它将是灾难性的。
比如用户如下使用：

```cpp
class Directory { 						// created by library client
public:
	Directory( params ); ...
};
Directory::Directory( params )
{
	...
	std::size_t disks = tfs.numDisks(); // use the tfs object ...
}
```
然后就可以这样使用：
`Directory tempDir( params ); // directory for temporary files`

但是**一定**要保证tfs在tempDir之前初始化。想要保证初始化的顺序是不可能的。

该怎么办呢？只能更改设计！（单例模式的一种实现）用local static对象代替：

```cpp
class FileSystem { ... }; 					// as before
FileSystem& tfs() 							// this replaces the tfs object; it could be
{ 											// static in the FileSystem class
	static FileSystem fs; 					// define and initialize a local static object
	return fs; 								// return a reference to it
}
class Directory { ... }; 					// as before
Directory::Directory( params ) 				// as before, except references to tfs are
{ 											// now to tfs()
	...
	std::size_t disks = tfs().numDisks();
	...
}
Directory& tempDir() 						// this replaces the tempDir object; it
{ 											// could be static in the Directory class
	static Directory td( params ); 			// define/initialize local static object
	return td; 								// return reference to it
}
```

上面的这种关于non-const的static对象的初始化在多线程程序中会出现问题。

>**Things to Remember**:
✦ Manually initialize objects of built-in type, because C++ only sometimes initializes them itself.
✦ In a constructor, prefer use of the member initialization list to assignment inside the body of the constructor. List data members in
the initialization list in the same order they’re declared in the class.
✦ Avoid initialization order problems across translation units by replacing non-local static objects with local static objects.

# chap02:ctor,dtor,and assignment operators

## item3：konw what functions c++ silently writes and calls.
当写下：
`class Empty{};`
等价于写下：

```cpp
class Empty {
public:
	Empty() { ... } 							// default constructor
	Empty(const Empty& rhs) { ... } 			// copy constructor
	~Empty() { ... } 							// destructor — see below
												// for whether it’s virtual
	Empty& operator=(const Empty& rhs) { ... } 	// copy assignment operator
};
```
注意生成的析构器是非虚的（除非从基类继承虚性质）

拷贝赋值运算符有一些限制，比如下面的引用和const成员。

```cpp
template<typename T>
class NamedObject {
public:
		// this ctor no longer takes a const name, because nameValue
		// is now a reference-to-non-const string. The char* constructor
		// is gone, because we must have a string to refer to.
	NamedObject(std::string& name, const T& value);
	... // as above, assume no
		// operator= is declared
private:
	std::string& nameValue; // this is now a reference
	const T objectValue; 	// this is now const
};
```
下面会发生什么：

```cpp
std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2); 	// when I originally wrote this, our
								// dog Persephone was about to
								// have her second birthday
NamedObject<int> s(oldDog, 36); // the family dog Satch (from my
								// childhood) would be 36 if she
								// were still alive
p = s; 							// what should happen to
								// the data members in p?
```
>c++不允许引用指向新的对象吗？

上面的代码编译器会拒绝编译！

>**Things to Remember**:
✦ Compilers may implicitly generate a class’s default constructor, copy constructor, copy assignment operator, and destructor.

## item6:explicitly disallow the use of compiler-generated functions you do not want.
`class HomeForSale { ... };`
每个house都是独一无二，拷贝没有什么意义！所以进行下面的操作限制拷贝（声明**不定义**且private）。

```cpp
class HomeForSale {
public:
...
private:
	...
	HomeForSale(const HomeForSale&); 		// declarations only
	HomeForSale& operator=(const HomeForSale&);
};
```
如果声明了友元或内部的调用上面的函数会在链接时候发生错误（没有定义函数）。可以把问题移动到编译期报错。

```cpp
class Uncopyable {
protected: 										// allow construction
	Uncopyable() {} 							// and destruction of
	~Uncopyable() {} 							// derived objects...
private:
	Uncopyable(const Uncopyable&); 				// ...but prevent copying
	Uncopyable& operator=(const Uncopyable&);
};

class HomeForSale: private Uncopyable { 		// class no longer
	... 										// declares copy ctor or
}; 												// copy assign. operator
```
>为什么要private继承？Uncopyable的析构器不必声明为虚？后面再回来看看

>**Things to Remember**:
✦ To disallow functionality automatically provided by compilers, declare the corresponding member functions private and give no implementations. Using a base class like Uncopyable is one way to do this.

## item7:declare dtors virtual in polymorphic base classes
*factory function*

如果不是为了作为基类**就不要**设置虚析构器，不这样的话就会引入虚机制破坏与其他语言的相容性和可移植性。
**summary**: declare a virtual destructor in a class if and only if that class contains at least one virtual function

不要继承没有virtual析构器的类。

>abstract类和纯虚函数的关系,纯虚析构器还**必须**要有定义！
>纯虚函数不能创建对象！

>**Things to Remember**:
✦ Polymorphic base classes should declare virtual destructors. If a class has any virtual functions, it should have a virtual destructor.
✦ Classes not designed to be base classes or not designed to be used polymorphically should not declare virtual destructors.

## item8：prevent exceptions from leaving dtor.

```cpp
class Widget {
public:
	...
	~Widget() { ... } 		// assume this might emit an exception
};
void doSomething()
{
	std::vector<Widget> v;
	...
} 							// v is automatically destroyed here
```
假设`v`中有10个`Widget`。如果在析构`v`的时候，第一个`widget`析构器报出exception，另一个也报出。**两个exception同时报出会出现问题**。

怎么解决这个问题呢？设计一个资源管理的类。如：

```cpp
class DBConnection {
public:
	...
	static DBConnection create(); 	// function to return
									// DBConnection objects; params
									// omitted for simplicity
	void close(); 					// close connection; throw an
}; 									// exception if closing fails
```
管理`DBConnection`对象的类：

```cpp
class DBConn { 			// class to manage DBConnection
public: 				// objects ...
	~DBConn() 			// make sure database connections
	{ 					// are always closed
		db.close();
	}
private:
	DBConnection db;
};
```
如下的使用：

```cpp
{ 										// open a block
	DBConn dbc(DBConnection::create()); // create DBConnection object
										// and turn it over to a DBConn
										// object to manage
	... 								// use the DBConnection object
										// via the DBConn interface
								} 		// at end of block, the DBConn
										// object is destroyed, thus
										// automatically calling close on
										// the DBConnection object
```
如果`close()`能被正确的调用上面的方案就可行，但如果产生exception的话就不好说了，有两种方案：
1. 终止程序：

```cpp
DBConn::~DBConn()
{
	try { db.close(); }
	catch (...) {
		make log entry that the call to close failed;
		std::abort();
	}
}
```
2.吞下异常：

```cpp
DBConn::~DBConn()
{
	try { db.close(); }
	catch (...) {
		make log entry that the call to close failed;
	}
}
```
有一个更好的策略（让用户能够react to出现的问题并去处理，下面设计了`close()`函数**给用户**使用）：

```cpp
class DBConn {
public:
	...
	void close() 											// new function for
	{ 														// client use
		db.close();
		closed = true;
	}
	~DBConn()
	{
		if (!closed) {
			try { 											// close the connection
				db.close(); 								// if the client didn’t
			}
			catch (...) { 									// if closing fails,
				make log entry that call to close failed; 	// note that and 
				... 										// terminate or swallow
			}
		}
	}
private:
	DBConnection db;
	bool closed;
};
```

>**Things to Remember**
✦ Destructors should never emit exceptions. If functions called in a destructor may throw, the destructor should catch any exceptions,
then swallow them or terminate the program.
✦ If class clients need to be able to react to exceptions thrown during an operation, the class should provide a regular (i.e., non-destructor) function that performs the operation.

## item9:never call virtual functions during ctor or dtor.
下面是股票交易的实例且有log

```cpp
class Transaction { 							// base class for all
public: 										// transactions
	Transaction();	
	virtual void logTransaction() const = 0; 	// make type-dependent
												// log entry
	...
};
Constructors, Destructors, operator= Item 9 49
Transaction::Transaction() 						// implementation of
{ 												// base class ctor
	...
	logTransaction(); 							// as final action, log this
} 												// transaction
class BuyTransaction: public Transaction { 		// derived class
public:
	virtual void logTransaction() const; 		// how to log trans-
												// actions of this type 
	...
};
class SellTransaction: public Transaction { 	// derived class
public:
	virtual void logTransaction() const; 		// how to log trans-
												// actions of this type
	...
};
```
当写下：
`BuyTransaction b;`
你能发现问题吗？（有了object model的知识理解这里就简单多了）

下面的代码也是同样的问题但是更加的隐蔽：

```cpp
class Transaction {
public:
	Transaction()
	{ init(); } 				// call to non-virtual...
	virtual void logTransaction() const = 0;
	...
private:
	void init()
	{
		...
		logTransaction(); 		// ...that calls a virtual!
	}
};
```

一个解决方案是非虚，派生类给其传递基类信息

```cpp
class Transaction {
public:
	explicit Transaction(const std::string& logInfo);
	void logTransaction(const std::string& logInfo) const;	// now a non-
															// virtual func
	...
};
Transaction::Transaction(const std::string& logInfo)
{
	...
	logTransaction(logInfo); 								// now a non-
} 															// virtual call
class BuyTransaction: public Transaction {
public:
	BuyTransaction( parameters )
	: Transaction(createLogString( parameters )) 			// pass log info
	{ ... } 												// to base class
	... 													// constructor
private:
	static std::string createLogString( parameters );
};
```
注意上面的函数`creatLogString()`是static的（**很重要**！）。这样不会使用到派生类未初始化的成员！

>**Things to Remember**:
✦ Don’t call virtual functions during construction or destruction, because such calls will never go to a more derived class than that of the currently executing constructor or destructor.

## item10:have assignment operators return a reference to *this

```cpp
int x, y, z;
x = y = z = 15; // chain of assignments
```
因为赋值是右结合的：
`x = (y = (z = 15));`
遵循惯例：

```cpp
class Widget {
public:
...
Widget& operator=(const Widget& rhs)	// return type is a reference to
{ 										// the current class
	...
	return *this; 						// return the left-hand object
}
...
Widget& operator+=(const Widget& rhs) 	// the convention applies to
{ 										// +=, -=, *=, etc.
	...
	return *this;
}
Widget& operator=(int rhs) 				// it applies even if the
	{ 									// operator’s parameter type ... 	
										// is unconventional
	return *this;
}
	...
};
```

>**Things to Remember**
✦ Have assignment operators return a reference to `*this`.

## item11: handle assignment to self in operator=

别名的存在可能是你在使用之前释放了内存。
下面的例子：

```cpp
class Bitmap { ... };
class Widget {
	...
	private:
	Bitmap *pb; 						// ptr to a heap-allocated object
};

Widget&
Widget::operator=(const Widget& rhs) 	// unsafe impl. of operator=
{
	delete pb; 							// stop using current bitmap
	pb = new Bitmap(*rhs.pb); 			// start using a copy of rhs’s bitmap
	return *this; 						// see Item 10
}
```
如果`=`两个的对象相同，上面会有什么问题？
传统的解决方案是：

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
但是`operator=`不仅仅是self-assignment-unsafe,而且是exception-unsafe。如果`new Bitmap`出现异常，那么pb指向了被删除的bitmap。

下面是一种实现：

```cpp
Widget& Widget::operator=(const Widget& rhs)
{
	Bitmap *pOrig = pb; 		// remember original pb
	pb = new Bitmap(*rhs.pb); 	// point pb to a copy of rhs’s bitmap
	delete pOrig; 				// delete the original pb
	return *this;
}
```
另一种方案是使用“copy and swap”：

```cpp
class Widget {
	...
	void swap(Widget& rhs); 				// exchange *this’s and rhs’s data; ... 	
											// see Item 29 for details
};
Widget& Widget::operator=(const Widget& rhs)
{
	Widget temp(rhs); 						// make a copy of rhs’s data
	swap(temp); 							// swap *this’s data with the copy’s
	return *this;
}
```
需要item29的知识`swap()`。还有变种：

```cpp
Widget& Widget::operator=(Widget rhs) 	// rhs is a copy of the object
{ 										// passed in — note pass by val
	swap(rhs); 							// swap *this’s data with
										// the copy’s
	return *this;
}
```
利用了两个特点：1拷贝赋值操作符参数可以不是引用；2利用参数传递的拷贝构造。

>**Things to Remember**:
✦ Make sure operator= is well-behaved when an object is assigned to itself. Techniques include comparing addresses of source and target objects, careful statement ordering, and copy-and-swap.
✦ Make sure that any function operating on more than one object behaves correctly if two or more of the objects are the same.

## item12:copy all part of an object
*copy function*：
```cpp
void logCall(const std::string& funcName); 			// make a log entry
class Customer {
public:
	...
	Customer(const Customer& rhs);
	Customer& operator=(const Customer& rhs);
	...
private:
	std::string name;
};

Customer::Customer(const Customer& rhs)
: name(rhs.name) 									// copy rhs’s data
{
	logCall("Customer copy constructor");
}
Customer& Customer::operator=(const Customer& rhs)
{
	logCall("Customer copy assignment operator");
	name = rhs.name; 								// copy rhs’s data
	return *this; 									// see Item 10
}
```
如果添加另一个成员：

```cpp
class Date { ... }; 	// for dates in time
class Customer {
public:
	... 				// as before
private:
	std::string name;
	Date lastTransaction;
};
```
原来的拷贝函数就是部分拷贝了。
下面的问题隐藏的更深：

```cpp
class PriorityCustomer: public Customer { 			// a derived class
public:
	...
	PriorityCustomer(const PriorityCustomer& rhs);
	PriorityCustomer& operator=(const PriorityCustomer& rhs);
	...
private:
	int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: priority(rhs.priority)
{
	logCall("PriorityCustomer copy constructor");
}
PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
	logCall("PriorityCustomer copy assignment operator");
	priority = rhs.priority;
	return *this;
}
```
上面没有拷贝基类的成员（会使用无参默认构造）：

```cpp
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: Customer(rhs), 					// invoke base class copy ctor
priority(rhs.priority)
{
	logCall("PriorityCustomer copy constructor");
}
PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
	logCall("PriorityCustomer copy assignment operator");
	Customer::operator=(rhs); 		// assign base class parts
	priority = rhs.priority;
	return *this;
}
```
>**Things to Remember**:
✦ Copying functions should be sure to copy all of an object’s data members and all of its base class parts.
✦ Don’t try to implement one of the copying functions in terms of the other. Instead, put common functionality in a third function that
both call.

# chap03: resource management
什么是resource？做完事后还给系统的资源！在c++中经常是动态分配内存！还有互斥锁，文件描述符，GUI的fonts/brushes，数据库连接，网络socket等。手动管理很不现实！

## item13：use objects to manage resources.
投资建模：

```cpp
class Investment { ... }; 	// root class of hierarchy of
							// investment types
```
通过item7的*factory function*在库中提供`investment`对象。

```cpp
Investment* createInvestment(); // return ptr to dynamically allocated
								// object in the Investment hierarchy;
								// the caller must delete it
								// (parameters omitted for simplicity)
```
如果：

```cpp
void f()
{
	Investment *pInv = createInvestment(); 	// call factory function
	... 									// use pInv
	delete pInv; 							// release object
}
```
上面的函数可能不能安全的释放资源。
下面使用`auto_ptr`来解决这个问题：

```cpp
void f()
{
	std::auto_ptr<Investment> pInv(createInvestment()); // call factory
														// function
	... 												// use pInv as
														// before
} 														// automatically
														// delete pInv via
														// auto_ptr’s dtor
```
使用对象管理资源的两个至关重要的方面：
- Resources are acquired and immediately turned over to resource-managing objects.
- Resource-managing objects use their destructors to ensure that resources are released（注意析构器中异常处理，见item8）

不能两个`auto_ptr`指向同一个对象，所以拷贝（拷贝构造和拷贝赋值操作符）的语义很特别：

```cpp
std::auto_ptr<Investment> 				// pInv1 points to the
pInv1(createInvestment()); 				// object returned from
										// createInvestment
std::auto_ptr<Investment> pInv2(pInv1); // pInv2 now points to the
										// object; pInv1 is now null
pInv1 = pInv2; 							// now pInv1 points to the
										// object, and pInv2 is null
```
但是上面的语义也说明这不是一个很好的方式(**不能用于STL容器**）！

另一个选择是*reference-counting smart pointer*(RCSP),很像垃圾回收，但是不能打破循环引用！
TR1的`tr1::shared_ptr`(见item54)是一个RCSP:

```cpp
void f()
{
...
std::tr1::shared_ptr<Investment> 	// pInv1 points to the
		pInv1(createInvestment()); 	// object returned from
									// createInvestment
std::tr1::shared_ptr<Investment> 	// both pInv1 and pInv2 now
pInv2(pInv1); 						// point to the object
pInv1 = pInv2; 						// ditto — nothing has
									// changed ...
} 									// pInv1 and pInv2 are
									// destroyed, and the
									// object they point to is
									// automatically deleted
```
auto_ptr和tr1::shared_ptr都在析构函数中使用delete，**而不是**delete[]。不能动态分配数组，**那该怎么办呢？**，使用`vector`和`string`。

如果要定制自己的类需要注意item14和item15

在item18中看如何修改` createInvestment`接口！

>**Things to Remember**:
✦ To prevent resource leaks, use `RAII` objects that acquire resources in their constructors and release them in their destructors.
✦ Two commonly useful `RAII` classes are tr1::shared_ptr and auto_ptr. tr1::shared_ptr is usually the better choice, because its behavior when copied is intuitive. Copying an auto_ptr sets it to null.

## item14: think carefully about copying behavior in resource-managing classes.

上面的两个指针对象是管理heap中资源的，有时候你可能要设计自己的资源管理类。
如互斥对象：

```cpp
void lock(Mutex *pm); 		// lock mutex pointed to by pm
void unlock(Mutex *pm); 	// unlock the mutex
```
设计类去管理：

```cpp
class Lock {
public:
	explicit Lock(Mutex *pm)
	: mutexPtr(pm)
	{ lock(mutexPtr); } 			// acquire resource
	~Lock() { unlock(mutexPtr); } 	// release resource
private:
	Mutex *mutexPtr;
};
```
用户可以如下使用：

```cpp
Mutex m; 		// define the mutex you need to use
...
{ 				// create block to define critical section
Lock ml(&m); 	// lock the mutex
... 			// perform critical section operations
} 				// automatically unlock mutex at end
				// of block 
```
但是如果发生**拷贝行为**该怎么办？

```cpp
Lock ml1(&m); 	// lock m
Lock ml2(ml1); 	// copy ml1 to ml2 — what should
				// happen here?
```
你有**多种选择**：
1. prohibit copying。详见item6

```cpp
class Lock: private Uncopyable { 	// prohibit copying — see
public: 							// Item 6
	... 							// as before
};
```
在本例中**适用**

2. reference-count the underlying resource。你可以内嵌`tr1::shared_ptr<Mutex>`，但是锁需要的是**释放**而不是删除。所以我们要更改默认的语义，定制`deleter`。

```cpp
class Lock {
public:
	explicit Lock(Mutex *pm) 				// init shared_ptr with the Mutex
	: mutexPtr(pm, unlock) 					// to point to and the unlock func
	{ 										// as the deleter†
		lock(mutexPtr.get()); 				// see Item 15 for info on “get”
	}
private:
	std::tr1::shared_ptr<Mutex> mutexPtr; 	// use shared_ptr
}; 											// instead of raw pointer
```
>`lock(mutexPtr.get())`的`get()`是什么？见item15

不必声明析构器！

3.  copy the underlying resource.
4. transfer ownership of the underlying resource.`auto_ptr`是一个例子。

>**Things to Remember**:
✦ Copying an RAII object entails copying the resource it manages, so the copying behavior of the resource determines the copying behavior of the RAII object.
✦ Common RAII class copying behaviors are disallowing copying and performing reference counting, but other behaviors are possible.

## item15:provide access to raw resource in resource-managing classes.
有时候必须要使用一些老的API。如：

```cpp
std::tr1::shared_ptr<Investment> pInv(createInvestment()); // from Item 13
```
假设你有函数：

```cpp
int daysHeld(const Investment *pi); // return number of days
									// investment has been held
```
你如果如下使用就会报错：

```cpp
int days = daysHeld(pInv); // error!
```
你有两种办法：显示转换和隐式转换
- **显式**：

```cpp
int days = daysHeld(pInv.get()); // fine, passes the raw pointer
// in pInv to daysHeld
```
- **隐式**通过重载的`->`和`*`：

```cpp
class Investment { 									// root class for a hierarchy
public: 											// of investment types
	bool isTaxFree() const;
	...
};
Investment* createInvestment(); 					// factory function

std::tr1::shared_ptr<Investment> 					// have tr1::shared_ptr
	pi1(createInvestment()); 						// manage a resource
	
bool taxable1 = !(pi1->isTaxFree()); 				// access resource
													// via operator->
...
std::auto_ptr<Investment> pi2(createInvestment()); 	// have auto_ptr
													// manage a
													// resource
bool taxable2 = !((*pi2).isTaxFree()); 				// access resource
													// via operator* ...
```

有时候你可能有一堆C API接口，如果你**要使用这些接口**必须使用`get()`这样的函数，这可能会很烦！比如：

```cpp
FontHandle getFont(); 				// from C API — params omitted
									// for simplicity
void releaseFont(FontHandle fh); 	// from the same C API

class Font { 						// RAII class
public:
	explicit Font(FontHandle fh) 	// acquire resource;
	: f(fh) 						// use pass-by-value, because the
	{} 								// C API does
	~Font() { releaseFont(f ); } 	// release resource
	... 							// handle copying (see Item 14)
private:
	FontHandle f; 					// the raw font resource
};
```
假设有一个大型的与字体相关的C API完全处理
`FontHandles`。`Font`类提供显式转换：

```cpp
class Font {
public:
	...
	FontHandle get() const { return f; } // explicit conversion function
	...
};
```
你需要这样使用`get()`：

```cpp
void changeFontSize(FontHandle f, int newSize); // from the C API
Font f(getFont());
int newFontSize;
...
changeFontSize(f.get(), newFontSize); 			// explicitly convert
												// Font to FontHandle
```
可以提供隐式的转换函数：

```cpp
class Font {
public:
	...
	operator FontHandle() const // implicit conversion function
	{ return f; }
	...
};
```
就可以像下面这样**自然**的使用这些C API了：

```cpp
Font f(getFont());
int newFontSize;
...
changeFontSize(f, newFontSize); // implicitly convert Font
								// to FontHandle
```
但是也是有风险的：

```cpp
Font f1(getFont());
...
FontHandle f2 = f1; 	// oops! meant to copy a Font
						// object, but instead implicitly
						// converted f1 into its underlying
						// FontHandle, then copied that
```
上面的f1如果销毁，那么f2就是**空悬指针**！所以还是推荐用`get()`

>Things to Remember
✦ APIs often require access to raw resources, so each RAII class should offer a way to get at the resource it manages.
✦ Access may be via explicit conversion or implicit conversion. In general, explicit conversion is safer, but implicit conversion is more convenient for clients

## item16:use the same form in corresponding uses of new and delete.
使用new和delete的时候分别发生两件事！所以一个问题是delete的时候删除多少个对象！回答是有多少析构器被调用！（内置类型怎么办？）

注意`typedef`:

```cpp
typedef std::string AddressLines[4]; 		// a person’s address has 4 lines,
											// each of which is a string
std::string *pal = new AddressLines;// note that “new AddressLines”
									// returns a string*, just like
									// “new string[4]” would	
delete pal; 	// undefined!
delete [] pal; 	// fine							
```
所以尽量不要对数组类型使用typedef！使用`vector`等STL

>**Things to Remember**
✦ If you use [] in a new expression, you must use [] in the corresponding delete expression. If you don’t use [] in a new expression, you
mustn’t use [] in the corresponding delete expression.


## item17:store newed objects in smart pointers in standalone statements.
根据优先级处理Widget：
```cpp
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
```
假设：

```cpp
processWidget(new Widget, priority());
```
上面的不会被编译！因为构造器是`explicit`！

```cpp
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```
但是上面的调用**会导致内存泄漏**！

假如按下列的顺序执行：
1. Execute “new Widget”.
2. Call priority.
3. Call the tr1::shared_ptr constructor.

如果调用`priority()`产生**异常**，那么资源就不会被分配到对象上。

>函数传参的语义是什么？理解上下两个方式的在传参上的区别！


```cpp
std::tr1::shared_ptr<Widget> pw(new Widget); 	// store newed object
												// in a smart pointer in a
												// standalone statement
processWidget(pw, priority()); 					// this call won’t leak
```

>Things to Remember
✦ Store newed objects in smart pointers in standalone statements.Failure to do this can lead to subtle resource leaks when exceptions
are thrown.

# chap04:Designs and Declarations
设计和声明好的c++接口！
**Guideline**：they should be easy to use correctly and hard to use incorrectly. 
依据上面的指导，我们讨论一系列话题： 
correctness,efficiency,encapsulation,maintainability, extensibility, and conformance to convention.

## item18：make interfaces easy to use correctly and hard to use incorrectly.
函数接口，类接口，模板接口；
假设设计：

```cpp
class Date {
public:
	Date(int month, int day, int year);
	...
};
```
但上面的顺序很难让人把握：

```cpp
struct Day { 			struct Month { 				struct Year {
explicit Day(int d) 	explicit Month(int m) 		explicit Year(int y)
: val(d) {} 			: val(m) {} 				: val(y){}
int val; 				int val; 					int val;
}; 						}; 							};

class Date {
public:
Date(const Month& m, const Day& d, const Year& y);
...
};
Date d(30, 3, 1995); 						// error! wrong types
Date d(Day(30), Month(3), Year(1995)); 		// error! wrong types
Date d(Month(3), Day(30), Year(1995)); 		// okay, types are correct
```
每年只有12个月，可以使用`enum`,但是这里不推荐使用！
>为什么不能使用`enum`？

可以使用下面的技巧：

```cpp
class Month {
public:
	static Month Jan() { return Month(1); } 	// functions returning all valid
	static Month Feb() { return Month(2); } 	// Month values; see below for
	... 										// why these are functions, not
	static Month Dec() { return Month(12); } 	// objects
	... 										// other member functions
private:
	explicit Month(int m); 						// prevent creation of new
												// Month values
	... 										// month-specific data
};

Date d(Month::Mar(), Day(30), Year(1995));
```
>为什么使用函数而不是用对象！见item4

加入const会避免手残，比如`*`的返回类型：

```cpp
if (a * b = c) ... // oops, meant to do a comparison!
```

>unless there’s a good reason not to, have your types behave consistently with the built-in types

size，count，length怎么选？

不要让客户记得做一些事情，如：

```cpp
Investment* createInvestment(); // from Item 13; parameters omitted
								// for simplicity
```
应该改成：

```cpp
std::tr1::shared_ptr<Investment> createInvestment();
```

如果要设置特定的deleter的话：

```cpp
std::tr1::shared_ptr<Investment> createInvestment()
{
	std::tr1::shared_ptr<Investment> retVal(static_cast<Investment*>(0),
	getRidOfInvestment);
	... 					// make retVal point to the
							// correct object
	return retVal;
}
```
>`static_cast`详见item27，最好不要像上面这样先定义地址值为0的指针再赋值正确的指针值。见item26

上面的deteter的定制可以避免cross-DLL问题。

>**Things to Remember**
✦ Good interfaces are easy to use correctly and hard to use incorrectly. You should strive for these characteristics in all your interfaces.
✦ Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.
✦ Ways to prevent errors include creating new types, restricting operations on types, constraining object values, and eliminating client resource management responsibilities.
✦ tr1::shared_ptr supports custom deleters. This prevents the crossDLL problem, can be used to automatically unlock mutexes (see Item 14), etc.

## item19:treat class design as type design
- How should objects of your new type be created and destroyed?
- How should object initialization differ from object assignment?
- What does it mean for objects of your new type to be passed by value?
- What are the restrictions on legal values for your new type?
- Does your new type fit into an inheritance graph?
- What kind of type conversions are allowed for your new type?
- What operators and functions make sense for the new type?
- What standard functions should be disallowed?
- Who should have access to the members of your new type?
- What is the “undeclared interface” of your new type?
- How general is your new type?
- Is a new type really what you need?
>**Things to Remember**
✦ Class design is type design. Before defining a new type, be sure to consider all the issues discussed in this Item.

## item20:perfer pass-by-reference-to-const to pass-by-value.
函数传参和返回通过copy constructor实现pass-by-value的，所以效率低。如：

```cpp
class Person {
public:
	Person(); 					// parameters omitted for simplicity
	virtual ~Person(); 			// see Item 7 for why this is virtual
	...
private:
	std::string name;
	std::string address;
};
class Student: public Person {
public:
	Student(); 					// parameters again omitted
	virtual ~Student();
	...
private:
	std::string schoolName;
	std::string schoolAddress;
};
```

```cpp
bool validateStudent(Student s); 		// function taking a Student
										// by value
Student plato; 							// Plato studied under Socrates
bool platoIsOK = validateStudent(plato);// call the function
```
如果如下声明就会避免这些问题：

```cpp
bool validateStudent(const Student& s);
```
传引用可以避免*slicing problem*:即把派生类按值传递给基类。如：

```cpp
class Window {
public:
	...
	std::string name() const; 		// return name of window
	virtual void display() const; 	// draw window and contents
};
class WindowWithScrollBars: public Window {
public:
	...
	virtual void display() const;
};
```
如果你想多态的使用`display()`，那么下面的函数是错误的：

```cpp
void printNameAndDisplay(Window w) 	// incorrect! parameter
{ 									// may be sliced!
std::cout << w.name();
w.display();
}
```
应该如下：

```cpp
void printNameAndDisplay(const Window& w) 	// fine, parameter won’t
{ 											// be sliced
std::cout << w.name();
w.display();
}
```

如果是**build-in类型**还是pass-by-value更加的高效，因为引用就是用指针实现的！**STL**中的迭代器和函数对象也是pass-by-value设计的！

>**Things to Remember**
✦ Prefer pass-by-reference-to-const over pass-by-value. It’s typically more efficient and it avoids the slicing problem.
✦ The rule doesn’t apply to built-in types and STL iterator and function object types. For them, pass-by-value is usually appropriate.

## item21:don't try to return a reference when you must return an object.
函数只能在stack和heap上创建新的对象，返回他们的引用会造成undefined behavior和内存泄漏。

天才的你可能使用下面的方案，返回一个`static`的对象：

```cpp
const Rational& operator*(const Rational& lhs, 						// warning! yet more
const Rational& rhs) 		// bad code!
{
	static Rational result; // static object to which a
							// reference will be returned
	result = ... ; 			// multiply lhs by rhs and put the
							// product inside result
	return result;
}
```
但是除了**线程安全**的问题，但是会有**更深**的缺点：

```cpp
bool operator==(const Rational& lhs, 	// an operator==
				const Rational& rhs); 	// for Rationals
Rational a, b, c, d;
...
if ((a * b) == (c * d)) {
	do whatever’s appropriate when the products are equal;
} else {
	do whatever’s appropriate when they’re not;
}
```

>**Things to Remember**
✦ Never return a pointer or reference to a local stack object, a reference to a heap-allocated object, or a pointer or reference to a local static object if there is a chance that more than one such object will be needed. (Item 4 provides an example of a design where returning a reference to a local static is reasonable, at least in single-threaded environments.)

## item22:declare data members private.
通过函数设置多种访问权限：

```cpp
class AccessLevels {
public:
	...
	int getReadOnly() const 		{ return readOnly; }
	void setReadWrite(int value) 	{ readWrite = value; }
	int getReadWrite() const 		{ return readWrite; }
	void setWriteOnly(int value) 	{ writeOnly = value; }
private:
	int noAccess; 	// no access to this int
	int readOnly; 	// read-only access to this int
	int readWrite; 	// read-write access to this int
	int writeOnly; 	// write-only access to this int
};
```
还可以确保*encapsulation*，通过函数来进行不同的实现！不然很难进行不同的实现。public的东西后面很难去改！

从封装的观点来看： there are really only two access levels: private (which offers encapsulation) and everything else (which doesn’t).

>**Things to Remember**
✦ Declare data members private. It gives clients syntactically uniform access to data, affords fine-grained access control, allows invariants to be enforced, and offers class authors implementation flexibility.
✦ protected is no more encapsulated than public.

## item23:prefer non-member non-friend functions to member functions.
假设web浏览器提供了三种清除功能：

```cpp
class WebBrowser {
public:
	...
	void clearCache();
	void clearHistory();
	void removeCookies();
	...
};
```
如果想全部清除，下面的实现哪个好？
**成员函数**：

```cpp
class WebBrowser {
public:
	...
	void clearEverything(); // calls clearCache, clearHistory,
							// and removeCookies
	...
};
```
**非成员函数**：

```cpp
void clearBrowser(WebBrowser& wb)
{
	wb.clearCache();
	wb.clearHistory();
	wb.removeCookies();
}
```
封装就是让更少的能够接触数据，如果成员函数和非成员非友元函数的功能一致，选非成员非友元的！


需要注意的是非成员函数并不代表不能是其他类的成员！（有些语言所有的函数都要放在类里，如java，c#），我们可以把`clearBrowser()`放到一些utility类并声明成static，这样就可以**不创建**类实例的使用！

在c++中有更加合适的办法，使用`namespace`:

```cpp
namespace WebBrowserStuff {
	class WebBrowser { ... };
	void clearBrowser(WebBrowser& wb);
	...
}
```

namespaces可以跨多个文件，所以可以拆分*convenient function*到多个文件，你可以根据兴趣包含对应的头文件：

```cpp
// header “webbrowser.h” — header for class WebBrowser itself
// as well as “core” WebBrowser-related functionality
namespace WebBrowserStuff {
class WebBrowser { ... };
	... 								// “core” related functionality, e.g.
										// non-member functions almost
										// all clients need
}
// header “webbrowserbookmarks.h”
namespace WebBrowserStuff {
	... 								// bookmark-related convenience
} 										// functions
// header “webbrowsercookies.h”
namespace WebBrowserStuff {
	... 								// cookie-related convenience
} 										// functions
...
```
上面的也是标准库的组织方式！这也意味着拓展性！

>**Things to Remember**
✦ Prefer non-member non-friend functions to member functions. Doing so increases encapsulation, packaging flexibility, and functional extensibility

## item24:declare non-member functions when type conversions should apply to all parameters.
虽然隐式的类型转换是坏主意，但是也有例外，比如int到有理数的转换：

```cpp
class Rational {
public:
	Rational(int numerator = 0, // ctor is deliberately not explicit;
	int denominator = 1); 		// allows implicit int-to-Rational
								// conversions
	int numerator() const; 		// accessors for numerator and
	int denominator() const; 	// denominator — see Item 22
private:
	...
};
```
如果把`operator *`声明成成员函数如何？

```cpp
class Rational {
public:
	...
	const Rational operator*(const Rational& rhs) const;
};
```
如果如下使用：

```cpp
Rational oneEighth(1, 8);
Rational oneHalf(1, 2);
Rational result = oneHalf * oneEighth; 	// fine
result = result * oneEighth; 			// fine
```
但是你想混合运算就会出错：

```cpp
result = oneHalf * 2; // fine
result = 2 * oneHalf; // error!
```
>第一个会出现隐式的转换！所以前面声明构造器的时候没有写explicit.


只有在参数列表的时候才会发生隐式的转换！所以上面的第二个调用发生了error。可以实现下面的：

```cpp
class Rational {
	... 										// contains no operator*
};
const Rational operator*(const Rational& lhs, 	// now a non-member
const Rational& rhs) 							// function
{
	return Rational(lhs.numerator() * rhs.numerator(),
	lhs.denominator() * rhs.denominator());
}
Rational oneFourth(1, 4);
Rational result;
result = oneFourth * 2; 						// fine
result = 2 * oneFourth; 						// hooray, it works!
```

还有一个问题，`operator*`是否声明为友元？在这个例子中不需要，因为public接口就能完成任务。

>成员函数的对立面是非成员函数，不是friend function。就像数据成员分成private和non-private。

>OO转变成模板设计的Rational见item46

>**Things to Remember**
✦ If you need type conversions on all parameters to a function (including the one that would otherwise be pointed to by the this pointer), the function must be a non-member. 

## item25:consider support for a non-throwing swap.

`swap()`是exception-safe编程的支柱！
经典实现：

```cpp
namespace std {
	template<typename T> 	// typical implementation of std::swap;
	void swap(T& a, T& b) 	// swaps a’s and b’s values
	{
		T temp(a);
		a = b;
		b = temp;
	}
}
```
看起来没什么了不起的，但是如果一些类型主要包含指针去指向真实的data，（常见于“pimpl idiom"即pointer to implementation)。例如：

```cpp
class WidgetImpl { 			// class for Widget data;
public: 					// details are unimportant
	...
private:
	int a, b, c; 			// possibly lots of data —
	std::vector<double> v; 	// expensive to copy!
...
};
```
使用pimpl idiom：

```cpp
class Widget { 							// class using the pimpl idiom
public:
	Widget(const Widget& rhs);
	Widget& operator=(const Widget& rhs)// to copy a Widget, copy its
	{ 									// WidgetImpl object. For 
		... 							// details on implementing
		*pImpl = *(rhs.pImpl); 			// operator= in general, 
		... 							// see Items 10, 11, and 12. 
	}
	...
private:
	WidgetImpl *pImpl; 					// ptr to object with this
}; 										// Widget’s data
```
如果要高效的实现交换只需要交换指针就行：

```cpp
namespace std {
	template<> 						// this is a specialized version
	void swap<Widget>(Widget& a, 	// of std::swap for when T is
	Widget& b) 						// Widget
	{
		swap(a.pImpl, b.pImpl); 	// to swap Widgets, swap their
	} 								// pImpl pointers; this won’t compile
}
```
但是上面的不会编译，因为指针是private的，可以声明friend，但是更加传统的实现是：

```cpp
class Widget { 						// same as above, except for the
public: 							// addition of the swap mem func
	...
	void swap(Widget& other)
	{
		using std::swap; 			// the need for this declaration
									// is explained later in this Item
		swap(pImpl, other.pImpl); 	// to swap Widgets, swap their
	} 								// pImpl pointers
	...
};
namespace std {
	template<> 						// revised specialization of
	void swap<Widget>(Widget& a, 	// std::swap
	Widget& b)
	{
		a.swap(b); 					// to swap Widgets, call their
	} 								// swap member function
}
```
如果都是`Widget`和`Widgetimpl`都是**模板类**如：

```cpp
template<typename T>
class WidgetImpl { ... };
template<typename T>
class Widget { ... };
```
你可能写下下面illegal的语句：

```cpp
namespace std {
	template<typename T>
	void swap<Widget<T> >(Widget<T>& a, // error! illegal code!
	Widget<T>& b)
	{ a.swap(b); }
}
```
>为什么上面有错误？为什么函数模板不支持部分特化，而类模板支持？

通常的做法是重载`swap()`：

```cpp
namespace std {
	template<typename T> 	// an overloading of std::swap
	void swap(Widget<T>& a, // (note the lack of “<...>” after
	Widget<T>& b) 			// “swap”), but see below for
	{ a.swap(b); } 			// why this isn’t valid code
}
```
虽然理论上是可以的，但是`std`是一个**特殊的**namespace，完全专门化模板是可以的，但是不允许新加模板，类，函数等等！

有了一个全新的解决方案（非成员的解决方案）：

```cpp
namespace WidgetStuff {
	... 					// templatized WidgetImpl, etc.
	template<typename T> 	// as before, including the swap
	class Widget { ... }; 	// member function
	...
	template<typename T> 	// non-member swap function;
	void swap(Widget<T>& a, // not part of the std namespace
	Widget<T>& b)
	{
		a.swap(b);
	}
}
```

在使用的时候不要指定`swap()`的哪个实例，而是如下的让编译器自动寻找最匹配的并把`std`**做保底**。

```cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
	using std::swap; 	// make std::swap available in this function
	...
	swap(obj1, obj2); 	// call the best swap for objects of type T
	...
}
```

最后永远不要让`swap()`抛出异常,针对的是**成员版本**。item29会说明为什么。

>**Things to Remember**
✦ Provide a swap member function when std::swap would be inefficient for your type. Make sure your swap doesn’t throw exceptions.
✦ If you offer a member swap, also offer a non-member swap that calls the member. For classes (not templates), specialize std::swap, too.
✦ When calling swap, employ a using declaration for std::swap, then call swap without namespace qualification.
✦ It’s fine to totally specialize std templates for user-defined types, but never try to add something completely new to std.


# chap05：Implementations

## item26：postpone variable definitions as long as possible.
ctor/dtor的使用使得未使用的类型也会有损耗。有时候无法避免：

尽量使用拷贝构造！

循环方式A，析构次数少，但是B的可维护性，可理解性更高，应该**默认选B**：

```cpp
// Approach A: define outside loop 		// Approach B: define inside loop
Widget w;
for (int i = 0; i < n; ++i) { 			for (int i = 0; i < n; ++i) {
	w = some value dependent on i; 			Widget w(some value dependent on i);
... 									...
} 										}
```
>**Things to Remember**
✦ Postpone variable definitions as long as possible. It increases program clarity and improves program efficiency.

## item27:minimize casting
有三种形式的cast：
前两种是old-style风格：c-style和function-style
```cpp
(T) expression // cast expression to be of type T
T(expression) // cast expression to be of type T
```
还有new-style或者c++-style casts:

```cpp
const_cast<T>(expression)
dynamic_cast<T>(expression)
reinterpret_cast<T>(expression)
static_cast<T>(expression)
```
只在传递给构造器才使用old-style了不用new-style：

```cpp
class Widget {
public:
	explicit Widget(int size);
	...
};
void doSomeWork(const Widget& w);
doSomeWork(Widget(15)); 			// create Widget from int
									// with function-style cast
doSomeWork(static_cast<Widget>(15));// create Widget from int
									// with C++-style cast
```
下面是常见的错误

```cpp
class Window { 								// base class
public:
	virtual void onResize() { ... } 		// base onResize impl
	...
};
class SpecialWindow: public Window { 		// derived class
public:
	virtual void onResize() { 				// derived onResize impl;
	static_cast<Window>(*this).onResize(); 	// cast *this to Window,
											// then call its onResize;
											// this doesn’t work!
	... 									// do SpecialWindow-
	} 										// specific stuff
...
};
```
>学过inside c++ object model理解上面这个应该就很简单了。

上面的cast并不能调用`Window::onResize()`(懂得都懂）。
解决方案是：

```cpp
class SpecialWindow: public Window {
public:
	virtual void onResize() {
		Window::onResize(); // call Window::onResize
		... 				// on *this
	}
...
};
```
>上面也提示了，使用cast可能是发生错误的信号，如果能不用就不用。

`dynamic_cast`的实现是相当slow的！

两种方法避免使用`dynamic_cast`。
先看本来的实现：

```cpp
class Window { ... };
class SpecialWindow: public Window {
public:
	void blink();
	...
};
typedef 											// see Item 13 for info
	std::vector<std::tr1::shared_ptr<Window> > VPW; // on tr1::shared_ptr
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin(); 		// undesirable code:
		iter != winPtrs.end(); 							// uses dynamic_cast
		++iter) {
					if (SpecialWindow *psw = dynamic_cast<SpecialWindow*>(iter->get()))
					psw->blink();
}
```
可以改成：

```cpp
typedef std::vector<std::tr1::shared_ptr<SpecialWindow> > VPSW;
VPSW winPtrs;
...
for (VPSW::iterator iter = winPtrs.begin(); 	// better code: uses
		iter != winPtrs.end(); 					// no dynamic_cast
		++iter)
	(*iter)->blink();
```
或者尽量的声明虚函数：

```cpp
class Window {
public:
	virtual void blink() {} 				// default impl is no-op;
	... 									// see Item 34 for why
}; 											// a default impl may be
											// a bad idea
class SpecialWindow: public Window {
public:
	virtual void blink() { ... } 			// in this class, blink
	... 									// does something
};
typedef std::vector<std::tr1::shared_ptr<Window> > VPW;
VPW winPtrs; 								// container holds
											// (ptrs to) all possible
... 										// Window types
for (VPW::iterator iter = winPtrs.begin();
	iter != winPtrs.end();
	++iter) 								// note lack of
	(*iter)->blink(); 							// dynamic_cast
```
下面的使用方式要极力的避免！丑陋且不宜维护：

```cpp
class Window { ... };
... 												// derived classes are defined here
typedef std::vector<std::tr1::shared_ptr<Window> > VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter)
{
	if (SpecialWindow1 *psw1 =
			dynamic_cast<SpecialWindow1*>(iter->get())) { ... }
	else if (SpecialWindow2 *psw2 =
			dynamic_cast<SpecialWindow2*>(iter->get())) { ... }
	else if (SpecialWindow3 *psw3 =
			dynamic_cast<SpecialWindow3*>(iter->get())) { ... }
	...
}
```

>**Things to Remember**
✦ Avoid casts whenever practical, especially dynamic_casts in performance-sensitive code. If a design requires casting, try to develop a cast-free alternative.
✦ When casting is necessary, try to hide it inside a function. Clients can then call the function instead of putting casts in their own code.
✦ Prefer C++-style casts to old-style casts. They are easier to see, and they are more specific about what they do.

## item28：avoid returning "handles" to object internals.

```cpp
class Point { 				// class for representing points
public:
	Point(int x, int y);
	...
	void setX(int newVal);
	void setY(int newVal);
	...
};

struct RectData { 		// Point data for a Rectangle
	Point ulhc; 		// ulhc = “ upper left-hand corner”
	Point lrhc; 		// lrhc = “ lower right-hand corner”
};
class Rectangle {
...
private:
	std::tr1::shared_ptr<RectData> pData; 	// see Item 13 for info on
}; 											// tr1::shared_ptr
```
如果获得左上和右下的函数如下：

```cpp
class Rectangle {
public:
	...
	Point& upperLeft() const { return pData->ulhc; }
	Point& lowerRight() const { return pData->lrhc; }
	...
};
```
下面就会出现一个使const对象改变值的自我矛盾！

```cpp
Point coord1(0, 0);
Point coord2(100, 100);
const Rectangle rec(coord1, coord2); 	// rec is a const rectangle from
										// (0, 0) to (100, 100)
rec.upperLeft().setX(50); 				// now rec goes from
										// (50, 0) to (100, 100)!
```

可以采取下面的办法防止出现问题（返回const的引用）：

```cpp
class Rectangle {
public:
	...
	const Point& upperLeft() const { return pData->ulhc; }
	const Point& lowerRight() const { return pData->lrhc; }
	...
}
```
但是这又会导致*dangling handles*

```cpp
class GUIObject { ... };
const Rectangle 					// returns a rectangle by
boundingBox(const GUIObject& obj); 	// value; see Item 3 for why
									// return type is const
```
用户会这么使用：

```cpp
GUIObject *pgo; 					// make pgo point to
... 								// some GUIObject
const Point *pUpperLeft = 			// get a ptr to the upper
 &(boundingBox(*pgo).upperLeft()); 	// left point of its
									// bounding box
```
就会导致上面的问题。


>如果返回对象而不是handles会解决这个问题吗？

但有些例外必须要返回引用（这些都有明确的生命期）如`operator[]`。

>**Things to Remember**
✦ Avoid returning handles (references, pointers, or iterators) to object internals. Not returning handles increases encapsulation, helps const member functions act const, and minimizes the creation of dangling handles.
## item29:strive for exception-safe code.
考虑下面的有背景图片的GUI menus：

```cpp
class PrettyMenu {
public:
	...
	void changeBackground(std::istream& imgSrc); 			// change background
	... 			// image
private:
 Mutex mutex; 		// mutex for this object
Image *bgImage; 	// current background image
int imageChanges; 	// # of times image has been changed
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
	lock(&mutex); 				// acquire mutex (as in Item 14)
	delete bgImage; 			// get rid of old background
	++imageChanges; 			// update image change count
	bgImage = new Image(imgSrc);// install new background
	unlock(&mutex); 			// release mutex
}

```
上面的函数`changeBackground()`是很差的，而exception-safe要求：
- leak no resources.
- don't allow data structures to become corrupted.

第一个好解决使用item14用对象管理mutex：

```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
	Lock ml(&mutex); 	// from Item 14: acquire mutex and
						// ensure its later release
	delete bgImage;
	++imageChanges;
	bgImage = new Image(imgSrc);
}
```
第二个怎么解决呢？先介绍三个术语,exception-safe function 提供其中之一的保证：
- the basic guarantee
- the strong guarantee
- the nothrow guarantee：所有的操作都使用**内置类型**

>`int doSomething() throw(); // note empty exception spec`
>什么意思？`throw()`

下面我们为函数提供strong guarantee：

```cpp
class PrettyMenu {
	...
	std::tr1::shared_ptr<Image> bgImage;
	...
};
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
	Lock ml(&mutex);
	bgImage.reset(new Image(imgSrc)); 	// replace bgImage’s internal
										// pointer with the result of the
										// “new Image” expression
	++imageChanges;
}
```
使用了**智能指针**和**语句重拍**。
但是还有问题，如果`imgsrc`构造器出现exception，那么上述的也只能提供基本保证！


上面的先放在一边，先学习一个strong guarantee的**通用策略***copy and swap*. swap是non-throwing的（见item25）

介绍**经典**的实现：
先实现"pimpl idiom"(item31介绍了）

```cpp
struct PMImpl { 										// PMImpl = “PrettyMenu
	std::tr1::shared_ptr<Image> bgImage; 				// Impl.”; see below for
	int imageChanges; 									// why it’s a struct
};
class PrettyMenu {
...
private:
	Mutex mutex;
	std::tr1::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
	using std::swap; 									// see Item 25
	Lock ml(&mutex); 									// acquire the mutex
	std::tr1::shared_ptr<PMImpl> 						// copy obj. data
			pNew(new PMImpl(*pImpl));
	pNew->bgImage.reset(new Image(imgSrc)); 			// modify the copy
	++pNew->imageChanges;
	swap(pImpl, pNew); 									// swap the new
														// data into place
} 														// release the mutex
```
>上面的pimpl idiom值得学习，注意拷贝语义！

上面并不能保证所有整个函数都是strongly exception-safe,比如调用了其他函数：

```cpp
void someFunc()
{
	... // make copy of local state
	f1();
	f2();
	... // swap modified state into place
}
```
**即便**f1和f2都是strongly exception-safe，如果发f1执行完了，执行f2的时候出现了异常。。。。

copy and swap的效率并不是很好

>Things to Remember
✦ Exception-safe functions leak no resources and allow no data structures to become corrupted, even when exceptions are thrown. Such functions offer the basic, strong, or nothrow guarantees.
✦ The strong guarantee can often be implemented via copy-and-swap,but the strong guarantee is not practical for all functions.
✦ A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls.
## item30:understand the ins and outs of inlining.
声明`inline`对编译器来说是request,不是command。分为显式声明和隐式声明。

inline函数应该在头文件。模板也必须在头文件内！（不同的编译器可能不同）

inline函数的指针调用不会inline

ctor和dtor不会inline（编译器会插入很多东西）：

```cpp
class Base {
public:
	...
private:
	std::string bm1, bm2; 		// base members 1 and 2
};
class Derived: public Base {
public:
	Derived() {} 				// Derived’s ctor is empty — or is it?
	...
private:
	std::string dm1, dm2, dm3; 	// derived members 1–3
};
```
`Derived()`可能是：

```cpp
Derived::Derived() 						// conceptual implementation of
{ 										// “empty” Derived ctor
	Base::Base(); 						// initialize Base part
	try { dm1.std::string::string(); } 	// try to construct dm1
	catch (...) { 						// if it throws,
		Base::~Base(); 					// destroy base class part and
		throw; 							// propagate the exception
	}
	try { dm2.std::string::string(); } 	// try to construct dm2
	catch(...) { 						// if it throws,
		dm1.std::string::~string(); 	// destroy dm1,
		Base::~Base(); 					// destroy base class part, and
		throw; 							// propagate the exception
	}
	try { dm3.std::string::string(); } 	// construct dm3
	catch(...) { 						// if it throws,
		dm2.std::string::~string(); 	// destroy dm2,
		dm1.std::string::~string(); 	// destroy dm1,
		Base::~Base(); 					// destroy base class part, and
		throw; 							// propagate the exception
	}
}
```
>**Things to Remember**
✦ Limit most inlining to small, frequently called functions. This facilitates debugging and binary upgradability, minimizes potential code bloat, and maximizes the chances of greater program speed.
✦ Don’t declare function templates inline just because they appear in header files.

## item31:Minimize compliation dependence between files.
小小的改变需要recompile &relink,这是因为接口和实现不能完全的分离。

```cpp
class Person {
public:
	Person(const std::string& name, const Date& birthday,
	const Address& addr);
	std::string name() const;
	std::string birthDate() const;
	std::string address() const;
	...
private:
	std::string theName; 	// implementation detail
	Date theBirthDate; 		// implementation detail
	Address theAddress; 	// implementation detail
};
```
为什么不是只定义接口呢，只有接口变了再重新编译？

```cpp
namespace std {
	class string; 				// forward declaration (an incorrect
} 								// one — see below)
class Date; 					// forward declaration
class Address; 					// forward declaration
class Person {
public:
	Person(const std::string& name, const Date& birthday,
	const Address& addr);
	std::string name() const;
	std::string birthDate() const;
	std::string address() const;
	...
};
```
首先因为前向声明的不确定性，其次编译器要知道对象的大小来分配空间。
可以用下面的两个类来实现一定程度的实现分离：

```cpp
#include <string> 	// standard library components
					// shouldn’t be forward-declared
#include <memory> 	// for tr1::shared_ptr; see below
class PersonImpl; 	// forward decl of Person impl. class
class Date; 		// forward decls of classes used in
class Address; 		// Person interface
class Person {
public:
	Person(const std::string& name, const Date& birthday,
	const Address& addr);
	std::string name() const;
	std::string birthDate() const;
	std::string address() const;
	...
private: 									// ptr to implementation;
	std::tr1::shared_ptr<PersonImpl> pImpl; // see Item 13 for info on
}; 											// std::tr1::shared_ptr
```

上面的分离的关键是把定义的依赖替换成声明的依赖。
- Avoid using objects when object references and pointers will do.
- Depend on class declarations instead of class definitions whenever you can.

函数的定义不需要类定义只需要类声明就行！

```cpp
class Date; 					// class declaration
Date today(); 					// fine — no definition
void clearAppointments(Date d); // of Date is needed
```
- Provide separate header files for declarations and definitions.

下面是实现使用pimpl idiom(也被成为**handle class**）：

```cpp
#include "Person.h" 					// we’re implementing the Person class,
										// so we must #include its class definition

#include "PersonImpl.h" 				// we must also #include PersonImpl’s class
										// definition, otherwise we couldn’t call
										// its member functions; note that
										// PersonImpl has exactly the same public
										// member functions as Person — their
										// interfaces are identical
Person::Person(const std::string& name, const Date& birthday,
const Address& addr)
: pImpl(new PersonImpl(name, birthday, addr))
{}
std::string Person::name() const
{
	return pImpl->name();
}
```

另一种实现handle class的方法是interface class（特定的虚基类）。

```cpp
class Person {
public:
	virtual ~Person();
	virtual std::string name() const = 0;
	virtual std::string birthDate() const = 0;
	virtual std::string address() const = 0;
	...
};
```
有纯虚函数的类不能实例化！所以必须提供factory function给派生类实例化虚基类：

```cpp
class Person {
public:
	...
	static std::tr1::shared_ptr<Person> 	// return a tr1::shared_ptr to a new
		create(const std::string& name, 	// Person initialized with the
				const Date& birthday, 		// given params; see Item 18 for
				const Address& addr); 		// why a tr1::shared_ptr is returned ...
};
```
客户可以如下的使用：

```cpp
std::string name;
Date dateOfBirth;
Address address;
...
// create an object supporting the Person interface
std::tr1::shared_ptr<Person> pp(Person::create(name, dateOfBirth, address));
...
std::cout 	<< pp->name() 			// use the object via the
			<< " was born on " 		// Person interface
			<< pp->birthDate()
			<< " and now lives at "
			<< pp->address();
			... 					// the object is automatically
									// deleted when pp goes out of
									// scope — see Item 13
```

所以背后是如何实现的？

```cpp
class RealPerson: public Person {
public:
	RealPerson(const std::string& name, const Date& birthday,
	const Address& addr)
	: theName(name), theBirthDate(birthday), theAddress(addr)
	{}
	virtual ~RealPerson() {}
	std::string name() const; 			// implementations of these
	std::string birthDate() const; 		// functions are not shown, but
	std::string address() const; 		// they are easy to imagine
private:
	std::string theName;
	Date theBirthDate;
	Address theAddress;
};
```
`Person::create`:

```cpp
std::tr1::shared_ptr<Person> Person::create(const std::string& name,
const Date& birthday,
const Address& addr)
{
	return std::tr1::shared_ptr<Person>(new RealPerson( name, birthday,
addr));
}
```
`RealPerson`阐述了两个实现接口类的机制：
1. 继承接口，实现接口
2. 再多继承中实现接口（见item40）


handle class和interface class都能实现接口和实现解耦，进而减少编译依赖。

>犬儒主义，看穿。即花费是什么？

handle class增加了indirection，interface class也增加indirection。

>**Things to Remember**
✦ The general idea behind minimizing compilation dependencies is to depend on declarations instead of definitions. Two approaches
based on this idea are Handle classes and Interface classes.
✦ Library header files should exist in full and declaration-only forms. This applies regardless of whether templates are involved.

# chap06:Inheritance and Object-Oriented Design

## item32:Make sure public inheritance models"is-a".

>**Things to Remember**
✦ Public inheritance means “is-a.” Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object.

## item33:avoid hiding inherited names.
模板继承的name hiding的问题见item43

>**Things to Remember**
✦ Names in derived classes hide names in base classes. Under public inheritance, this is never desirable.
✦ To make hidden names visible again, employ using declarations or forwarding functions.

## Item 34: Differentiate between inheritance of interface and inheritance of implementation.
public继承可以细分为inheritance of function interfaces & inheriance of function implementations。

到底是继承接口还是继承实现？override实现或者直接继承实现？

下面的例子：

```cpp
class Shape {
public:
	virtual void draw() const = 0;
	virtual void error(const std::string& msg);
	int objectID() const;
	...
};
class Rectangle: public Shape { ... };
class Ellipse: public Shape { ... };
```

**纯虚函数**的目的是继承接口（纯虚函数也可以定义，但是使用的时候要用类名）：

```cpp
Shape *ps = new Shape; // error! Shape is abstract
Shape *ps1 = new Rectangle; // fine
ps1->draw(); // calls Rectangle::draw
Shape *ps2 = new Ellipse; // fine
ps2->draw(); // calls Ellipse::draw
ps1->Shape::draw(); // calls Shape::draw
ps2->Shape::draw(); // calls Shape::draw
```

**虚函数**的目的继承接口和default实现。
但是继承default实现会出现问题（会让你不经意的使用）：

```cpp
class Airport { ... }; // represents airports
class Airplane {
public:
	virtual void fly(const Airport& destination);
	...
};
void Airplane::fly(const Airport& destination)
{
	default code for flying an airplane to the given destination
}
class ModelA: public Airplane { ... };
class ModelB: public Airplane { ... };
```
如果着急着上线新飞机，但新飞机有不同的`fly()`忘记override了：

```cpp
class ModelC: public Airplane {
... // no fly function is declared
};
```

```cpp
Airport PDX(...); // PDX is the airport near my home
Airplane *pa = new ModelC;
...
pa->fly(PDX); // calls Airplane::fly!
```
就会造成灾难！一个可行的方法是（声明为纯虚，添加`protected`的`defaultFly()`：

```cpp
class Airplane {
public:
	virtual void fly(const Airport& destination) = 0;
	...
protected:
	void defaultFly(const Airport& destination);
};
void Airplane::defaultFly(const Airport& destination)
{
	default code for flying an airplane to the given destination
}
```
然后：

```cpp
class ModelA: public Airplane {
public:
	virtual void fly(const Airport& destination)
	{ defaultFly(destination); }
	...
};
class ModelB: public Airplane {
public:
	virtual void fly(const Airport& destination)
	{ defaultFly(destination); }
	...
}

class ModelC: public Airplane {
public:
	virtual void fly(const Airport& destination);
	...
};
void ModelC::fly(const Airport& destination)
{
	code for flying a ModelC airplane to the given destination
}
```
> 在定义虚函数的时候要思考如果派生类没有覆写怎么办？

上面的解决方案也有缺点，函数名的激增，故而提供了另一种解决方法（融合了`fly()`和`defaultFly()`)：

```cpp
void Airplane::fly(const Airport& destination) // an implementation of
{ // a pure virtual function
	default code for flying an airplane to
	the given destination
}
class ModelA: public Airplane {
public:
	virtual void fly(const Airport& destination)
	{ Airplane::fly(destination); }
	...
};
class ModelB: public Airplane {
public:
	virtual void fly(const Airport& destination)
	{ Airplane::fly(destination); }
	...
};
class ModelC: public Airplane {
public:
	virtual void fly(const Airport& destination);
	...
};
void ModelC::fly(const Airport& destination)
{
	code for flying a ModelC airplane to the given destination
}
```

非虚函数的目的是继承接口和**强制**实现！

>**Things to Remember**
✦ Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes always inherit base class interfaces.
✦ Pure virtual functions specify inheritance of interface only.
✦ Simple (impure) virtual functions specify inheritance of interface plus inheritance of a default implementation.
✦ Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation.

## item35:consider alternatives to virtual functions.
游戏设计：

```cpp
class GameCharacter {
public:
	virtual int healthValue() const; // return character’s health rating;
	... // derived classes may redefine this
};
```
**The Template Method Pattern via the Non-Virtual Interface Idiom**

一个古老的流派：所有的虚函数都应该是private的。

```cpp
class GameCharacter {
public:
	int healthValue() const 			// derived classes do not redefine
	{ 									// this — see Item 36
		... 							// do “before” stuff — see below
		int retVal = doHealthValue(); 	// do the real work
		... 							// do “after” stuff — see below
		return retVal;
	}
	...
private:
	virtual int doHealthValue() const 	// derived classes may redefine this
	{
		... 							// default algorithm for calculating
	} 									// character’s health
};
```
这是non-virtual interface（NVI）idiom。也可以说非虚函数是虚函数的warpper。

NVI保证了调用虚函数的时候有正确的上下文（context）！比如在"before"  stuff之前上锁等等。
>你最好不要让客户直接调用虚函数！

>access-level，在基类虚函数声明为private，在派生类能够声明成public吗？

**The Strategy Pattern via Function Pointers**

```cpp
class GameCharacter; // forward declaration
// function for the default health calculation algorithm
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter {
public:
	typedef int (*HealthCalcFunc)(const GameCharacter&);
	explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
	: healthFunc(hcf )
	{}
	int healthValue() const
	{ return healthFunc(*this); }
	...
private:
	HealthCalcFunc healthFunc;
};
```
- 不同的实例可以有不同的函数：

```cpp
class EvilBadGuy: public GameCharacter {
public:
	explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc)
	: GameCharacter(hcf )
	{ ... }
	...
};
int loseHealthQuickly(const GameCharacter&); // health calculation
int loseHealthSlowly(const GameCharacter&); // funcs with different
// behavior
EvilBadGuy ebg1(loseHealthQuickly); // same-type characEvilBadGuy ebg2(loseHealthSlowly); // ters with different
// health-related
// behavior
```
- 使用指针可以在**运行时**改变指针值（即改变函数）

注意函数指针调用的函数无法访问non-public的类成员。如果使用友元和public accessor函数（这些函数可能更希望hidden）可能或多或少的破坏封装。

**The Strategy Pattern via tr1::function**

```cpp
class GameCharacter; // as before
int defaultHealthCalc(const GameCharacter& gc); // as before
class GameCharacter {
public:
	// HealthCalcFunc is any callable entity that can be called with
	// anything compatible with a GameCharacter and that returns anything
	// compatible with an int; see below for details
	typedef std::tr1::function<int (const GameCharacter&)> HealthCalcFunc;
	explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
	: healthFunc(hcf )
	{}
	int healthValue() const
	{ return healthFunc(*this); }
	...
private:
	HealthCalcFunc healthFunc;
};
```
注意：
`std::tr1::function<int (const GameCharacter&)>`
客户使用的时候就有了惊人的灵活性：

```cpp
short calcHealth(const GameCharacter&); 		// health calculation
												// function; note
												// non-int return type
struct HealthCalculator { 						// class for health
	int operator()(const GameCharacter&) const 	// calculation function
	{ ... } 									// objects
};
class GameLevel {
public:
	float health(const GameCharacter&) const; 	// health calculation 
	... 										// mem function; note
}; 												// non-int return type
class EvilBadGuy: public GameCharacter { 		// as before
	...
};

class EyeCandyCharacter: public GameCharacter { // another character
	... 									// type; assume same
}; 											// constructor as
											// EvilBadGuy
EvilBadGuy ebg1(calcHealth); 				// character using a
											// health calculation
											// function
EyeCandyCharacter ecc1(HealthCalculator()); // character using a
											// health calculation
											// function object
GameLevel currentLevel;
...
EvilBadGuy ebg2( 							// character using a
	std::tr1::bind(&GameLevel::health, 		// health calculation
					currentLevel, 			// member function;
					_1) 					// see below for details
);
```
**The “Classic” Strategy Pattern**
更加合适的设计是把health-calculation函数设置在health-calculation继承关系中的虚函数：
![在这里插入图片描述](https://img-blog.csdnimg.cn/ed6b84018e52438a86e9d811a8fe2348.png)
下面是代码阐述：

```cpp
class GameCharacter; // forward declaration
class HealthCalcFunc {
public:
	...
	virtual int calc(const GameCharacter& gc) const
	{ ... }
	...
};
HealthCalcFunc defaultHealthCalc;
class GameCharacter {
public:
	explicit GameCharacter(HealthCalcFunc *phcf = &defaultHealthCalc)
	: pHealthCalc(phcf)
	{}
	int healthValue() const
	{ return pHealthCalc->calc(*this); }
	...
private:
	HealthCalcFunc *pHealthCalc;
};
```
总结：
■ Use the **non-virtual interface idiom** (NVI idiom), a form of the Template Method design pattern that wraps public non-virtual member functions around less accessible virtual functions.
■ Replace virtual functions with **function pointer data members**, a stripped-down manifestation of the Strategy design pattern.
■ Replace virtual functions with **tr1::function data members**, thus allowing use of any callable entity with a signature compatible with what you need. This, too, is a form of the Strategy design pattern.
■ Replace virtual functions in one hierarchy with **virtual functions in another hierarchy**. This is the conventional implementation of the Strategy design pattern.


>Things to Remember
✦ Alternatives to virtual functions include the NVI idiom and various forms of the Strategy design pattern. The NVI idiom is itself an example of the Template Method design pattern.
✦ A disadvantage of moving functionality from a member function to a function outside the class is that the non-member function lacks access to the class’s non-public members.
✦ tr1::function objects act like generalized function pointers. Such objects support all callable entities compatible with a given target signature.

## item36:never redefine an inherited non-virtual function.
>Things to Remember
✦ Never redefine an inherited non-virtual function.

## item37:never redefine a function's inherited default parameter value.
*virtual functions are dynamically bound, but defaultparameter values are statically bound.*

出于性能的需要，才做出了这样奇怪的设计！

使用NVI idiom解决这个问题：

```cpp
class Shape {
public:
	enum ShapeColor { Red, Green, Blue };
	void draw(ShapeColor color = Red) const // now non-virtual
	{
	doDraw(color); // calls a virtual
	}
	...
private:
	virtual void doDraw(ShapeColor color) const = 0; // the actual work is
}; // done in this func
class Rectangle: public Shape {
public:
	...
private:
	virtual void doDraw(ShapeColor color) const; // note lack of a
	... // default param val.
};
```
>Things to Remember
✦ Never redefine an inherited default parameter value, because default parameter values are statically bound, while virtual functions — the only functions you should be redefining — are dynamically bound.

## Item 38: Model “has-a” or “is-implemented-in-terms-of” through composition.
*application domain*和*implementation domain*

>**Things to Remember**
✦ Composition has meanings completely different from that of public inheritance.
✦ In the application domain, composition means has-a. In the implementation domain, it means is-implemented-in-terms-of.

## item39:use private inheritance judiciously.
*private inheritance means that implementation only should be inherited; interface should be ignored.*

If D privately inherits from B, it means that D objects are implemented **in terms of** B objects.
所以这里的in-terms-of和item38中**composition**是什么关系？-------能compose就compose,然后才是private inherited。

假若有`Widget`类，我们想追踪成员函数的调用次数：

```cpp
class Timer {
public:
	explicit Timer(int tickFrequency);
	virtual void onTick() const; 		// automatically called for each tick
	...
};
```
让`Widget`public继承是肯定不对的，所以：

```cpp
class Widget: private Timer {
private:
	virtual void onTick() const; // look at Widget usage data, etc.
	...
};
```
如果我们也可以使用composition（很复杂）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/00ab16bb72b14d149db0e79389d74e7f.png)
但是我们还是更喜欢public继承+composition（给出了两点理由）。

但是还是有一些edge case！比如**空类**（没有非静态数据成员，没有虚函数和虚基类）

>**Things to Remember**
✦ Private inheritance means is-implemented-in-terms of. It’s usually inferior to composition, but it makes sense when a derived class
needs access to protected base class members or needs to redefine inherited virtual functions.
✦ Unlike composition, private inheritance can enable the empty base optimization. This can be important for library developers who strive
to minimize object sizes.

## Item 40: Use multiple inheritance judiciously.
关于MI的讨论分为两派。

首先是多继承会有命名冲突：

```cpp
class BorrowableItem { // something a library lets you borrow
public:
	void checkOut(); // check the item out from the library
	...
};
class ElectronicGadget {
private:
	bool checkOut() const; // perform self-test, return whether
	... // test succeeds
};
class MP3Player: // note MI here
	public BorrowableItem, // (some libraries loan MP3 players)
	public ElectronicGadget
{ ... }; // class definition is unimportant
MP3Player mp;
mp.checkOut(); // ambiguous! which checkOut?
```
虽然两个函数的accessibility是不一样的，一个是public，一个是private的；**但是先检查名字，再检查accessible。**

为了消除ambiguity：

```cpp
mp.BorrowableItem::checkOut(); // ah, that checkOut...
```

还有一个严重的问题：
![在这里插入图片描述](https://img-blog.csdnimg.cn/493a5b9032eb40e2b06375778fc4ca5d.png)
重复还是不重复c++没有立场，它只是提供了另一种选择：
![在这里插入图片描述](https://img-blog.csdnimg.cn/7d656ff20f184fe18c2f2773451c1461.png)

从理论上说public继承永远都是virtual。所以public继承**都**声明成虚的。**但是**正确性不是唯一的要求！虚继承很复杂！

所以有两个建议：
1. 尽量不要使用虚基类
2. 如果必须使用虚基类，不要放置数据。

下面我们用下面的接口类去建模person

```cpp
class IPerson {
public:
	virtual ~IPerson();
	virtual std::string name() const = 0;
	virtual std::string birthDate() const = 0;
};
```
`IPerson`的用户不能实例化，需要用factory function去实例化派生类：

```cpp
// factory function to create a Person object from a unique database ID;
// see Item 18 for why the return type isn’t a raw pointer
std::tr1::shared_ptr<IPerson> makePerson(DatabaseID personIdentifier);
// function to get a database ID from the user
DatabaseID askUserForDatabaseID();

DatabaseID id(askUserForDatabaseID());
std::tr1::shared_ptr<IPerson> pp(makePerson(id)); // create an object
// supporting the
// IPerson interface
... // manipulate *pp via
// IPerson’s member
// functions
```
如何使得`makePerson()`返回指针呢？我们需要创建一个派生类`CPerson`你可以重新去写这个类，也可以从已有的类`PersonInfo`去拓展：

```cpp
class PersonInfo {
public:
	explicit PersonInfo(DatabaseID pid);
	virtual ~PersonInfo();
	virtual const char * theName() const;
	virtual const char * theBirthDate() const;
	...
private:
	virtual const char * valueDelimOpen() const; // see
	virtual const char * valueDelimClose() const; // below
	...
};
```
一些实现：

```cpp
const char * PersonInfo::valueDelimOpen() const
{
	return "["; // default opening delimiter
}
const char * PersonInfo::valueDelimClose() const
{
	return "]"; // default closing delimiter
}
const char * PersonInfo::theName() const
{
	// reserve buffer for return value; because this is
	// static, it’s automatically initialized to all zeros
	static char value[Max_Formatted_Field_Value_Length];
	// write opening delimiter
	std::strcpy(value, valueDelimOpen());
	append to the string in value this object’s name field (being careful
	to avoid buffer overruns!)
	// write closing delimiter
	std::strcat(value, valueDelimClose());
	return value;
}
```
`valueDelimOpen`和 `valueDelimClose`都是虚函数，那么你就可以在派生类中改写你的返回规则，如：“Homer”, not “[Homer]”.

`CPerson`和`PersonInfo`的关系是is-implemented-in-terms-of。可以使用composition或者private继承。在这里要**改写虚函数**，故选择继承。

```cpp
class IPerson { // this class specifies the
public: // interface to be implemented
	virtual ~IPerson();
	virtual std::string name() const = 0;
	virtual std::string birthDate() const = 0;
};
class DatabaseID { ... }; // used below; details are
// unimportant
class PersonInfo { // this class has functions
public: // useful in implementing
	explicit PersonInfo(DatabaseID pid); // the IPerson interface
	virtual ~PersonInfo();
	virtual const char * theName() const;
	virtual const char * theBirthDate() const; 
	...
private:
	virtual const char * valueDelimOpen() const;
	virtual const char * valueDelimClose() const; 
	...
};

class CPerson: public IPerson, private PersonInfo { // note use of MI
public:
	explicit CPerson(DatabaseID pid): PersonInfo(pid) {}
	virtual std::string name() const // implementations
	{ return PersonInfo::theName(); } // of the required
	// IPerson member
	virtual std::string birthDate() const // functions
	{ return PersonInfo::theBirthDate(); }
private: // redefinitions of
	const char * valueDelimOpen() const { return ""; } // inherited virtual
	const char * valueDelimClose() const { return ""; } // delimiter
};
```
上面的关系如下（UML）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/1b5d10bab42e4d31abd9a0d2fa6d67ec.png)

>Things to Remember
✦ Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance.
✦ Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It’s most practical when virtual base classes have no data.
✦ Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.

# chap07:Templates and Generic Programming
**generic programming** — the ability to write code that is independent of the types of objects being manipulated.

c++的模板机制是图灵完全的：**template metaprogramming**: the creation of programs that execute inside C++ compilers and that stop running when compilation is complete