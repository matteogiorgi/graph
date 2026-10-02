# Object oriented graph implementation

This repo contains a Java implementation of a generic undirected graph, without [loops](https://mathworld.wolfram.com/GraphLoop.html) or [multiple edges](https://mathworld.wolfram.com/MultipleEdge.html), along with a small social network demo that builds a "friendship" graph from a text file, computes some simple metrics, and draws the graph in the browser.

This was the midterm project for the *Programmazione II* (Programming Languages) course; the official specification and the original report are both in Italian:

- [Project specs](specifiche_graph.pdf) (`specifiche_graph.pdf`)
- [Project report](relazione_graph.pdf) (`relazione_graph.pdf`)




## The assignment

The specification asked for:

1. A full **specification** of the abstract data type `Graph<E>` as a Java interface, explaining the design choices. `Graph<E>` is a collection of homogeneous generic objects of type `E`, organized as a graph.
2. An **implementation** of the ADT, documented with its *abstraction function* and *representation invariant*.
3. A **social network** built on top of `Graph<E>`, where nodes are users and edges are friendships, plus some simple metrics such as the distance between two users (the length of the shortest path, i.e. how many intermediaries separate them) and the diameter of the network (the longest of all shortest paths, which measures how compact the network is).




## Repository layout

```
.
├── README.md
├── LICENSE                 GNU GPL v3
├── specifiche_graph.pdf    assignment (Italian)
├── relazione_graph.pdf     project report (Italian)
└── src/
    ├── Graph.java          the Graph<E> interface (ADT specification)
    ├── GraphMap.java       TreeMap-based implementation + RepInvariantException
    ├── Graphs.java         static utilities: file loading, printing, HTML drawing
    ├── SocialNetTest.java  interactive social network demo (main)
    ├── friends.txt         sample social network
    ├── arbor.js            arbor.js 0.91 graph layout library (third party)
    ├── graphics.js         arbor-graphics canvas helpers (third party)
    ├── renderer.js         arbor canvas renderer with drag & drop (third party)
    └── jquery.min.js       jQuery 1.4.4 (third party)
```

Code comments and console messages are in Italian.




## Design

### `Graph<E>`: the specification

[`Graph.java`](https://github.com/matteogiorgi/graph/blob/master/src/Graph.java) defines a graph as a mutable collection of homogeneous objects of type `E` with no duplicates, fully described by the pair `<V, A>`:

- `V = { x | x instanceof E }`, a finite set of vertices;
- `A = { <x,y> | x != y, x,y ∈ V }`, a finite set of edges, each one identified by the pair of vertices it connects.

Each method is documented in the Liskov style, with `MODIFIES` and `EFFECTS` clauses. The interface declares 15 operations:

| Category | Methods |
|---|---|
| Manipulation | `addVertex`, `removeVertex`, `addEdge`, `removeEdge` |
| Inspection & info | `existsVertex`, `existsEdge`, `numVertex`, `numEdge`, `degreeVertex`, `distanceInBetween`, `graphDiameter` |
| Other | `listVertex`, `adjacentVertex`, `toArray`, `equals` |


### `GraphMap<E>`: the implementation

[`GraphMap.java`](https://github.com/matteogiorgi/graph/blob/master/src/GraphMap.java) implements `Graph<E>` as an adjacency map:

```java
public class GraphMap<E extends Comparable<E>> implements Iterable<E>, Graph<E>
{
    private TreeMap<E, TreeSet<E>> st;   // vertex -> set of adjacent vertices
    private int num_edge;                // edge counter
    ...
}
```

- `E` must implement `Comparable<E>`, so vertices and adjacency sets are kept in their natural order. Lookups of vertices and edges take logarithmic time, and iteration and printing come out sorted.
- `GraphMap` implements `Iterable<E>`, which iterates over the vertices in ascending order.
- Every edge `<x,y>` is stored twice, as `y ∈ st.get(x)` and as `x ∈ st.get(y)`. This keeps the graph undirected.

**Abstraction function**

```
AF = <V, A>  where  V = st.keySet()
                    A = { <x,y> | st.containsKey(x) && st.containsKey(y) && st.get(x).contains(y) }
```

**Representation invariant** (summarized)

- `st != null`, and no key or adjacent vertex is `null`;
- every adjacent vertex is itself a key of the map (no dangling edges);
- no vertex is adjacent to itself (no loops);
- adjacency is symmetric: `y ∈ st.get(x) ⇒ x ∈ st.get(y)`;
- `num_edge` = (Σ degrees) / 2, and `0 ≤ num_edge ≤ N(N-1)/2`.

The private method `repOk()` checks the invariant and throws a `RepInvariantException` when it is broken. Because this was a teaching project, `repOk()` is called at the end of every mutator and constructor, so that each one can be seen to be correct. This makes mutators O(V + E), which is fine for a demo but not for production use.

**Constructors**

- `GraphMap()` creates an empty graph.
- `GraphMap(Set<E> vertexes, List<TreeSet<E>> adjoints)` pairs each vertex, in iteration order, with the matching adjacency set, so a whole graph can be built in one call.

**Breadth-first search: the `Path` inner class**

`distanceInBetween`, `pathInBetween` and `graphDiameter` all rely on a private inner class `Path`. Its constructor runs a BFS and fills two maps, which together describe the BFS (shortest-path) spanning tree:

- `distMap`: vertex → distance from the root;
- `prevMap`: vertex → its predecessor on a shortest path.

`Path` takes variadic arguments. With one vertex it explores the whole connected component from that root. With two vertices it stops as soon as the target is reached, which builds only the part of the tree needed to answer the query.

- `distanceInBetween(v, w)` returns the shortest-path length, or `-1` if `v` and `w` are in different connected components.
- `pathInBetween(v, w)` walks `prevMap` back from `w` and returns the intermediate vertices on a shortest path.
- `graphDiameter()` runs a BFS from every vertex and returns the largest distance found (O(V·(V+E))). Unreachable pairs are ignored, so on a disconnected graph the result is the largest diameter among its components.

**Extra methods** beyond the interface: `addSetVertex`, `addAttachedVertex`, `removeSetVertex`, `removeAllVertex`, `addSetEdge`, `removeSetEdge`, `isolateVertex`, `pathInBetween`, `commonNeighbours` (returns a `Stream<E>`), `toString`.

**Exposure of the representation.** Since `E` is generic, the vertex objects themselves cannot be safely copied. `listVertex()` and `adjacentVertex()` at least return unmodifiable `NavigableSet` views, so clients cannot change the internal structure through them.

**Exceptions.** Exception handling is defensive, and every exception is unchecked (a subclass of `RuntimeException`):

| Situation | Exception |
|---|---|
| `null` argument | `NullPointerException` |
| vertex not in the graph | `IllegalArgumentException` (via private `checkVertex`) |
| adding an existing vertex/edge, removing a missing edge | `java.lang.reflect.MalformedParametersException` |
| representation invariant violated | `RepInvariantException` (custom) |


### `Graphs`: static utilities

[`Graphs.java`](https://github.com/matteogiorgi/graph/blob/master/src/Graphs.java) plays the same role for graphs that `Arrays` and `Collections` play for arrays and collections:

- `asGraph(FileReader)` parses a text file and returns a new `GraphMap<String>` using the second constructor.
- `readAndFill(GraphMap<String>, FileReader)` parses a text file and fills an existing (empty) graph with `addSetVertex`/`addEdge`.
- `drawGraph(GraphMap<String>, String name)` writes a temporary `GraphTest<name>.html` page, which is deleted when the JVM exits, and opens it in the default browser. The page uses [arbor.js](https://github.com/samizdatco/arbor) to show the graph as a force-directed layout whose nodes can be dragged with the mouse.
- `printInfo(GraphMap<E>)` prints the number of vertices, the number of edges, the diameter and the sorted vertex list.


### Input format: `friends.txt`

Each line describes one vertex followed by its neighbours. Tokens can be separated by `-`, `:` or whitespace:

```
matteo-piero:marco:simone
simone-matteo:marco:nicola:giovanna:stefania:mario
nino
```

- The first token is the vertex and the remaining tokens are its friends.
- A line with only one name creates an isolated vertex (`nino` above).
- Blank lines are ignored.
- An edge may be listed on both endpoints' lines.

The sample file describes a network of 23 people and 30 friendships, split into several connected components, with a diameter of 8.


### `SocialNetTest`: the interactive demo

[`SocialNetTest.java`](https://github.com/matteogiorgi/graph/blob/master/src/SocialNetTest.java) builds the same network twice: `alpha` with `Graphs.asGraph` and `beta` with `Graphs.readAndFill`. It draws both in the browser, prints their info and checks that `alpha.equals(beta)` is `true`. It then runs a text menu where you pick a graph and an operation:

```
 0. No operation                      7. Insert a vertex already linked to friends
 1. Info on a vertex (degree,         8. Isolate a vertex
    friends)                          9. Remove an edge
 2. Info on a pair (distance, path,  10. Remove a set of edges
    common friends)                  11. Insert an edge
 3. Remove a vertex                  12. Insert a set of edges
 4. Remove a set of vertices         13. Empty the graph
 5. Insert a vertex
 6. Insert a set of vertices
```

When an operation is given a list, end it with `//`. After every change, the modified graph is drawn again and the two graphs are compared once more.




## Build and run

Requires **Java 8 or later**, because the code uses lambdas, streams, `MalformedParametersException` and `Collections.unmodifiableNavigableSet`.

Run the program from inside `src/`. It reads `friends.txt` from the current directory, and the generated HTML pages load the `.js` files from there too.

```sh
cd src
javac *.java
java SocialNetTest
```

Enter `0` at the graph-selection prompt to quit. If the platform has no desktop browser (`java.awt.Desktop` is not supported), nothing is drawn and the console part works as usual.
