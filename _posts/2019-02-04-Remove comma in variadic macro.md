---
title: "Remove the comma in variadic macro"
layout: post
tags: [C, Pre-Processor, GCC, Variadic Macro]
published: false
comments: true
---

# Problem Definition
### Remove the comma in variadic macro when there is no argument with GCC extension.

For example, if we define a macro as the example in [GNU's preprocessor section: Variadic Macro](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html).

` #define eprintf(…) fprintf (stderr, __VA_ARGS__)`

When we call `eprintf();` it becomes `fprintf(stderr,);`  
Then  it raise compiler error: *[Error] expected expression before ')' token*

In fact, for typical case, you can directly modify the above macro to this.

`#define eprintf(format, …) fprintf (stderr, format, ##__VA_ARGS__)`

> Historically, GNU CPP has also had another extension to handle the trailing comma: the ‘##’ token paste operator has a special meaning when placed between a comma and a variable argument. Despite the introduction of `__VA_OPT__`, this extension remains supported in GNU CPP, for backward compatibility. and the variable argument is left out when the eprintf macro is used, then the comma before the ‘##’ will be deleted. This does not happen if you pass an empty argument, nor does it happen if the token preceding ‘##’ is anything other than a comma.

But, `##__VA_ARGS__` is only supported with extension, which means it doesn't support when you compile your code with C standard, such as `gcc -std=c11`
# Idea
### The idea all comes from [Jens Gustedt's Blog: Detect empty macro arguments](https://gustedt.wordpress.com/2010/06/08/detect-empty-macro-arguments/?fbclid=IwAR1Qv1NSGoLEorClLebfno-j5EbNvUTti2s49TRIEaelxobZH7-GkQjK5ww)

> The idea is to put the arguments that we want to test between the macro and its parenthesis, such that the macro only triggers if the arguments are empty:

In fact, I am just going to explain this idea in detail by my human-compiler XD.

Fisrt, we define two macro: (Note: `TEST()` is just for explanation, we won't define it in real practice.)

```c
#define _TRIGGER_PARENTHESIS_(...) ,
#define TEST(...) _TRIGGER_PARENTHESIS_ __VA_ARGS__ (/*empty*/)
```

**Key Point: `TEST()` will only be able to expand to `,` when we pass empty argument in TEST().**

Give some test arguments in `TEST()` to give a better explanation.

`TEST(1)` becomes `_TRIGGER_PARENTHESIS_ 1 ()`.

Since we only define `_TRIGGER_PARENTHESIS_(...) `.
Iit's undefined if only `_TRIGGER_PARENTHESIS_ ` appears.

As a result, `TEST(1)` becomes `()` in the end.

When we have the parenthesis in argument, it will expand to `,()`. 

For example, `TEST((1))` becomes `_TRIGGER_PARENTHESIS_ (1) (/*empty*/)`. then it expand to `,()` in the end.

Second, we define another macro `HAS_COMMA` to detect whether argument contains `,` or not.

```c
#define _ARG3(_0, _1, _2, ...) _2
#define HAS_COMMA(...) _ARG3(__VA_ARGS__, 1, 0)
```

Same as above, give some test arguments in `HAS_COMMA()` to give a better explanation.

`HAS_COMMA()` -> `_ARG3( , 1, 0)` -> `0`
`HAS_COMMA(1)` -> `_ARG3(1 , 1, 0)` -> `0`
`HAS_COMMA(,)` -> `_ARG3( , , 1, 0)` -> `1`
`HAS_COMMA(1,2)` -> `_ARG3(1, 2 , 1, 0)` -> `1`
`HAS_COMMA(())` -> `_ARG3(() , 1, 0)` -> `0`
`HAS_COMMA(foo(1,2,3))` -> `_ARG3(foo(1,2,3) , 1, 0)` -> `0`

Note: `_ARG3` macro only support at most 2 arguments, so this one will expand as the 3rd argument:

`HAS_COMMA(1,2,3)` -> `_ARG3(1, 2, 3, 1, 0)` -> `3`
# Solution
Now we have one macro to generate `,` when argument is empty, and another macro to detect `,`.

Combining these macros, we can derive the `ISEMPY()` macro do perform the argument check.

```c
#define ISEMPTY(...) \
    _ISEMPTY( \
        HAS_COMMA(__VA_ARGS__), \
        HAS_COMMA(_TRIGGER_PARENTHESIS_ __VA_ARGS__),  \
        HAS_COMMA(__VA_ARGS__ (/*empty*/)), \
        HAS_COMMA(_TRIGGER_PARENTHESIS_ __VA_ARGS__ (/*empty*/)) \
        ) 
```
We leave the `_ISEMPTY` macro to later, let's dig into these four `HAS_COMMA` macro.

`HAS_COMMA(__VA_ARGS__)` is for typical cases, as explained above, it will return `1` if and only if argument contains `,`.
`HAS_COMMA(_TRIGGER_PARENTHESIS_ __VA_ARGS__)` is to check whether `__VA_ARGS__`'s first argument contains `()`
`HAS_COMMA(_TRIGGER_PARENTHESIS_ (1))` -> `HAS_COMMA(,)` -> `1`.
`HAS_COMMA(__VA_ARGS__ (/*empty*/))` is for special case, it checks whether the argument is another macro/function without argument.
For example, `#define foo(...) 1`, then `HAS_COMMA(foo ())` -> `HAS_COMMA(1)` -> `1`.

Same, give some arguments in `ISEMPTY`.

```c
		ISEMPTY() -> ISEMPTY(
		HAS_COMMA()  -> _ARGS(  ,  1, 0) -> 0
		HAS_COMMA(_TRIGGER_PARENTHESIS_ ), ->_ARGS( , 1, 0) -> 0
		HAS_COMMA(()), -> _ARGS((), 1, 0) -> 0
		HAS_COMMA(_TRIGGER_PARENTHESIS_ ())) -> _ARGS( , , 1, 0) -> 1
		_ISEMPTY(0,0,0,1) -> HAS_COMMA(PASTES(_IS_EMPTY_CASE, 0, 0, 0, 1)) -> HAS_COMMA(_IS_EMPTY_CASE_0001) -> HAS_COMMA(,) -> 1
```
```c
//ISEMPTY((1)) -> _ISEMPTY(
//HAS_COMMA((1)), -> _ARG3((1) ,1,0) -> 0
//HAS_COMMA(_TRIGGER_PARENTHESIS_ (1)), ->  _ARG3(    , , 1, 0) ->1
//HAS_COMMA((1) ()), -> _ARG3((1) (), 1, 0) ->0
//HAS_COMMA(_TRIGGER_PARENTHESIS_ (1)())  -> _ARG3(  ,(),1,0) ->1 
// _ISEMPTY(0,1,0,1) -> HAS_COMMA(PASTES(_IS_EMPTY_CASE_, 0, 1, 0, 1)) -> HAS_COMMA(_IS_EMPTY_CASE_0101) -> HAS_COMMA() -> 0
```


```c
#define _ARG3(_0, _1, _2, ...) _2
#define HAS_COMMA(...) _ARG3(__VA_ARGS__, 1, 0)

// Expand for more arguments, you can refer to following examples
// 4 Arguments
//#define _ARG4(_0, _1, _2, _3, ...) _3
//#define HAS_COMMA(...) _ARG4(__VA_ARGS__, 1, 1, 0)

// 16 arguments
//#define _ARG16( _0,  _1,  _2,  _3,  _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, ...) _15
//#define HAS_COMMA(...) _ARG16(__VA_ARGS__, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0)

#define _TRIGGER_PARENTHESIS_ (...) ,

#define ISEMPTY(...) \
    _ISEMPTY( \
        HAS_COMMA(__VA_ARGS__), \
        HAS_COMMA(_TRIGGER_PARENTHESIS_ __VA_ARGS__),  \
        HAS_COMMA(__VA_ARGS__ (/*empty*/)), \
        HAS_COMMA(_TRIGGER_PARENTHESIS_ __VA_ARGS__ (/*empty*/)) \
        ) 
												
#define PASTES(_0, _1, _2, _3, _4 ) _0 ## _1 ## _2 ## _3 ## _4
#define _ISEMPTY(_0, _1, _2, _3) HAS_COMMA(PASTES(_IS_EMPTY_CASE_, _0, _1, _2, _3))

#define PRINT(...) _PRINT(ISEMPTY(__VA_ARGS__), __VA_ARGS__)
#define _PRINT(count, ...) _PRINT_EXPAND_CHECK_EMPTY(count, __VA_ARGS__)
#define _PRINT_EXPAND_CHECK_EMPTY(count, ...) PRINT_EXPAND_IS_EMPTY_ ##count (__VA_ARGS__)
#define PRINT_EXPAND_IS_EMPTY_0(...)  foo(1, 2, 3, __VA_ARGS__)
#define PRINT_EXPAND_IS_EMPTY_1(...) foo(1, 2, 3)

void foor(int a, ...)
{
}
int bar(...)
{
    return 0;
}

int main(){
    PRINT();
		PRINT(1);
		PRINT(1,2);
		
		//Parenthesis case
		PRINT((1));
		
		//Function call case
		PRINT(bar(1));
		PRINT(bar(1,2,3));
		
		//NOTE: This case will fail since we support at most 2 arguments in _ARG3
		PRINT(1,2,3);
	
		//ISEMPTY(1,2) -> _ISEMPTY(
		// HAS_COMMA(1,2), -> _ARGS(1, 2, 1, 0) -> 1
		//HAS_COMMA(_TRIGGER_PARENTHESIS_ 1, 2), -> _ARGS(1, 2, 1, 0) -> 1
		//HAS_COMMA(1, 2 ()),  ->_ARGS(1, 2 (), 1, 0) -> 1
		//HAS_COMMA(_TRIGGER_PARENTHESIS_ 1,  2())) -> _ARG3(1, 2, (), 1, 0) ->1
		// _ISEMPTY(1,1,1,1) -> HAS_COMMA(PASTES(_IS_EMPTY_CASE, 1, 1, 1, 1))
-> HAS_COMMA(_IS_EMPTY_CASE_1111)		-> HAS_COMMA() -> 0
//ISEMPTY(1,2,3) -> _ISEMPTY(
// HAS_COMMA(1,2,3),  -> _ARGS3( 1, 2, 3 , 1, 0) -> 3
//HAS_COMMA(_TRIGGER_PARENTHESIS_ 1, 2,3), -> _ARG3(1,2,3, 1,0) ->3 
//HAS_COMMA(1,2,3 ()), -> _ARG3(1,2,3 (), 1,0) -> 3()
//HAS_COMMA(_TRIGGER_PARENTHESIS_ 1,2,3 ()) -> _ARG3(1,2,3 (), 1,0) -> 3()
//_ISEMPTY(3,3,3(), 3()) -> HAS_COMMA(PASTES(_IS_EMPTY_CASE_, 3 3 3(), 3()))  -> HAS_COMMA(_IS_EMPTY_CASE_333()3()) -> build fail!!

//ISEMPTY(bar(1)) -> _ISEMPTY(
// HAS_COMMA(bar(1)),  -> _ARG3(bar(1) ,1,0) -> 0
//HAS_COMMA(_TRIGGER_PARENTHESIS_ bar(1)), -> _ARG3(bar(1) ,1, 0 )-> 0

// HAS_COMMA(bar(1) ()),  -> _ARG3(bar(1) (), 1,0) -> 0
//HAS_COMMA(_TRIGGER_PARENTHESIS_ bar(1)())  -> _ARG3(bar(1) (), 1,0) -> 0
//_ISEMPTY(0,0,0,0) -> HAS_COMMA(PASTES(_IS_EMPTY_CASE_, 0, 0,0,0)) -> HAS_COMMA(_IS_EMPTY_CASE_0000) -> HAS_COMMA() -> 0


}.
```