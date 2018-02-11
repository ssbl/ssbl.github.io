---
layout: post
title: "tail recursion optimization in c"
date: 2018-02-10 17:00:43 -0500
comments: true
categories:
- c
- recursion
- compilers
---

I recently came across a recursive binary search implementation in the
source code of [ag](https://github.com/ggreer/the_silver_searcher):

```c
int binary_search(const char *needle,
                  char **haystack,
                  int start,
                  int end) {
  int mid;
  int rc;

  if (start == end) {
    return -1;
  }

  mid = start + ((end - start) / 2);

  rc = strcmp(needle, haystack[mid]);
  if (rc < 0) {
    return binary_search(needle, haystack, start, mid);
  } else if (rc > 0) {
    return binary_search(needle, haystack, mid + 1, end);
  }

  return mid;
}
```

My first thoughts were: why would one want to incur the cost of using
recursion when we could use the less-expensive iterative version
instead? But I figured that this must have occurred to the author of
the code as well, so I decided to look deeper.

Tail calls
---

The fact that shed the most light on this issue was that the function
called itself using tail calls: function calls that appear as the last
statement in the function body. Tail calls are optimized by default in
functional languages such as Scheme and Haskell, but C is a different
beast. I could only vaguely remember this quiz on
[ridiculousfish's blog](
http://ridiculousfish.com/blog/posts/will-it-optimize.html), which
shows how a recursive factorial function is optimized by gcc:

```c
int factorial(int x) {
  if (x > 1) return x * factorial(x-1);
  else return 1;
}
```

Using gcc or clang with optimizations turned on, the call to
`factorial` will be converted to a loop:

```c
int factorial(int x) {
  int result = 1;
  while (x > 1) result *= x--;
  return result;
}
```

It turns out that this is exactly what happens with our
`binary_search` function in ag! With optimizations turned on, this
is roughly what it compiles to:

```c
int binary_search(char *needle,
                  char **haystack,
                  int start,
                  int end) {
  while (start != end) {
    int mid = start + (end - start) / 2;
    int rc = strcmp(needle, haystack[mid]);

    if (rc < 0) {
      end = mid;
    } else if (rc > 0) {
      start = mid + 1;
    } else {
      return mid;
    }
  }

  return -1;
}
```

A note on the [generated assembly](https://godbolt.org/g/3JJymH): the
generated code first computes `end - start` to check the loop
condition, and then *reuses* this result to compute the value of
`mid`. *Really* cool.

When can the compiler optimize tail calls?
---

This type of optimization can be applied even if the recursive call
does not appear as the last statement in the function. In such a case,
the compiler checks if all statements after the call can be moved
before the call.

Moreover, tail recursion elimination is a subset of tail call
optimization (TCO) [^1], which deals with arbitrary tail calls to any
type of functions. TCO is more involved in C, unlike Scheme, because
the called function might access variables that are on the calling
function's stack. One of the benefits of eliminating tail calls is
that stack frames can be reused. This in turn means that we need to
guarantee that all variables on the caller's stack can be safely
deallocated before the tail call. This is not possible when the called
function accesses variables on the caller's stack.

Lessons learned
---

This little investigation has provided me with a more nuanced view
of recursive functions. Perhaps, before jumping to reason about
stack blowups, I might first check: is the function tail recursive?

[^1]: Going by various sources online, this is also referred to simply as an "efficient tail call implementation".