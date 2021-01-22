---
title: The golang for-loop gotcha
layout: post
tags:
    - golang
keywords:
    - golang
---

This bit me **again** the other day so what better way to exorcize the
resulting bug in production code than to write a small article about
it?

So, let's raise hands: who among us **has not** written code that
looks somewhat like this trivial example:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	for i := 0; i<5; i++ {
		go func() {
			fmt.Print(i)
		}()
	}
    // let's wait a little bit for the goroutines to execute before we exit
	time.Sleep(100 * time.Millisecond)
}
```

*(very few hands raised in the audience)* :grin:

Now, one reasonable assumption would be that this program will print
the numbers from 1 to 5.

This assumption would be wrong though; the program consistently prints
the string `55555` (if the result is different on your computer, which
shouldn't be, then try increasing the sleep duration at the end).

Why is that? The reason is that all goroutines close over the **same**
variable (`i` in this case) so, when they are executed, the variable
will have assumed its last value after the completion of the iteration
loop (which is `5`).

We can observe the difference in behaviour when we introduce a small
delay at the end of each iteration like so:

```go
    ...
    // data race
	for i := 0; i<5; i++ {
		go func() {
			fmt.Print(i)
		}()
		time.Sleep(50 * time.Millisecond)
	}
    ...
```

The output in this case would be `01234` because each goroutine would
most probably be executed before the value of the shared loop variable
`i` was incremented by one.

In essence, this data race (which can lead to wicked and hard-to-find
bugs) is caused by the unsafe **sharing of mutable state** (in this
case `i`) among the competing concurrent goroutines and is well
described on the interwebs (this
[article](https://eli.thegreenplace.net/2019/go-internals-capturing-loop-variables-in-closures/)
for example has an excellent explanation). The official Golang wiki
even lists this situation as [one of the two most common golang
mistakes](https://github.com/golang/go/wiki/CommonMistakes) that one
can make (hint: the other mistake also has to do with `for` loop
variables).

So, what is the correct way of implementing the use case of
 asynchronously processing a series of values in a `for` loop using
 goroutines? The answer is simply to stop sharing the mutable state,
 i.e. copy the state variable (in this case `i`) within each iteration
 and pass it to each goroutine like so:

```go
    ...
    // no data race
	for i := 0; i<5; i++ {
		go func(n int) {
			fmt.Print(n)
		}(i)
	}
    ...
```

In this implementation, we are passing `i` **by value** as an argument
(`n`) into the goroutine; `n` is now a goroutine-local copy of `i` and
further changes to `i` are not reflected on `n`.

Alternatively, we could close over an iteration-local **copy** of `i`
as follows:

```go
    ...
    // no data race
	for i := 0; i<5; i++ {
        n := i
		go func() {
			fmt.Print(n)
		}()
	}
    ...
```

It is perhaps worth pointing out that Golang's `for` loop variable
gotcha is not some idiosyncratic language runtime misbehaviour that is
specific to the `for` loop but, rather, a classic example of failing
to synchronize concurrent access to a shared resource either by
locking it or copying it (our preferred solution). Here's an example
of the same problem without a `for` loop:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	n := 10
    // data race
	go func() {
		time.Sleep(100 * time.Millisecond)
		fmt.Print(n)
	}()
	n = 11
	time.Sleep(200 * time.Millisecond)
}
```

This will print `11` instead of `10`.

So, when closing over an outer variable in something like a
gouroutine, a good tip would be to always think: who can access the
outer variable and if/how access needs to be synchronized. Passing a
copy of the state is always a safe choice!
