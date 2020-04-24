一般来说，在花括号内声明一个名称会将此名称的可见性限制在大括号定义的范围内。在C++98-style的枚举类型中声明的枚举成员不是这样的。这些枚举成员的名称属于包含枚举类的范围，这意味着该范围内的任何其他东西都不能有相同的名称:
```cpp
	enum Color { black, white, red };	// black, white, red are
										// in same scope as Color
	auto white = false;			// error! white already
					// declared in this scope
```
这些枚举成员名称泄漏到包含它们的枚举类的定义范围，这一事实导致了这种enum的正式术语：unscoped。它们的C++11版本scoped enums不会以这种方式泄漏名称：
```cpp
	enum class Color { black, white, red };     // black, white, red
                                                // are scoped to Color
												
	auto white = false; 		// fine, no other
								// "white" in scope
												
	Color c = white; 			// error! no enumerator named
								// "white" is in this scope
												
	Color c = Color::white; 	// fine
	
	auto c = Color::white; 		// also fine (and in accord
								// with Item 5's advice)
```
因为scoped enums是通过“enum类”声明的，所以它们有时被称为enum类。
