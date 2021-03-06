---
title: Notes on Jena internals
---

***<span style="font-size:larger;">Note: These notes are not kept up to date.</span>***

**They may be of interest into the original design of the _Enhanced Node_ mechanism.**

## Enhanced Nodes

This note is a development of the original note on the enhanced
node and graph design of Jena 2.

### Key objectives for the enhanced node design

One problem with the Jena 1 design was that both the DAML layer and
the RDB layer independently extended Resource with domain-specific
information. That made it impossible to have a DAML-over-RDB
implementation. While this could have been fixed by using the
"enhanced resource" mechanism of Jena 1, that would have left a
second problem.

In Jena 1.0, once a resource has been determined to be a DAML Class
(for instance), that remains true for the lifetime of the model. If
a resource starts out not qualifying as a DAML Class (no
`rdf:type daml:Class`) then adding the type assertion later doesn't
make it a Class. Similarly, of a resource is a DAML Class, but then
the type assertion is retracted, the resource is still apparently a
class.

Hence being a DAMLClass is a *view* of the resource that may change
over time. Moreover, a given resource may validly have a number of
different views simultaneously. Using the current `DAMLClass`
implementation method means that a given resource is limited to a
single such view.

A key objective of the new design is to allow different views, or
*facets*, to be used dynamically when accessing a node. The new
design allows nodes to be polymorphic, in the sense that the same
underlying node from the graph can present different encapsulations
- thus different affordances to the programmer - on request.

In summary, the enhanced node design in Jena 2.0 allows programmers
to:

-   provide alternative perspectives onto a node from a graph,
    supporting additional functionality particular to that perspective;
-   dynamically convert a between perspectives on nodes;
-   register implementations of implementation classes that present
    the node as an alternative perspective.

### Terminology

To assist the following discussion, the key terms are introduced
first.

node
  ~ A subject or object from a triple in the underlying graph
graph
  ~ The underlying container of RDF triples that simplifies the
    previous abstraction Model
enhanced node
  ~ An encapsulation of a node that adds additional state or
    functionality to the interface defined for node. For example, a bag
    is a resource that contains a number of other resources; an
    enhanced node encapsulating a bag might provide simplified
    programmatic access to the members of the bag.
enhanced graph
  ~ Just as an enhanced node encapsulates a node and adds extra
    functionality, an enhanced graph encapsulates an underlying graph
    and provides additional features. For example, both Model and
    DAMLModel can be thought of as enhancements to the (deliberately
    simple) interface to graphs.
polymorphic
  ~ An abstract super-class of enhanced graph and enhanced node
    that exists purely to provide shared implementation.
personality
  ~ An abstraction that circumscribes the set of alternative views
    that are available in a given context. In particular, defines a
    mapping from types (q.v.) to implementations (q.v.). This seems to
    be taken to be closed for graphs.
implementation
  ~ A factory object that is able to generate polymorphic objects
    that present a given enhanced node according to a given type. For
    example, an alt implementation can produce a sub-class of enhanced
    node that provides accessors for the members of the alt.

#### Key points

Some key features of the design are:

-   every enhanced graph has a single graph personality, which
    represents the types of all the enhanced nodes that can be created
    in this graph;
-   every enhanced node refers to that personality
-   different kinds of enhanced graph can have different
    personalities, for example, may implement interfaces in different
    ways, or not implement some at all.
-   enhanced nodes wrap information in the graph, but keep no
    independent state; they may be discarded and regenerated at whim.

### How an enhanced node is created

#### Creation from another enhanced node

If `en` is an enhanced node representing some resource we wish to
be able to view as being of some (Java) class/interface `T`, the
expression `en.as(T.class)` will either deliver an EnhNode of type
`C`, if it is possible to do so, or throw an exception if not.

To check if the conversion is allowed, without having to catch
exceptions, the expression `en.canAs(T.class)` delivers `true` iff
the conversion is possible.

#### Creation from a base node

Somehow, some seed enhanced node must be created, otherwise `as()`
would have nothing to work on. Subclasses of enhanced node provide
constructors (perhaps hidden behind factories) which wrap plain
nodes up in enhanced graphs. Eventually these invoke the
constructor
    `EnhNode(Node,EnhGraph)`

It's up to the constructors for the enhanced node subclasses to
ensure that they are called with appropriate arguments.
#### internal operation of the conversion

`as(Class T)` is defined on EnhNode to invoke `asInternal(T)` in
`Polymorphic`. If the original enhanced node `en`is already a valid
instance of `T`, it is returned as the result. Validity is checked
by the method `isValue()`.

If `en` is not already of type `T`, then a cache of alternative
views of `en` is consulted to see if a suitable alternative exists.
The cache is implemented as a *sibling ring* of enhanced nodes -
each enhanced node has a link to its next sibling, and the "last"
node links back to the "first". This makes it cheap to find
alternative views if there are not too many of them, and avoids
caches filling up with dead nodes and having to be flushed.

If there is no existing suitable enhanced node, the node's
personality is consulted. The personality maps the desired class
type to an `Implementation` object, which is a factory with a
`wrap` method which takes a (plain) node and an enhanced graph and
delivers the new enhanced node after checking that its conditions
apply. The new enhanced node is then linked into the sibling ring.

### How to build an enhanced node & graph

What you have to do to define an enhanced node/graph
implementation:

1.  define an interface `I` for the new enhanced node. (You could
    use just the implementation class, but we've stuck with the
    interface, because there might be different implementations)
2.  define the implementation class `C`. This is just a front for
    the enhanced node. All the state of `C` is reflected in the graph
    (except for caching; but beware that the graph can change without
    notice).
3.  define an `Implementation` class for the factory. This class
    defines methods `canWrap` and `wrap`, which test a node to see if
    it is allowed to represent `I` and construct an implementation of
    `C`respectively.
4.  Arrange that the personality of the graph maps the class of `I`
    to the factory. At the moment we do this by using (a copy of) the
    built-in graph personality as the personality for the enhanced
    graph.

For an example, see the code for `ReifiedStatementImpl`.


## Reification API

### Introduction

This document describes the reification API in Jena2, following
discussions based on the 0.5a document. The essential decision made
during that discussion is that reification triples are captured and
dealt with by the Model transparently and appropriately.

### Context

The first Jena implementation made some attempt to optimise the
representation of reification. In particular it tried to avoid so
called 'triple bloat', *ie* requiring four triples to represent the
reification of a statement. The approach taken was to make a
*Statement* a subclass of *Resource* so that properties could be
directly attached to statement objects.

There are a number of defects in the Jena 1 approach.

-   Not everyone in the team was bought in to the approach
-   The *.equals()* method for *Statement*s was arguably wrong and
    also violated the Java requirements on a *.equals()*
-   The implied triples of a reification were not present so could
    not be searched for
-   There was confusion between the optimised representation and
    explicit representation of reification using triples
-   The optimisation did not round trip through RDF/XML using the
    the writers and ARP.

However, there are some supporters of the approach. They liked:
-   the avoidance of triple bloat
-   that the extra reifications statements are not there to be
    found on queries or ListStatements and do not affect the *size()*
    method.

Since Jena was first written the RDFCore WG have clarified the
meaning of a reified statement. Whilst Jena 1 took a reified
statement to denote a statement, RDFCore have decided that a
reified statement denotes an occurrence of a statement, otherwise
called a stating. The Jena 1 *.equals()* methods for *Statement*s
is thus inappropriate for comparing reified statements.
The goal of reification support in the Jena 2 implementation are:

-   to conform to the revised RDF specifications
-   to maintain the expectations of Jena 1; *ie* they should still be
    able to reify everything without worrying about triple bloat if
    they want to
-   as far as is consistent with 2, to not break existing code, or
    at least make it easy to transition old code to Jena 2.
-   to enable round tripping through RDF/XML and other RDF
    representation languages
-   enable a complete standard compliant implementation, but not
    necessarily as default

### Presentation API

*Statement* will no longer be a subclass of *Resource*. Thus a
statement may not be used where a resource is expected. Instead, a
new interface *ReifiedStatement* will be defined:

    public interface ReifiedStatement extends Resource
        {
        public Statement getStatement();
        // could call it a day at that or could duplicate convenience
        // methods from Statement, eg getSubject(), getInt().
        ...
        }

The *Statement* interface will be extended with the following
methods:
    public interface Statement
        ...
        public ReifiedStatement createReifiedStatement();
        public ReifiedStatement createReifiedStatement(String URI);
    /* */
        public boolean isReified();
        public ReifiedStatement getAnyReifiedStatement();
    /* */
        public RSIterator listReifiedStatements();
    /* */
        public void removeAllReifications();
        ...

*RSIterator* is a new iterator which returns *ReifiedStatement*s.
It is an extension of *ResourceIterator*.
The *Model* interface will be extended with the following methods:

    public interface Model
        ...
        public ReifiedStatement createReifiedStatement(Statement stmt);
        public ReifiedStatement createReifiedStatement(String URI, Statement stmt);
    /* */
        public boolean isReified(Statement st);
        public ReifiedStatement getAnyReifiedStatement(Statement stmt);
    /* */
        public RSIterator listReifiedStatements();
        public RSIterator listReifiedStatements(Statement stmt);
    /* */
        public void removeReifiedStatement(reifiedStatement rs);
        public void removeAllReifications(Statement st);
        ...

The methods in *Statement* are defined to be the obvious calls of
methods in *Model*. The interaction of those models is expressed
below. Reification operates over statements in the model which use
predicates **rdf:subject**, **rdf:predicate**, **rdf:object**, and
**rdf:type** with object **rdf:Statement**.
*statements with those predicates are, by default, invisible*. They
do not appear in calls of *listStatements*, *contains*, or uses of
the *Query* mechanism. Adding them to the model will not affect
*size()*. Models that do not hide reification quads will also be
available.

### Retrieval

The *Model::as()* mechanism will allow the retrieval of reified
statements.

    someResource.as( ReifiedStatement.class )

If *someResource* has an associated reification quad, then this
will deliver an instance *rs* of *ReifiedStatement* such that
*rs.getStatement()* will be the statement *rs* reifies. Otherwise a
*DoesNotReifyException* will be thrown. (Use the predicate
*canAs()* to test if the conversion is possible.)
It does not matter how the quad components have arrived in the
model; explicitly asserted or by the *create* mechanisms described
below. If quad components are removed from the model, existing
*ReifiedStatement* objects will continue to function, but
conversions using *as()* will fail.

### Creation

*createReifiedStatement(Statement stmt)* creates a new
*ReifiedStatement* object that reifies *stmt*; the appropriate
quads are inserted into the model. The resulting resource is a
blank node.

*createReifiedStatement(String URI, Statement stmt)* creates a new
*ReifiedStatement* object that reifies *stmt*; the appropriate
quads are inserted into the model. The resulting resource is a
*Resource* with the URI given.

### Equality

Two reified statements are *.equals()* iff they reify the same
statement and have *.equals()* resources. Thus it is possible for
equal *Statement*s to have unequal reifications.

### IsReified

*isReified(Statement st)* is true iff in the *Model* of this
*Statement* there is a reification quad for this *Statement*. It
does not matter if the quad was inserted piece-by-piece or all at
once using a *create* method.

### Fetching

*getAnyReifiedStatement(Statement st)* delivers an existing
*ReifiedStatement* object that reifies *st*, if there is one;
otherwise it creates a new one. If there are multiple reifications
for *st*, it is not specified which one will be returned.

### Listing

*listReifiedStatements()* will return an *RSIterator* which will
deliver all the reified statements in the model.

*listReifiedStatements( Statement st )* will return an *RSIterator*
which will deliver all the reified statements in the model that
reifiy *st*.

### Removal

*removeReifiedStatement(ReifiedStatement rs)* will remove the
reification *rs* from the model by removing the reification quad.
Other reified statements with different resources will remain.

*removeAllReifications(Statement st)* will remove all the
reifications in this model which reify *st*.

### Input and output

The writers will have access to the complete set of *Statement*s
and will be able to write out the quad components.

The readers need have no special machinery, but it would be
efficient for them to be able to call *createReifiedStatement* when
detecting an reification.

### Performance

Jena1's "statements as resources" approach avoided triples bloat by
not storing the reification quads. How, then, do we avoid triple
bloat in Jena2?

The underlying machinery is intended to capture the reification
quad components and store them in a form optimised for reification.
In particular, in the case where a statement is completely reified,
it is expected to store only the implementation representation of
the *Statement*.

*createReifiedStatement* is expected to bypass the construction and
detection of the quad components, so that in the "usual case" they
will never come into existence.


## The Reification SPI

### Introduction

This document describes the reification SPI, the mechanisms by
which the Graph family supports the Model API reification
interface.

Graphs handle reification at two levels. First, their reifier
supports requests to reify triples and to search for reifications.
The reifier is responsible for managing the reification information
it adds and removes - the graph is not involved.

Second, a graph may optionally allow all triples added and removed
through its normal operations (including the bulk update
interfaces) to be monitored by its reifier. If so, all appropriate
triples become the property of the reifier - they are no longer
visible through the graph.

A graph may also have a reifier that doesn't do any reification.
This is useful for internal graphs that are not exposed as models.
So there are three kinds of `Graph`:

Graphs that do no reification;
Graphs that only do explicit reification;
Graphs that do implicit reification.

### Graph operations for reification

The primary reification operation on graphs is to extract their
`Reifier` instance. Handing reification off to a different class
allows reification to be handled independently of other Graph
issues, eg query handling, bulk update.

#### Graph.getReifier() -\> Reifier

Returns the `Reifier` for this `Graph`. Each graph has a single
reifier during its lifetime. The reifier object need not be
allocated until the first call of `getReifier()`.
### add(Triple), delete(Triple)

These two operations may defer their triples to the graph's reifier
using `handledAdd(Triple)` and `handledDelete(Triple)`; see below
for details.

### Interface Reifier

Instances of `Reifier` handle reification requests from their
`Graph` and from the API level code (issues by the API class
`ModelReifier`.

#### reifier.getHiddenTriples() -\> Graph

The reifier may keep reification triples to itself, coded in some
special way, rather than having them stored in the parent `Graph`.
This method exposes those triples as another `Graph`. This is a
dynamic graph - it changes as the underlying reifications change.
However, it is read-only; triples cannot be added to or removed
from it.
    The `SimpleReifier` implementation currently does not implement a
    dynamic graph. This is a bug that will need fixing.

#### reifier.getParentGraph() -\> Graph

Get the `Graph` that this reifier serves; the result is never
`null`. (Thus the observable relationship between graphs and
reifiers is 1-1.)

#### class AlreadyReifiedException

This class extends `RDFException`; it is the exception that may be
thrown by `reifyAs`.
### reifier.reifyAs( Triple t, Node n ) -\> Node

Record the `t` as reified in the parent `Graph` by the given `n`
and returns `n`. If `n` already reifies a different `Triple`, throw
a `AlreadyReifiedException`.
Calling `reifyAs(t,n)` is like adding the triples:

`n rdf:type ref:Statement`
`n rdf:subject t.getSubject()`
`n rdf:predicate t.getPredicate()`
`n rdf:object t.getObject()`
to the associated Graph; however, it is intended that it is
efficient in both time and space.

#### reifier.hasTriple( Triple t ) -\> boolean

Returns true iff some `Node n` reifies `t` in this `Reifier`,
typically by an unretracted call of `reifyAs(t,n)`.
The intended (and actual) use for `hasTriple(Triple)` is in the
implementation of `isReified(Statement)` in `Model`.

#### reifier.getTriple( Node n ) -\> Triple

Get the single `Triple` associated with `n`, if there is one. If
there isn't, return `null`.
A node reifies at most one triple. If `reifyAs`, with its explicit
check, is bypassed, and extra reification triples are asserted into
the parent graph, then `getTriple()` will simply return `null`.

### reifier.allNodes() -\> ExtendedIterator

Returns an (extended) iterator over all the nodes that (still)
reifiy something in this reifier.
This is intended for the implementation of `listReifiedStatements`
in `Model`.

#### reifier.allNodes( Triple t ) -\> ClosableIterator

Returns an iterator over all the nodes that (still) reify the
triple \_t\_.
### reifier.remove( Node n, Triple t )

Remove the association between `n` and the triple`t`. Subsequently,
`hasNode(n)` will return false and `getTriple(n)` will return
`null`.
This method is used to implement `removeReification(Statement)` in
`Model`.

#### reifier.remove( Triple t )

Remove all the associations between any node `n` and `t`; ie, for
all `n` do `remove(n,t)`.
This method is used to implement `removeAllReifications` in
`Model`.

#### handledAdd( Triple t ) -\> boolean

A graph doing reification may choose to monitor the triples being
added to it and have the reifier handle reification triples. In
this case, the graph's `add(t)` should call `handledAdd(t)` and
only proceed with its add if the result is `false`.
A graph that does not use `handledAdd()` [and `handledDelete()`]
can only use the explicit reification supplied by its reifier.

#### handledRemove( Triple t )

As for `handledAdd(t)`, but applied to `delete`.

### SimpleReifier

`SimpleReifier` is an implementation of `Reifier` suitable for
in-memory `Graph`s built over `GraphBase`. It operates in either of
two modes: with and without triple interception. With interception
enabled, reification triples fed to (or removed from) its parent
graph are captured using `handledAdd()` and `handledRemove`;
otherwise they are ignored and the graph must store them itself.
`SimpleReifier` keeps a map from nodes to the reification
information about that node. Nodes which have no reification
information (most of them, in the usual case) do not appear in the
map at all.

Nodes with partial or excessive reification information are
associated with `Fragments`. A `Fragments` for a node `n` records
separately

the `S`s of all `n ref:subject S` triples
the `P`s of all `n ref:predicate P` triples
the `O`s of all `n ref:subject O` triples
the `T`s of all `n ref:type T[Statement]` triples
If the `Fragments` becomes *singular*, ie each of these sets
contains exactly one element, then `n` represents a reification of
the triple `(S, P, O)`, and the `Fragments` object is replaced by
that triple.
(If another reification triple for `n` arrives, then the triple is
re-exploded into `Fragments`.)


