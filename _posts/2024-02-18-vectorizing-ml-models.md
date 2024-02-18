---
title: Vectorizing ML models for fun
layout: post
description: Auto-vectorizing ML models using union-find.
date: 2024-02-18
---

Hello everyone! I'm back. I wasn't satisfied with adding a `Dot` operator to
micrograd and manually using it in the MLP implementation from the [last
post](/blog/compiling-ml-models). I kept wondering if it was possible to add
that to the graph automatically using an optimizer. So I did just that.

Forget about all my changes to micrograd: we're going to start from a clean
micrograd and talk about autovectorization. The idea isn't new, but this seems
to be a simple enough and small enough language that it is easy instead of very
difficult.

Remember the `Value` class? That was the main node type in the graph
intermediate representation. It works on scalars (numbers) instead of bigger
structures like vectors. But Andrej is doing tensor math---dot products, matrix
multiplications. He's just encoding it at the lowest level---the scalar level.
We're going to try and find that high-level structure and lift it out.

## Union-find

To do that we're going to use the union-find/disjoint-set data structure,
which, if you don't think about it too hard, is really simple (proving it fast
is a different matter entirely). If you are a programming languages person and
haven't used it much, I recommend taking a look at CF's [Implementing a Toy
Optimizer](https://www.pypy.org/posts/2022/07/toy-optimizer.html). That was
what a) made it stick, b) made it simple, and c) got me hooked.

This is, regrettably, an invasive change we will have to make to the `Value`
class. I think we could do it with a data structure on the side, but maybe I
will leave that as an exercise for the reader.

```diff
diff --git a/micrograd/engine.py b/micrograd/engine.py
index afd82cc..1c1863a 100644
--- a/micrograd/engine.py
+++ b/micrograd/engine.py
@@ -7,8 +7,33 @@ class Value:
         self.grad = 0
         # internal variables used for autograd graph construction
         self._backward = lambda: None
-        self._prev = set(_children)
+        self._prev = tuple(_children)
         self._op = _op # the op that produced this node, for graphviz / debugging / etc
+        self.forwarded = None
+
+    def find(self):
+        op = self
+        while isinstance(op, Value):
+            next = op.forwarded
+            if next is None:
+                return op
+            op = next
+        return op
+
+    def args(self):
+        return [v.find() for v in self._prev]
+
+    def arg(self, idx):
+        return self._prev[idx].find()
+
+    def make_equal_to(self, other):
+        self.find().set_forwarded(other)
+
+    def set_forwarded(self, other):
+        if self._op == '':
+            assert self.data == other.data
+        else:
+            self.forwarded = other
```

The important parts are adding a `forwarded` field to the object, the `find`
function, and the `make_equal_to` function. Forcing an order for the `_prev`
isn't really necessary but it makes debugging easier.

Union-find is a data structure with two operations:

* What set is this object in?
* Add this object to a set

The funky part is that it's not exactly objects and sets. When you talk about
an object, you also talk about the entire set it's a part of and you are
implicitly speaking about the "representative value" for that set. So really,
the two operations are:

* What set is this object in?
* Merge these two sets

And when you think of the sets as equivalence classes for operations---ops
that, when run, would produce the same value---you can start to see its value
in optimizing an SSA/SSI IR. You can optimize a node in the graph without
rewriting all its uses.

The main cognitive difference when using union-find in an optimizer is that
when you are looking at an object, you need to make sure you are looking at the
representative of the equivalence class---you need to call `.find()`.

## Current state of scalar math

Remember that `dot(a, b) = a[0]*b[0] + ... + a[n]*b[n]`. Unfortunately, we
don't have that kind of many-argument `+` explicitly encoded in the scalar IR.
What we have looks more like:

```
v0 = input
v3 = input
v6 = * v0 v3
v10 = + v6 0
v1 = input
v4 = input
v7 = * v1 v4
v11 = + v10 v7
v2 = input
v5 = input
v8 = * v2 v5
v12 = + v11 v8
```

It's this deeply-nested tree structure instead of a shallow one. Where's the
dot product? I don't see anything that looks like a long series of adds...
Ideally instead we would have:

```
v0 = input
v3 = input
v6 = * v0 v3
v1 = input
v4 = input
v7 = * v1 v4
v2 = input
v5 = input
v8 = * v2 v5
v15 = + v6 v7 v8
```

Where all the `*` nodes have been brought together as siblings in the `+` node.
Now finding a dot product looks a lot easier. You can see there's a wide `+`
node where all the children are multiplications. Great.

Let's think about how we would do this at the individual node level. In order
to make one `+` out of multiple, the tree must look like:

```
v2 = v0 + v1
v4 = v2 + v3
```

So, a plus made up of other nested plus nodes. If we take the children of `v2`
(`v0` and `v1`) and bring them up to be children of `v4`, we get `v4 = v0 + v1 + v3`.
Neat. And if `v4`'s children aren't all `+`, that's fine; we just leave the
other operations as they are.

```python
def optimize_one(v):
    if v._op == "+":
        args = v.args()
        if any(arg._op == "+" for arg in args):
            new_args = []
            for arg in args:
                if arg._op == "+":
                    new_args.extend(arg.args())
                else:
                    new_args.append(arg)
            v.make_equal_to(Value(0, tuple(new_args), "+"))
```

Remember that `v0` and `v1` might be entire graphs on their own, so this is
only one step of a bigger loop. If `v0` and `v1` are also deeply nested plus
trees, we want to flatten them as well.

That sounds to me like a good opportunity to build up the graph from the
"bottom" (the leaves) up. And what operation do we already have to find that
traversal order? `topo`, of course.

```python
def run_optimize_one(v):
    topo = v.topo()
    for op in topo:
        optimize_one(op.find())
```

If you wanted to write this as a DFS that returned a new copy of the graph you
could totally do that too, but you would construct a lot of copies that you
don't need to make, I think.

Let's check to see if this optimization works. To do that, I use a little
`collections.Counter` to check the distribution of `Value` node types in the
graph.

```python
# Fake MNIST
dim_in = 28 * 28
net = MLP(dim_in, [50, 10])
model = net([Value(i, (), "input") for i in range(dim_in)])
loss = sum(model)  # fake "loss" function to turn the array into a scalar
print(" ", count(loss.find()))
changed = optimize(loss.find())
print(" ", count(loss.find()))
```

And now if we run it and cross our fingers...

```console?prompt=$
$ time /usr/bin/pypy3.8 test.py
  Counter({'': 39761, '+': 39710, '*': 39700, 'input': 784, 'ReLU': 50})
  Counter({'': 39761, '*': 39700, 'input': 784, '+': 51, 'ReLU': 50})
$
```

Alright, that worked extremely well. It looks like the number of `+` nodes went
from 39,000 (39 *thousand*!) to just 51. Fifty-one! And it left all the other
operations in the graph unchanged. Super.

## Finding dot products

Now that we have all these big `+` nodes, we can turn all `+`-of-`*` patterns
into `Dot`. Well, kinda. We have one more step first: `Array`.

Since we're now making vector operations, it makes sense to add a first-class
type for them.

```python
class Array(Value):
    def __init__(self, data):
        super().__init__(0, data, 'array')
```

Unlike the other node types, there is no transformation happening on the data.
It's collecting other nodes together. So there is no forward or backward
happening here.

Alright, let's take a look at the pass to make `Dot`s:

```python
def optimize(v):
    run_optimize_one(v):
    topo = v.find().topo()
    for op in topo:
        args = op.args()
        if op._op == "+" and any(arg._op == "*" for arg in args):
            mul_args = tuple(arg for arg in args if arg._op == "*")
            assert all(len(arg._prev) == 2 for arg in mul_args)
            mul_left = Array(tuple(arg.arg(0) for arg in mul_args))
            mul_right = Array(tuple(arg.arg(1) for arg in mul_args))
            other_args = tuple(arg for arg in args if arg._op != "*")
            op.make_equal_to(Value(0, (Dot(mul_left, mul_right), *other_args), "+"))
```

For each `+`, if it has `*` children, we can optimize. So if it has `*`
children, we partition those out from the rest. Remember that since the MLP
also adds the bias to the long scalarized dot products, we will have some
data `Value` nodes in there too.

We split all the `*` nodes---which are still all binary---into two `Array`s and
make a `Dot` out of those. Then we add all the other non-`*` arguments to it in
a new `+` node.

Now, what we have right now will make a new array everytime someone does a dot
product with `[v0, v1, v2]`. We probably don't want to make a new array for the
same collection of objects; we can have some reusable storage. To do that, we
memoize the array creation; if the tuple of arguments has been seen before,
return the old `Array`.

```python
@functools.lru_cache(maxsize=None)
def hashcons_array(vs):
    return Array(vs)

def optimize(v):
    run_optimize_one(v):
    topo = v.find().topo()
    for op in topo:
        args = op.args()
        if op._op == "+" and any(arg._op == "*" for arg in args):
            # ...
            mul_left = hashcons_array(tuple(arg.arg(0) for arg in mul_args))
            mul_right = hashcons_array(tuple(arg.arg(1) for arg in mul_args))
            # ...
```

We should probably know what we're building, so let's take a look at the fancy
new `Dot` operator:

```python
class Dot(Value):
    def __init__(self, left, right):
        assert len(left._prev) == len(right._prev)
        super().__init__(0, (left, right), 'dot')

        # TODO(max): Figure out a way to compute this automatically using chain
        # rule.
        def _backward():
            left = self._prev[0]
            right = self._prev[1]
            for i in range(left._prev):
                left._prev[i].grad += right._prev[i].data*self.grad
                right._prev[i].grad += left._prev[i].data*self.grad

        self._backward = _backward
```

The only really notable thing is the hand-derived (hopefully correct)
backpropagation function. You can see my little note about that, too. I think
it's probably possible to use this same technique to build more complex
backpropagation functions automatically as we optimize the graph. But I haven't
figured that one out yet.

Again, I ask: does it work? Great question.

```console?prompt=$
$ time /usr/bin/pypy3.8 test.py
  Counter({'': 39761, '+': 39710, '*': 39700, 'input': 784, 'ReLU': 50})
  Counter({'': 39761, 'input': 784, 'array': 53, 'dot': 51, '+': 51, 'ReLU': 50})
$
```

All the `+` and `*` went away! 39,000 of each turned to just 51 dot products
and handful of adds (for the bias).

## A note

Remember to think very hard if your machine learning is actually going to help
people instead of being a solution search of a problem, or worse, hurt people.

## Next steps

Can we turn these `Dot` into `Matmul`s? What about automatically deriving
`_backward` functions?

What about scheduling? Sure, we have all these vector operations now, but no
CPU actually supports that natively. We have to encode it as x86\_64 `dppd`
instructions or something. Maybe e-graphs would be fun here to optimally
schedule them.