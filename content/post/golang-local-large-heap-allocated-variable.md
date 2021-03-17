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
 
I guess most go developers (including me ^) will be curious about how `large` is large that a local variable will be allocated to heap. For Go 1.15, the answer is `10MiB`. See [gc/go.go](https://github.com/golang/go/blob/release-branch.go1.15/src/cmd/compile/internal/gc/go.go#L19)
 
{{< highlight terraform >}}
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
 
{{< highlight terraform >}}
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
 
`n *Node` is one node of [Syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree). We will only focus on the variable size factor for each node in this post. `Slice` is also not considered in this post either.
 
Combining the two code blocks, a variable will be moved to heap when its type width is larger than `maxStackVarSize(10MiB)`, or its type's element width is larger than `maxImplicitStackVarSize(64kb)`.
 
Now it's time to write some code to verify this. Consider we have a file `case1.go` with code like below:
 
{{< highlight terraform >}}
package main
 
type employer1 struct {
 Name  string
 Age   int
 Title string
}
 
//go:noinline
func getEmployer1Array() [262144]employer1 {
 var emps [262144]employer1
 for i := 0; i < 262144; i++ {
   emps[i] = employer1{
     Name:  "adam",
     Age:   23,
     Title: "ceo",
   }
 }
 return emps
}
 
func main() {}
{{< /highlight >}}
 
Type `employer1` width is 40 byte in my machine(amd64), containing 2 strings(16 bytes each), 1 int(8 bytes). Array size `262,144` will make `var emps` to be 10MiB large. I.e, `n.Type.Width` is 10MiB, and `n.Type.Elem().Width` is 40Byte.
 
Then we can run command:
 
{{< highlight text >}}
$ go build -gcflags="-m -l" case1.go
{{< /highlight >}}
 
Result is empty:
 
{{< highlight text >}}
smiletrl@Rulins-MacBook-Pro example4 % go build -gcflags="-m -l"
{{< /highlight >}}
 
It means no variable has been moved to heap yet.
 
Now lets increase the array size by 1, and we have:
 
{{< highlight golang >}}
package main
 
type employer1 struct {
 Name  string
 Age   int
 Title string
}
 
//go:noinline
func getEmployer1Array() [262145]employer1 {
 var emps [262145]employer1
 for i := 0; i < 262145; i++ {
   emps[i] = employer1{
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
 
{{< highlight golang >}}
smiletrl@Rulins-MacBook-Pro example2 % go build -gcflags="-m -l"
# github.com/smiletrl/golang_escape/pkg/example2
./case1.go:11:6: moved to heap: emps
{{< /highlight >}}
 
Cool! We see the variable `var emps` has been moved to heap!
 
## Conclusion
 
Now that we know the exact size of a local `large` heap allocated variable, we can be sure of writing code with this size in mind, and try to keep a variable in a reasonable size to stay within the stack.
 
It's not saying we can allocate as many 10MiB local variables in stack as we want. Stack for each Goroutine has its max size limit as well. We may talk about it in a separate post.
 
For previous Go versions, we may use a similar way to find the exact size, and for later versions, change has happened. We will have a separate post :)
 
You may find more explanation and escape examples at https://github.com/smiletrl/golang_escape.
 