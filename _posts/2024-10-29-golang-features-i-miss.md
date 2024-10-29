---
title: Golang features I (used to) miss
categories: [golang, missing features, iterator, slices]
---

About 7 years ago, I was faced with some projects that implemented APIs that had
strict performance requirements-not so strict that a language with a garbage
collector was out of the question, but strict enough that I couldn't use Ruby or
any other language that is orders of magnitude slower than C. JVM languages were
also not a good fit because I needed new instances of the service to spin up
quickly. So I chose Golang as the implementation language because of its speed,
simplicity and the fact that it was a modern and mature language with a good
ecosystem. I also considered Rust, which would fit most of my requirements, but
it wasn't as mature as Golang at the time so I went with the safe bet.

Over that time, I've grown accustomed to the language even though a few things
felt off. Coming to it from Ruby, Scala, Clojure, Python and other languages, I
**can't** say I immediately fell in love with it the way I've
[seen](https://www.reddit.com/r/golang/comments/uzugxy/what_do_you_like_about_go/)
other developers do and put a gopher sticker on my laptop. I've never shaken the
feeling that the way the language forces the developer to write much more
verbose code than I was used to writing in other languages is a problem. Mind
you, this is not about refusing to adjust to language idioms (e.g. the way it
forces the developer to do error handling). I'm talking about features that help
write much less verbose code while still maintaining readability like generics,
collection primitives (e.g. map, filter, collect) and immutable data structures.

On the other hand, I appreciate the philosophy of the language of keeping it
straightforward and avoiding adding features just because there is demand for
them from people like me. The maintainers want to take their time to keep the
language simple, minimal only adding features if they aren't sacrificing these
traits. I can respect that, but in the real world we need to get things done and
there is a limit to how much repetitive code I'm willing to write. Thankfully
they are listening and already implemented
[generics](https://gobyexample.com/generics) and built a
[slices](https://pkg.go.dev/slices) package on top of them so that we don't have
to keep writing a [Max](https://pkg.go.dev/slices#Max) function for every type a
collection could hold. I'm still left with my biggest missing feature: powerful
collection primitives like map, filter, reduce, etc.

Things are changing on that front as well. The [iter](https://pkg.go.dev/iter)
package is a wonderful first step and I'm sure more will come. Similar to how
everyone was writing a custom `Max` function (and now they don't have to) in
their projects, it unlocks ways of writing code that that were previously not
possible. Even if it is "just" [syntactic
sugar](https://go.dev/wiki/RangefuncExperiment#can-range-over-a-function-perform-as-well-as-hand-written-loops).
Let's look at one [recent example](https://pkg.go.dev/slices#Chunk) and how I
was implementing it in my codebase which I can now thankfully stop doing.


## EachSlice

Coming from Ruby, I was used to having a rich tool-set to work with
collections. One example is being able to iterate a collection in chunks of
arbitrary length. This is useful if the collection data that you're working with
contains batches you want to process together. The answer in Ruby is
[each_slice](https://ruby-doc.org/3.3.5/Enumerable.html#method-i-each_slice).
What would it take to implement something like that in Golang? Well, just like
the Max function **without generics**, we can implement something like
[this](https://stackoverflow.com/questions/59494349/how-to-iterate-on-slices-4-items-at-a-time)


```golang
func EachSliceInt(slice []int, step int, f func(int, []int)) {
	for i := 0; i < len(slice); i += step {
		end := i + min(step, len(slice[i:]))

		f(i, slice[i:end:end])
	}
}
```

We can use that [implementation](https://go.dev/play/p/a84QrN6qc7s) with code
like this

```golang
EachSliceInt(
	[]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15},
	4,
	func(i int, s []int) { fmt.Println(i, s) },
)
```

To get output like this

```bash
0 [1 2 3 4]
4 [5 6 7 8]
8 [9 10 11 12]
12 [13 14 15]
```

The problem with that is that for each type a slice could hold, we'd have have a
separate implementation. Let's make the implementation
[generic](https://go.dev/play/p/JvvyLuNwdAT):

```golang
func EachSlice[S ~[]E, E any](slice S, step int, f func(int, S)) {
	for i := 0; i < len(slice); i += step {
		end := i + min(step, len(slice[i:]))

		f(i, slice[i:end:end])
	}
}
```

Now we can use it with any type of slice

```golang
EachSlice(
	[]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15},
	4,
	func(i int, s []int) { fmt.Println(i, s) },
)

EachSlice(
	[]string{
		"one", "two", "three", "four", "five",
		"six", "seven", "eight", "nine", "ten",
		"eleven", "twelve", "thirteen", "fourteen", "fifteen",
	},
	4,
	func(i int, s []string) { fmt.Println(i, s) },
)
```

To get output like this


```bash
0 [1 2 3 4]
4 [5 6 7 8]
8 [9 10 11 12]
12 [13 14 15]
0 [one two three four]
4 [five six seven eight]
8 [nine ten eleven twelve]
12 [thirteen fourteen fifteen]
```

Now for the final touch let's make it
[possible](https://go.dev/play/p/G7EvR-aM9lT) to use the function in a
[for-range](https://go.dev/blog/range-functions)
loop.

```golang
func EachSlice[S ~[]E, E any](slice S, step int) iter.Seq2[int, S] {
	return func(yield func(index int, slice S) bool) {
		for i := 0; i < len(slice); i += step {
			end := i + min(step, len(slice[i:]))

			if !yield(i, slice[i:end:end]) {
				return
			}
		}
	}
}
```

This last implementation allows us to use code like this

```golang
for c := range EachSlice(
    []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15},
    4,
) {
    fmt.Println(i, c)
}
```

Which works the same as before

```bash
0 [1 2 3 4]
4 [5 6 7 8]
8 [9 10 11 12]
12 [13 14 15]
```

What we've done in the last example is that we've (re)implemented the
[slices.Chunk](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/slices/iter.go;l=92)
function. We added an index to make is possible for the client code to know
which chunk index is currently being processed, but other than that we could
use the builtin function as is. I always wanted rich tools to deal with
collections to appear in the standard library sometime and (just like with the
Max function) they slowly are with the advent of the [slices
package](https://pkg.go.dev/slices) and the [maps package](https://pkg.go.dev/maps).

There are even more interesting [things
happening](https://github.com/golang/go/issues/61898) to make working with
collections in Golang a more pleasant experience. The generics implementation
is still limited in that it doesn't support [method type
parameters](https://github.com/golang/go/issues/49085) so code like
`myList.Backward().Filter(func(i int) bool { return i % 3 == 0}).Contains(42)`
will still not be possible, but even with the limitation, the code with these recent
changes will be much less verbose and contain a lot less boilerplate while still
having having same Golang feel of simplicity over complexity. That's a good
direction to be heading in and I'm looking forward to seeing how the language
evolves.
