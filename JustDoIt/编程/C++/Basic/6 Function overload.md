
How overloaded functions are differentiated:

|Function property|Used for differentiation|Notes|
|---|---|---|
|Number of parameters|Yes||
|Type of parameters|Yes|Excludes typedefs, type aliases, and const qualifier on value parameters. Includes ellipses.|
|Return type|No||

Note that a function’s return type is not used to differentiate overloaded functions. We’ll discuss this more in a bit.


For ***member functions***, additional function-level qualifiers are also considered:

|Function-level qualifier|Used for overloading|
|---|---|
|const or volatile|Yes|
|Ref-qualifiers|Yes|

As an example, a const member function can be differentiated from an otherwise identical non-const member function (even if they share the same set of parameters).

⚠️：If a parameter is given a default argument, all subsequent parameters (to the right) must also be given default arguments.

