# Transitive Closure

**Due**: Tuesday, February 27, 11:59PM.

Available on http://autograde.org. Use your credentials to login.

In this project we will implement a domain-specific programming
language that allows us to specify network topologies and check their
connectivity. Production datacenters and commercial networks are
composed of myriad hosts, routers, and switches, connected in a
complex web of infrastructure. An important but challenging question
is simply: which nodes may connect to which others?  Even in the
modern age, network misconfigurations still account for [global
internet outage
events](https://www.forbes.com/sites/forbestechcouncil/2022/01/27/whats-causing-all-these-network-outages-and-what-can-cios-do-to-prevent-them/?sh=6dc268d37e7b). To
address this, several cloud providers have begun to provide tools for
analyzing properties of networks specified via domain-specific
configuration languages. For example, AWS's [VPC Reachability
Analyzer](https://aws.amazon.com/blogs/aws/new-vpc-insights-analyzes-reachability-and-visibility-in-vpcs/)
is a commercial offering that is now used by some cloud system
engineers as part of their day-to-day jobs.

In this project, we will build a minimal language for expressing
network topologies and implement transitive closure, the algorithm for
graph reachability. This will allow us to check for connectivity
properties of (potentially huge) graphs. Our language is loosely
inspired by [NetKAT](https://cornell-pl.github.io/cs6114/netkat.html),
and we encourage you to read about its features if software-defined
networking is of interest to you.

### Academic Integrity

The coding on this project is to be completed by you alone, without
help from any other students. You are encouraged to discuss the
project specification at a high level, but should not discuss
specifics or show students your solution code.

## Input Format

Networks include various pieces of physical infrastructure: routers,
switches, and servers. In this project we will ignore the
physical properties of network nodes, and will simply call every
entity capable of sending or receiving data a "node." Nodes are
connected by (directed) links. Both nodes and links are specified as
commands, each command residing on a unique line of the input file
format. Syntactically, the two specification commands are (a) the
`NODE <name>` command, which specifies the existence of a node named
`<name>` and (b) the `LINK <from> <to>` command, which establishes a
(directed) link from node `<from>` to node `<to>`. For convenience, we
assume all nodes are specified before any occurrences of `LINK`
commands. For example, the following is valid:

```
NODE node0-name
NODE node1-name
LINK node0-name node1-name
```

But the next is not:

```
NODE node0-name
LINK node0-name node1-name
NODE node1-name
```

We can visualize the first graph in the following way:

```
    +------------+  link    +------------+
    | node0-name | -------> | node1-name |
    +------------+          +------------+
```

Let's add some more nodes and links:

```
NODE node0-name
NODE node1-name
NODE node2-name
LINK node0-name node1-name
LINK node1-name node2-name
```

Now our network looks like this:

```
    +------------+  link    +------------+  link    +------------+
    | node0-name | -------> | node1-name | -------> | node2-name |
    +------------+          +------------+          +------------+
```

# Transitive Closure

The above networks are toy examples. In practice, networks may have
millions of nodes, and orders-of-magnitude more links. Network
engineers work hard to ensure that networks are designed to adhere to
specific connectivity invariants. For example, we may wish to ensure
that `main-server` can connect to `backup-srv`:

```
NODE main-server
NODE backup-srv
NODE switch
NODE db-server
LINK main-server switch
LINK switch main-server
LINK switch backup-srv
LINK backup-srv switch
LINK db-server switch
LINK switch db-server
```

Note that we explicitly enumerated bidirectional links. The above
specification may be visualized as:

```
    +-------------+          +------------+          +------------+
    | main-server | <------> |   switch   | <------> | backup-srv |
    +-------------+          +------------+          +------------+
                                 ^
    +------------+               |
    | db-server  | <-------------+ 
    +------------+
```

Now, we can make queries:

```
CONNECT main-server backup-srv
CONNECT db-server main-server
```

Checking these queries is easy for small examples: we can trace in our
head a path between `main-server` and `db-server`. But larger examples
may involve a great many nodes, and even more links between them. As
such, we need an algorithm which will infer *transitive* links through
the input graph:

```
          +----------------------------------------------+
          v                                              v
    +-------------+          +------------+          +------------+
    | main-server | <------> |   switch   | <------> | backup-srv |
    +-------------+          +------------+          +------------+
          |                      ^                        ^
    +------------+               |                        |
    | db-server  | <-------------+                        |
    +------------+                                        |
          ^                                               |
          +-----------------------------------------------+
```

We do this through an iterative process named *transitive
closure*. Transitive closure consists of a series of steps applied in
an iterative fashion, until no more answers are possible. In other
words, the algorithm is defined in terms of its behavior at each "time
step."

- The transitive closure (of links) at time 0 is simply the set of
  extensionally-specified (input) links.

- To construct the transitive closure at time n+1, look at the
  transitive closure at time n. For any pair of links in that graph,
  `(x,y)` and `(y,z)`, such that the intermediate node `y` is
  matching, draw a (possibly new) link between `x` and `z`, `(x,z)`.

- Repeat this process until no new links are found. I.e., until the
  transitive closure at some time n is equal to the transitive closure
  at time n+1

This process necessarily terminates, as long as you are careful to
ensure elements are not added twice (which will be easy using the
correct datastructures, namely sets and hashes). This is because there
are a finite number of nodes, and so the "worst-case" scenario would
be that every node was connected to every other (as it is in the above
graph). At each step of the process, we add more information to
(monotonically) increase our knowledge base (of transitive links). At
some point we will either (a) not add any new links or (b) add all
possible links, at which point no more links may be added and we
terminate.

A generalized version of this reasoning gives us the [Knaster-Tarski fixed-point theorem](https://en.wikipedia.org/wiki/Knaster%E2%80%93Tarski_theorem) and the [Klenne fixed-point theorem](https://en.wikipedia.org/wiki/Kleene_fixed-point_theorem). 

# Hashes

We will represent graphs of links as Racket `hashes`. We strongly
encourage you to read [their documentation](https://docs.racket-lang.org/reference/hashtables.html). You
will need to know at least the following key functions: `(hash key0 value0
...)`, `(hash-ref key)`, `(hash-set hsh key value)`, and `(hash-keys
hsh)`. Note: you are specifically **forbidden** from using
`make-hash`, `hash-set!`, and other mutable variants of hashes.

To represent a graph as a hash, we will adopt the following
representation: keys will be strings, and their associated values will
be **sets** of strings, manipulated using Racket's `set`s (whose
[documentation you should read
here](https://docs.racket-lang.org/reference/sets.html). For example,
the first example graph would be represented as

```
(define x (hash "node0-name" (set "node1-name")
                "node1-name" (set "node2-name")))
```

To add a new link to the graph, we must be careful to use `set-add` to
extend the set of nodes. For example, consider that we wanted to
*extend* the above graph with a pointer from `node0-name` to
`node3-name`, we would do it via the following:

```
(hash-set x "node0-name" (set-add (hash-ref x "node0-name") "node3-name"))
```

Make sure you understand in the above code how `set-add` and
`hash-ref` are used in combination to ensure that no
previously-present nodes are dropped.

# Tasks

You will implement several functions, I have ranked them roughly in
order of difficulty.

- `(parse-line l)` -- You will parse an input line given as a string
  and you will transform it into an output that conforms to
  `line?`. Hint: use `string-split` and matching. (Difficulty:
  easy/medium)

- `(forward-link? graph n0 n1)` -- Check whether there is a forward
  link from `n0` to `n1` in the graph `graph`. Return `#t` iff `n1` is
  linked to from (pointed at by) `n0`. (Difficulty: easier)

- `(add-link graph from to)` -- Add an "edge" in the graph from node 
  `from` to node `to`. Hint: use `hash-ref`, `set-add`, and `hash-set` 
  to accomplish this. (Difficulty: easier)

- `(build-init-graph input)` -- Assume that `input` is a program given
  as input. You will build up an *initial* graph datastructure
  corresponding to the program. Essentially, you are building an
  initial graph upon which you will subsequently perform iterative
  rounds of transitive closure. You will do this by looking at each
  line of input and changing the hash in one of two ways:

    -> If you see a `(node <n>)` command, you will add a self link
    between `n` and itself.
    -> If you see an `(link <from> <to>)` command, you will insert an
    edge from `from` to `to`.

  (Difficulty: medium)

- `(transitive-closure graph)` -- Perform the transitive closure of
  the graph `graph`. Your solution must be the final answer of
  transitive closure, not just a single iteration. In other words,
  your solution must possess the property that there are no additional
  links possible to add. (Difficulty: harder)

### Implementing Transitive Closure

There are a variety of algorithms to implement transitive closure. I
will describe the so-called "naive" approach here, and then sketch a
more refined (semi-naive) approach. I would recommend you implement
your code using a combination of either recursive functions or the
functions `foldl/r`. My solution is roughly ten lines and uses `foldl`
three times.

Here is an iterative algorithm for calculating the transitive closure
of a graph.

- Proceeding one iteration at a time, accumulate a variable `solution`

- At time step zero, initialize `solution` to be the graph of initial
  links.

- To obtain the next time step, iterate over each edge in `solution`,
  `(x,y)` (this can be done using `hash-keys` and a recursive function
  or `foldl`)

   - For each of these edges, `(x,y)` identify the set of edges
   starting from `y`, `(y,z)`, add `(x,z)` to the graph (this can be
   done using `hash-ref`, which will return a set: you can iterate
   over that set as a list by using `set->list`). Hint: you may want
   to call `hash-ref` with a third argument to specify a "default"
   value of the empty set (e.g., `(hash-ref h key (set))`).

 - At some point this process will add no new edges, and `solution` at
   timestep `n` will be the same as `solution` at timestamp
   `n+1`. When this happens, the search is over.

As an example, let's see what happens on the following

```
NODE node0
NODE node1
NODE node2
LINK node0 node1
LINK node1 node2
```

- We start with `solution` being the graph `{node0 -> {node1}, node1 -> {node2}}`

- We examine edge `(node0,node1)`:
  
  - Consider the set of edges beginning with `node1`: this is just the
    single edge `(node1,node2)`. We add it to the graph, and now our
    graph is `{node0 -> {node1, node2}, node1 -> {node2}}`

- We examine edge `(node1,node2)`:
  
  - Consider the set of edges beginning with `node2`: there are no
    such edges, so the graph doesn't change.
  
- At the end of the first iteration (time step), the graph is `{node0
  -> {node1, node2}, node1 -> {node2}}`.

- During the next iteration, we reexamine `node0` and `node1`, but we
  don't discover any new edges, so the graph remains the same.

- We're done: the final answer is `{node0 -> {node1, node2}, node1 -> {node2}}`

All in all my reference solution uses roughly 14 lines of code and
uses `foldl` several times. I predict reasonable solutions will be
between 10 and 50 lines, if you find yourself doing significantly more
work please consult the instructors to ensure you've got the algorithm
down.

# Bonus Tests

There are three bonus tests: `bonus-add-edge-3`,
`bonus-transitive-closure-ring100`, and
`bonus-transitive-closure-ring200`. The difference between these tests
and the normal (secret) tests is that they operate over much larger
graphs (the last two are ring graphs of size 200 and 300). To pass
these tests, ensure you don't add extra algorithmic overhead to your
solution. The normal secret tests use graphs up to 30 nodes in
size. In sum, you can earn roughly 18% bonus.

# Testing

Once you implement all of your work, you will unlock the ability to 
run `connectivity.rkt` with an input file. The `demo` function canonicalizes 
the database and prints out the starting database and its transitive closure:

```
racket connectivity.rkt demo/1.net "CONNECTED n13 n52"
```

This will allow you to see your code in action. To run the testing
infrastructure on your code, use `tester.py`. It is invoked as
follows:

```
python3 tester.py -av
```

You may wish to add more example networks in the `demos` folder.

# Submitting your code for testing

**NOTE** Before you can submit your project for grading you *must* git
add, commit, and push. On a terminal (in your project directory)
type:

```
# Add all files in the directory
git add .
# Make a commit
git commit -m "my commit message here"
# Push to server
git push
```

Once you have done a git commit and push, go to the autograder and
select for your project to be graded.
