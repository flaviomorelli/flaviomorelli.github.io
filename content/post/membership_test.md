---
title: "Membership tests in Julia, R and Python"
date: 2021-03-11T22:27:21+01:00
draft: false
math: true
tags: [julia, python, R, programming]
---

Membership tests –  checking whether an element is in a collection – is a common operation in statistical programming. However, depending on the programming language, the interface and underlying assumptions can vary a lot. After a short introduction to how Python and R implement membership tests, we will take a deeper look into Julia, touching on unique features such as **generic programming** and **multiple dispatch**.

## 1. Membership test operations in Python

Usually, a membership test has its own reserved word or special operation in the language. For example, in Python one could write

```python
>>> 4 in [1, 2, 3]
False

>>> [1, 2] in [1, 2, 3]
False

>>> "a" in "abc"
True

>>> "ab" in "abc"
True
```

Note that in Python the `in` keyword is also used inside the for loop (`for elem in iterable`). If we are testing `list_1 in list_2`, we will test whether the first list as a whole is contained in the other list. When dealing with strings, `in` tests whether our first string is a substring of the second string. 

If you are working with `numpy` arrays, the story is a bit different – as usual with `numpy`. We need the function `numpy.isin` for membership tests. 

```python
>>> import numpy as np
>>> a = np.array(1) # This array has zero dimensions!
array(1) 

>>> num_array = np.arange(1, 4)
array([1, 2, 3])

>>> np.isin(a, num_array) # Same number of dimensions as variable a
array(True)

>>> np.isin(np.array([1]), num_array) # 1 dimension. The input dimensions are kept!
array([ True])

>>> np.isin(np.array([1, 2]), num_array) # Elementwise check
array([ True,  True])

>>> np.isin(np.array([1, 2]), invert=True) # set invert=True to get 'not in'
array([False, False])
```

First, we notice that the input dimensions are preserved in the membership test. Second, the membership test is done elementwise in `numpy`. This was not the case when we tested `[1, 2] in [1, 2, 3]`, where `[1, 2]` as a whole was taken to be a single element. Third, it is not so easy to just write something like `not 4 in [1, 2, 3]` when dealing with `numpy` arrays. To test non-membership, we have to set `invert=True`, because the dimensions of the input array are preserved. The `pandas` library also has an `isin` function for `Series` and `DataFrames`, which works similarly, but without the `invert` option. 

## 2. Membership test operations in R

In R, there is the `%in%` operator which is just another way of using the `match` function (check the documentation `?%in%`). This function is necessary because in R `in` can only be used in a `for`-loop. There are some differences with base Python, but the results are very similar to `np.isin`. 

```R
R> 1 %in% c(1, 2, 3)
[1] TRUE

R> c(1, 2) %in% c(1, 2, 3)
[1] TRUE TRUE

R> 1 %in% list(first = 1, second = 2, third = 3)
[1] TRUE

R> "a" %in% "abc"
[1] FALSE
```

The first example is just checking whether 1 is contained within the vector with the elements 1, 2 and 3. The last three examples, however, are more similar to the way `numpy` deals with membership tests. In the second example, it is checking whether each one of the elements in `c(1, 2)` is in the vector `c(1, 2, 3)`. This differs from the base Python version, which takes the whole list `[1, 2]` to be the element to be searched in the iterable, therefore returning false. In base Python, you would rather use a list comprehension to get the same kind of behavior as R or `numpy`: `[i in (1, 2, 3) for i in (1, 2)]`.

The third example shows that a list in R behaves pretty much like a vector. The single elements of the list can be named (here: `first, second, third`), but the names themselves are not taken into account by `%in%` – not unlike a named tuple. More surprisingly to Python users, the last example with a string returns `FALSE`. This is because strings are *not* **iterables** in R (remember this is a language that in a way is almost as old as C!). There are special functions for this such as `grepl` in the `base` package and `str_detect` from the package `stringr`, also included in the `tidyverse`. 

## 3. Julia

Julia is a modern programming language that is mainly focused on scientific computation. It feels more natural for scientific computing than Python and it has a more modern design than R. Julia's behavior for membership tests is more similar to base Python, but it differs in how it deals with strings. 

### `in`, \\(\in\\), \\(\ni\\)  in Julia

The most basic operation is `in` which tests whether an item is in a given array. Going back to the first example with basic Python we see that the results are the same:  

```julia
julia> 4 in [1, 2, 3]
false

julia> [1, 2] in [1, 2, 3]
false
```

However, `in` is not just a keyword, but a function. As Julia is based on generic programming and multiple dispatch, it actually has 33  different versions of the `in` function. In contrast to Python and R, Julia is statically typed. Each version has a **different signature** that depends on the type of the parameters: 

```julia
julia> in # type the function name to get the number of methods
in (generic function with 33 methods)
    
julia> in(4, [1, 2, 3]) # call in the usual function notation
false
    
julia> @which in(4, [1, 2, 3]) # signature of the method based on the types
in(x, itr) in Base at operators.jl:1055
```

So when calling `in` with an integer and an array we get the signature `in(x, itr)`. We have **two generic types** called `x` and iterable `itr`. We can check the code in `operators.jl`:

```julia
function in(x, itr)
    anymissing = false
    for y in itr
        v = (y == x)
        if ismissing(v)
            anymissing = true
        elseif v
            return true
        end
    end
    return anymissing ? missing : false
end
```

Although we are not stating it explicitly, we assume through duck typing that `itr` is an iterable, because we are using it in the `for` loop. 

Now, let's see what happens when we try to do a similar membership test with strings:

```julia
julia> "a" in "abc"
ERROR: use occursin(x, y) for string containment
  
julia> 'a' in "abc"
true
  
julia> @which in('a', "abc") # get the method signature
in(c::AbstractChar, s::AbstractString) in Base at strings/search.jl:141
```

Surprisingly, `"a"` fails while there is no problem with `'a'`. Unlike Python or R, Julia treats single quotation marks as characters and double quotations as strings. They are two different types, like in good old C. Julia is telling us that if we want to test whether `"a"` is a substring of `"abc"`, we have to use the `occursin` function. However, we can test whether the character `'a'` is in the `"abc"` string with the `in` function. For this comparison, we are using a different comparison function from `Base` which can be found in `strings/search.jl`:

```julia
in(c::AbstractChar, s::AbstractString) = (findfirst(isequal(c),s)!==nothing)
```

With **multiple dispatch**, we determine which function to use at runtime depending on the types. This version of the `in` function in particular is using a specialized function `findfirst` for strings, which checks for each character in `s` whether it is equal to `c` and then checks whether something was found at all with `!==nothing`. (`nothing` is similar to `None` in Python.) 

There is also an alias to the `in` function, \\(\in\\), which can be used by writing `\in` and then hitting the `<tab>` key. Its reverse is \\(\ni\\), which is just swaps the arguments. The following are equivalent:

```julia
julia> 4 ∈ [1, 2, 3] # as infix operator
false

julia> ∈(4, [1, 2, 3]) # as function
false

julia> [1, 2, 3] ∋ 4 
false
```

As a side note, arrays in Julia do not have to contain elements of the same type. Usually, the element type will be `Any` when combining types. However, note that in most cases this is less efficient than making sure all the elements are of the same type:

```julia
julia> [1, 'c']
2-element Array{Any,1}:
 1
  'c': ASCII/Unicode U+0063 (category Ll: Letter, lowercase)
```

### Broadcasting in Julia

We still have not addressed the question of whether it is possible to do membership tests as in `numpy` or R. The first idea would be to try the broadcasting capabilities of Julia. This can be used by adding a dot before an infix operator or after a function name:

```julia
julia> [1,2] .∈ [1, 2, 3]
ERROR: DimensionMismatch("arrays could not be broadcast to a common size; got a dimension with lengths 2 and 3")
```

That does not work as expected. Julia is trying to do an elementwise comparison, but the array **dimensions do not match**. Unlike R or `numpy`, Julia does not recycle or copy arrays nor tries to interpret what you mean. You get what you type. Let's look at a broadcasting example that would work

```julia
julia> [1, 2] .∈ [[1, 4], [3, 0]]
2-element BitArray{1}:
 1
 0
 
julia> in.([1, 2], [[1, 4], [3, 0]]) # this alternative does the same
```

Julia tests whether `1` is in `[1, 4]` and whether `2` is in `[3, 0]`. The result is `1` (true) for the first test and `0` for the second one. But this is not what we are trying to replicate. It turns out that there is no built-in function to replicate the behavior of `numpy` and R. Again, we need a **list comprehension**, which is also a familiar feature of Python:

```Julia
julia> [i in [1, 2, 3] for i in [1, 2]]
2-element Array{Bool,1}:
 1
 1
```

This is the exact result we got from R and `numpy`. However, the type is now different: `Array{Bool,1}` instead of `BitArray{1}`. This is not a major issue. They are both subtypes of `AbstractArray{Bool}` and are mostly interchangeable. They only differ in how they store the boolean value.

We now turn to the next major topic: how to do test membership for strings in Julia.

### Strings: `occursin` in Julia

Membership tests can be done with the `in` function for a wide array of types. However, strings their own specialized function: `occursin`. There are two reasons for this. First, strings are *not* iterables in Julia, unlike Python. Second, `occursin` has functionality specific to strings. Third, testing for membership in a collection is not necessarily the same as testing a substring or a regular expression. Let's take a look at the method signature:

```julia
occursin(needle::Union{AbstractString,Regex,AbstractChar}, haystack::AbstractString)
```

Basically, `occursin` finds a 'needle' in the 'haystack'. The needle can be either a subtype of `AbstractString` or `AbstractCharacter`, but *also* a regular expression. 

```julia
julia> occursin("a", "abc") # is the needle a substring of the haystack?
true

julia> occursin("ab", "abc") # the needle string can be arbitrary
true

julia> occursin('a', "abc") # when needle is a character it is the same as in('a', in "abc")
true

julia> occursin(r"a.c", "abc") # using a regular expression
true

```

It is very neat that there is a built-in function for doing substring tests that also allows you to check **regular expressions** without using a regular expressions library. Moreover, we can use both `in` and `occursin` to test character membership in a string. However, we can check with the `@code_native` macro that the implementations of both functions are different. 

```julia
julia> @code_native in('b', "abc")
...
julia> @code_native occursin('b', "abc")
...
```

The output is omitted, but one key difference is that `occursin` does not use the `findfirst` function that we showed earlier for the `in` function. 

## 4. Conclusion

In this post, looked at membership tests in Julia, Python and R. This is a very common operation when programming. However, even with simple functions it is worth looking at the details, as it reveals a lot about design choices of a programming language. Personally, I think that Julia has a much more consistent an modern feel to it. Python is great, but the differences between Python, `numpy`, `pandas`, `torch`, etc. can be frustrating. As for R, it is a bag of surprises. The language is much older and does not always match the intuition you develop with a more modern programming langugae. More often than not you have to try something out, and see if it works as expected!

Follow me on Twitter [@mexiamorelli](https://twitter.com/mexiamorelli)