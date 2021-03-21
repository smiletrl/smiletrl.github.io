title = "How large is large enough to allocate a local variable in heap for Golang"

+++
author = "Rulin Tang"
title = "How large is large enough to allocate a local variable to heap in Golang"
date = "2021-03-16"
description = "local large heap allocated variable in golang"
featured = true
tags = [
   "heap",
   "featured"
]
categories = [
  "Escape analysis",
]
series = ["Golang"]
aliases = ["golang-allocate-large-local-variable-to-heap"]
thumbnail = "images/building.png"
+++
 
This post shows how a local variable will be `large` enough to be allocated to heap.
<!--more-->
## Stack or heap
Stack is usually more efficient for variable allocation than heap. There're plenty of articles online explaining why stack allocation is much faster, especially for Golang. So I'm not going to talk about the difference between stack and heap in the article.
According to [Golang FAQ](https://golang.org/doc/faq#stack_or_heap),
>  Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
I guess most go developers (including me ^) will be curious about how `large` is large that a local variable will be allocated to heap. For Go version 1.5 - 1.16, the answer is `10MiB`. This is decided by golang [escape analysis](https://github.com/golang/go/wiki/CompilerOptimizations#escape-analysis). See [gc/go.go](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/go.go#L19). This was [defined](https://go-review.googlesource.com/c/go/+/4851/3/src/cmd/internal/gc/go.go#56) by the new Go Parser in Go 1.5.
{{< highlight golang >}}
var (
// maximum size variable which we will allocate on the stack.
// This limit is for explicit variable declarations like "var x T" or "x := ...".
// Note: the flag smallframes can update this value.
maxStackVarSize = int64(10 * 1024 * 1024)
// maximum size of implicit variables that we will allocate on the stack.
//   p := new(T)          allocating T on the stack
//   p := &T{}            allocating T on the stack
//   s := make([]T, n)    allocating [n]T on the stack
//   s := []byte("...")   allocating [n]byte on the stack
// Note: the flag smallframes can update this value.
maxImplicitStackVarSize = int64(64 * 1024)
)
{{< /highlight >}}
and [gc/esc.go](https://github.com/golang/go/blob/release-branch.go1.15/src/cmd/compile/internal/gc/esc.go#L172)
{{< highlight golang >}}
func mustHeapAlloc(n *Node) bool {
if n.Type == nil {
  return false
}
// Parameters are always passed via the stack.
if n.Op == ONAME && (n.Class() == PPARAM || n.Class() == PPARAMOUT) {
  return false
}
if n.Type.Width > maxStackVarSize {
  return true
}
if (n.Op == ONEW || n.Op == OPTRLIT) && n.Type.Elem().Width >= maxImplicitStackVarSize {
  return true
}
if n.Op == OMAKESLICE && !isSmallMakeSlice(n) {
  return true
}
return false
}
{{< /highlight >}}
`n *Node` is one node of [Syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree). We will only focus on the variable size factor for each node in this post. `Slice` will have more factors, including this size factor.
Combining the two code blocks, a variable will be moved to heap when its type width is larger than `maxStackVarSize(10MiB)`, or its type's element width is larger than (or equal) `maxImplicitStackVarSize(64kb)`.
 
Now it's time to write some code to verify this. Consider we have a file `case.go` with code like below:
{{< highlight golang >}}
package main
type employer struct {
Name  string
Age   int
Title string
}
//go:noinline
func getEmployerArray() [262144]employer {
var emps [262144]employer
for i := 0; i < 262144; i++ {
  emps[i] = employer{
    Name:  "adam",
    Age:   23,
    Title: "ceo",
  }
}
return emps
}
func main() {}
{{< /highlight >}}
 
For `var emps [262144]employer`, its type is `emps`, and type element is struct `employer`.  Type `employer` width is 40 byte in my machine(amd64), containing 2 strings(16 bytes each), 1 int(8 bytes). Array size `262,144` will make `var emps` to be 10MiB large. I.e, `n.Type.Width` is 10MiB, and `n.Type.Elem().Width` is 40Byte in this case.
Then we can run command:
{{< highlight text >}}
$ go build -gcflags="-m -l" case.go
{{< /highlight >}}
Result is empty:
{{< highlight text >}}
smiletrl@Rulins-MacBook-Pro example % go build -gcflags="-m -l"
{{< /highlight >}}
It means no variable has been moved to heap yet.
Now lets increase the array size by 1, and we have:
{{< highlight golang >}}
package main
type employer struct {
Name  string
Age   int
Title string
}
//go:noinline
func getEmployerArray() [262145]employer {
var emps [262145]employer
for i := 0; i < 262145; i++ {
  emps[i] = employer{
    Name:  "adam",
    Age:   23,
    Title: "ceo",
  }
}
return emps
}
func main() {}
{{< /highlight >}}
With this change, `n.Type.Width` value becomes `10MiB + 40Byte`. This time, we get:
{{< highlight text >}}
smiletrl@Rulins-MacBook-Pro example % go build -gcflags="-m -l"
# github.com/smiletrl/golang_escape/pkg/example
./case.go:11:6: moved to heap: emps
{{< /highlight >}}
Cool! We see the variable `var emps` has been moved to heap!
 
## Implicit variable size
 
For implicit variable, we get this definition from above research.
 
{{< highlight golang >}}
var (
// maximum size of implicit variables that we will allocate on the stack.
//   p := new(T)          allocating T on the stack
//   p := &T{}            allocating T on the stack
//   s := make([]T, n)    allocating [n]T on the stack
//   s := []byte("...")   allocating [n]byte on the stack
// Note: the flag smallframes can update this value.
maxImplicitStackVarSize = int64(64 * 1024)
)
{{< /highlight >}}
 
The maximum implicit variable size is `65,536Byte = 64*1024Byte`. Now let's try some tests with pattern `p := &T{}`
 
{{< highlight golang >}}
package main
 
import (
 "fmt"
)
 
type employer struct {
 Name  string
 Title string
 Age   int
}
 
type empList struct {
 List [1638]employer
}
 
//go:noinline
func getemployerSlice() {
 emps := &empList{}
 if 1 > 2 {
   fmt.Println(emps)
 }
}
 
func main() {}
{{< /highlight >}}
 
In above code, size of `empList` is `65,520Byte = 40Byte * 1638`, which is smaller than the maximum implicit variable size (65,536). In this case, we expect `var emps := &empList{}` not escaping.
 
{{< highlight golang >}}
smiletrl@Rulins-MacBook-Pro example % go build -gcflags="-m -l" case.go
# command-line-arguments
./case.go:19:10: &empList{} does not escape
{{< /highlight >}}
 
Above result is what we have expected. Now let's increase the size of `empList` to `65,560Byte = 40Byte * 1639`, which is larger than the maximum implicit variable size (65,536). In this case, we expect `var emps := &empList{}` escaping.
 
{{< highlight golang >}}
package main
 
import (
 "fmt"
)
 
type employer struct {
 Name  string
 Title string
 Age   int
}
 
type empList struct {
 List [1639]employer // increase this size!
}
 
//go:noinline
func getemployerSlice() {
 emps := &empList{}
 if 1 > 2 {
   fmt.Println(emps)
 }
}
 
func main() {}
{{< /highlight >}}
 
And here's the result:
 
{{< highlight golang >}}
smiletrl@Rulins-MacBook-Pro example % go build -gcflags="-m -l" case.go
# command-line-arguments
./case.go:19:10: &empList{} escapes to heap
{{< /highlight >}}
 
Cool! `&empList` has escaped to the heap as we have expected!
 
## Extra verification of escaping result
 
If you are curious about the escape result, here's a bench test approach to verify the result. We are going to write a simple benchmark test to inspect the heap allocation.
 
We will create a test file called `case_test.go` for above implicit variable example with  `empList` size being `65,560Byte = 40Byte * 1639`.
 
{{< highlight golang >}}
package main
 
import (
 "testing"
)
 
func BenchmarkSlice(b *testing.B) {
 for n := 0; n < b.N; n++ {
   getemployerSlice()
 }
}
{{< /highlight >}}
 
Now let's run the test:
 
{{< highlight text >}}
smiletrl@Rulins-MacBook-Pro example % go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: github.com/smiletrl/golang_escape/pkg/example
cpu: Intel(R) Core(TM) i9-9980HK CPU @ 2.40GHz
BenchmarkSlice-16       139160        8475 ns/op     73728 B/op        1 allocs/op
PASS
ok    github.com/smiletrl/golang_escape/pkg/example 3.408s
{{< /highlight >}}
Above result shows one heap allocation with `1 allocs/op`, and it has allocated `73728Byte` for each op, which is a bit larger than `empList`'s size. Heap allocation is going to take a bit more memory space than the real variable size for some meta information. You may find more explanation for the [benchmark result here](https://golang.org/pkg/testing/#BenchmarkResult).
 
Feel free to do some exercise for smaller implicit variable size, and above explicit variable size example.
 
## Conclusion
Now that we know the exact size of a local `large` heap allocated variable, we can be sure of writing code with this size in mind, and try to keep a variable in a `reasonable size` to stay within the stack.
It's not saying we can allocate as many 10MiB (for Go version 1.5 - 1.16) local variables in stack as we want. Stack for each Goroutine has its max size limit as well. We may talk about that in a separate post.
You may find more explanation and escape examples at https://github.com/smiletrl/golang_escape.
 