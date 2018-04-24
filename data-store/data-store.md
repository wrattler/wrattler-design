---
title: The Wrattler Data Store
author: James Geddes
date: Draft April 2018
bibliography: data-store.bib
...


The following constitute very drafty thoughts on the design of a “data store”
for Wrattler. Comments, suggestions, and criticism are welcome. 


# Background

## Motivation

A major motivation for the creation of Wrattler is to provide a common framework
into which our various to-be-developed AI assistants shall fit. We anticipate
that these asistants will generate “suggested analyses” whose intent---roughly
roughly---is to add “additional information” to the data. We don't know the
order in which the assistants will run, and it's likely that they will run at
unpredictable times during the data wrangling process, so it will be useful to
have some commmon data language that is rich enough to express whatever
annotations the assistants create, in whatever order they create it. By having
such a language the development of a new assistant will require only a single
translation (to and from the common language) rather than between every possible
pair of agents.

A typical notebook in contemporary practice, such as one created by Jupyter,
comprises a series of “executable cells” of code interspersed with formatted
text and output such as charts. These cells are not independent but are
typically steps in a larger piece of analysis; there is therefore the question
of how state is maintained between the execution of one cell and the next.

The current answer is that the cells are executed in a single runtime process
(the “kernel”) so that the execution of each cell takes place in the environment
created by all the cells executed previously. Cells can be run out of order and
so the environment in which a cell executes might not be that which would be
created by executing the cells lexically preceding the current one.

Once we have a common data language, however, we can imagine purely functional
cells; if cells are purely functional, then it's not necessary for them to be in
the same language, so now we can see our way to a polyglot system. And if we're
going to be tracking how cells depend on other cells, then we can include proper
“lineage” or “provenance.”

Thus, the need to create a common data language seems to offer us a chance to
rethink and improve how notebooks work.

A personal note: I have not been a big fan of notebooks. One reason, I think, is
that sometimes to write clearly requires that the lexical ordering of the
analytical writeup be different from the execution ordering. It is rather like
the distinction, in programming, between adding comments, which are subservient
to the ordering of the code, and *literate programming*, in which the order of
the code is subservient to the needs of exposition. (I admit that literate
programming, despite being created by Knuth, is not commonly used. I don't know
why not.)


## Risks and challenges

The phrase “common data language” should give us pause. There's no shortage of
ideas out there but frankly none of them seem to work well in practice (at least
none of them grab me as a data scientist). So that's a worry.


## Existing examples of a “common data language”

In the following examples, there is perhaps a distinction to be made between
*theories of data* and *ways of representing data*. So, for example, the
relational model is mostly concerned with a certain way of thinking about data;
whereas JSON, say, is mostly concerned with representing data. It is true that
JSON represents certain kinds of data (roughly, those which can be thought of as
a *dictionary* of key-value pairs, where the values may be one of a set of
primitive types or another dictionary) but one usually discusses the
serialisation more than the nature of the data.

  - The relational model 

    Well-defined theory, in which the notion of “tabular data” corresponds
    reasonably well with that of “relation.” Essentially all modern databases
    (up until the recent interest in so-called “NoSQL” models) are based around
    the relational model. In practice, implementations (Relational Database
    Management Systems, or RDMSs) tend not to be completely faithful to the
    theory. For example, there are two relations having no columns but these are
    usually not supported.

    Many concepts in “data wrangling” come from database practice: filters,
    selections, and joins, for example.

  - Types, in the programming language sense

    Reasonably well-defined theory, albeit one that still appears to be
    evolving; not implemented widely outside certain programming languages, most
    notably the ML family of languages. 
    
    However, *type providers*, which expose data as F# types, are one example of
    where types are used to describe and provide an interface to structured
    data.
  
  - Ontologies

    The theory here is logic. There is a family of logics, known as “description
    logics,” whose expressive power is greater than that of propositional logic
    but less than first-order predicate logic, such that queries are guaranteed
    to terminate. As I understand it most ontology systems are based on one of
    these.
    
    Ontologies are, for whatever reason, not widely used by data scientists,
    being used in practice mainly in particular domains, such as healthcare or
    certain industries, whose practitioners acknowledge great difficulties in
    knowledge management.

  - "NoSQL" models

    As far as I can tell, these are predominantly key-value stores; there is no
    metadata to speak of, and very little theory of knowledge *per se*.
  
  - JSON, XML, RDF, YAML, and other structured formats

    Sometimes called “self-documenting data,” which I think refers to the fact
    that certain hierarchical structures are encoded directly in the
    data. Perhaps better thought of as “*ad hoc* data”.
 
    I think of these more as a seralisation mechanism for particular kinds of
    underlying structure, rather as csv is a seralisation of tabular data.

  - [http://schema.org/](Schema.org)

  - “Provenance Graphs”

    I don't know enough about these.[^provenance]
    
[^provenance]: See, eg, https://www.usenix.org/legacy/event/tapp10/tech/full_papers/buneman.pdf
  

# Desiderata

The following are things that I, as a data scientist, would like to see
supported by the system I use every day. (Much of the following was described in
Tomas' design notes.)

## Tabular data

In our prototype for Wrattler we store “tabular” data only, which are data that
consist of a set of rows, each of which “has the same set of fields.” A more
formal definition is given [below](Overview of the Relational Model).

Tabular data covers many use cases and forms the basis of relational
databases. Existing language-based systems, *e.g.*, R or Python Pandas,
typically support some form of tables (which R calls "data frames" and I think
Pandas has copied that nomenclature) so tables are a pre-existing *lingua
franca* of data. Even vectors might be thought of as a single-column table.[^vectors]

We should probably start by building on tables, with an eye to where we want to
go.

[^vectors]: That's “vector” in the data science sense of “sequence indexed by
that naturals up to some $N$” rather than in the mathematical sense of “things
for which there is available the operations of addition, and multiplication by
reals.“

## Semantic annotations

The semantics of data frames as understood by R and Python is rather
impoverished. What is known in these systems is roughly that each column has a
particular "primitive type” -- for example, string, float, integer, and possibly
Boolean.

Real-world data contain a greater richness of meaning than these representations
support.[^types-in-R]

[^types-in-R]: FIXME: Say something about R class mechanism. Is there something
    equivalent in Pandas?

### The meaning of primitive types

At present, many things are left unsaid which should be made explicit in order
to communicate between different languages or assistants. One example is the
size and format of the representation of real numbers (16 bits, 32 bits,
arbitrary precision, arbitrary size, fixed precision decimal, and so on), or
whether non-numeric values such as “`NaN`” and “`+Inf`” are included.

Since we intend to allow communications between different systems, we had better
be able to communicate the representations available (and possibly translate
between them).

### Interpretations of data represented as primitive types

Real-world data has an interpretation. A particular float might be a
representation of a *real number*, specifically a *physical measurement*,
specifically a *temperature*, specifically a temperature *in Celsius*. Very few
systems annotate data with this interpretation.

In R, at least, it is also possible for the class of a column to be a “derived”
type; that is, one which is internally held as a primitive but which is
interpreted as something else. An example is a “Date”, which may internally be
held as a (floating point) number, interpreted as the number of days since some
reference date. 

The data store should be aware of these interpretations -- and possibly provide
a standard set of common interpretations -- but need not understand the detailed
operations available.

### Complex types

Real-world data often contains values of different types in the same column. For
example, survey responses often include special values for the various reasons
for a non-response; these are often coded as negative integers in order to
distinguish them from the categorical answer scale, coded as positive integers.

Raw data frequently contains values that can be thought of as different types,
arising by inconsistencies or errors in data collection, such as measurements
record in different units. 

In our protoype assistant “ptype,” what is returned is a probabilistic guess as
to the type of each value in a column, with the type of the column itself being
a mixture.

However, most systems (especially databases!) insist that every column is a
single type, apart possibly from a distinguished, catch-all, “out of type”
value, such as R's “`NA`” value, or SQL's “`NULL`”, the semantics of which is
usually poorly-understood.

The other kind of complex data occurs when a single “value” is composed of more
than one primitive value, typically from multiple columns. An example is
geographical coordinates, with longitude in one column and latitude in
another. One often wishes to treat such data as a whole, for example, when
making a map, converting to an address, or converting between coordinate
systems. 

### “User-defined“ annotations

A new system may fail to be used because it is too inflexible and cannot be
extended in ways that the user requires for her particular purpose. On the other
hand, it can also fail because the extension mechanism is *too* open, providing
few semantic guarantees or constraints. Navigating this strait is not trivial.


## Distinction between data, representation, and interpretation.

TODO: I presume there is some existing terminology for this. For any particular
value (eg, in a csv file) there are three meanings which we might be talking
about:

1. The kind of this value, thought of as a real-world quantity (eg, a real
   number);
   
2. The approximation of that quantity in our necessarily-finite computer (eg, as
   a 32-bit IEEE 754 floating point number);
   
3. The representation of that approximation as, for example, a sequence of UTF8
   characters. 
   





# Appendix: Overview of the Relational Model

FIXME: Someone who knows should check this!

## Formalism

Fix, once and for all, a finite set of *domains*, $\mathcal{D} =
\{\mathcal{D}_1, \mathcal{D}_2, \dotsc, \mathcal{D}_n\}$. Each domain
$\mathcal{D}_i$ is a (possibly infinite) set.

A *relation type* is a finite tuple of domains. By a *relation* is meant a
finite subset of the cartesian product of the domains in a relation type. The
*header* of a relation is that relation type which is the tuple of domains in
which its arguments are valued.

By a *database* is meant the following data:
    1. A finite set of *names*;
    2. For each name, a relation type; and
    3. For each name a relation, whose header is the given relation type.

(That, at any rate, is what I understand the formalism to be.)

There is also given an algebra on relations.

## Interpretation

The intended interpretation, I think, is as follows. Each domain represents the
“allowed primitive values of a certain type.” One could imagine that the domains
are things like “the real numbers,”, “the natural numbers”, “the set {true,
false}.” In practice, the domains are in fact such things as: “strings of ASCII
characters of at length at most 1024,” “floating point numbers,” “fixed
precision decimals with at most 10 digits before the decimal point and two
digits after.”

Each relation represents “a set of facts”, each fact asserting the truth of some
“proposition” $P(C_1, \dotsc, C_n)$ where $P$ is an $n-$-ary predicate and the
$C_i$ are elements of the domains (and the particular domains are fixed for each
$P$).

There is given an algebra on relations which I won't detail, but roughly has
unary and binary operations corresponding to the database notions of filtering
(selecting rows), projection (selecting columns), and join.

## Problems

The formalism above is somewhat unsatisfactory. In part, it may be that I have
some of the formalism wrong. However, it's also true that in practice one
frequently sees large variations on this model---such as “object-oriented
databases”---which suggests that there is some unfulfilled need.[^date]

[^date]: It's true that some practitioners, most notably Date and
    Darwen,[^third-manifesto] assert that there would be no need for such
    extensions if only the relational theory were implemented correctly. For
    whatever reason, they have not been able to persuade the database community
    of this.

For example one might imagine that the language of facts would be (first-order)
logic. But (a model of a particular) first-order logic doesn't talk about
multiple domains, just a single “domain of discourse.” The arguments of a
predicate may be filled with *any* term, not just an element of a specific
domain. So perhaps the relational model is some kind of typed logic?

The question of the nature of identity also seems perennially perplexing (at
least to me). In first-order logic, there are constant symbols, standing for
individuals. It is individuals (and functions of individuals) that appear in the
arguments of predicates, rather than elements from a domain, as in the
relational model. Thus, if, in the relational model, one wants to assert the
existence of a particular person, such as Fred Flintstone, one must create a
relation (of persons, perhaps) whose domains are collectively sufficient as to
uniquely identify Fred Flintstone amongst all other persons. In practice, such
domains are not always available (or at least obvious) and it is common to find
onself having two persons whom the known facts fail to individuate.

Suppose there are two persons about whom the known facts are identical. Such
persons cannot be distinct tuples in a relation expressing the known facts,
since relations are a set, not a bag. If I have understood Date correctly, I
think he would argue that if the facts about two individuals are identical, then
there is no observation (or query) that would distinguish them, and so nothing
is gained *by* distinguishing them. 

The interpretation that is commonly given in textbooks of a tuple in a “Persons”
table is something like: “There exists a person having the following
characteristics...” and that assertion is no less true if there are two persons
satisfying the given condition. But I do not believe that is the interpretation
that database practitioners have in mind when they create a Persons table: it
seems to me that, if it were explicit, the interpretation would be “There exists
a person *and* that person has the following characteristics.”

In support of my argument that this interpretation is common I note that
database designers will frequently invent an “id” column, typically valued in
the domain of integers, whose purpose is precisely to identify individuals. “id”
columns are everywhere. 

Perhaps what one ought to do is to specify a new domain---the domain of Persons,
say, and value the “id” columns in *that*. But now one needs a model of how
domains are to be created. The only coherent model that I can see is that in
fact domains *are* relations but if they are, that relationship is not part of
the formalism.

Furthermore, there are many facts about particular domains of discourse that one
might wish to express that cannot be expressed simply by giving the headers of a
set of relations. In implementations, one is allowed to assert *a priori*
constraints on the tuples in a relation, and on the functional dependencies
between one relation and another.

Presumably, many of these constraints are the sort of thing that ontologies
describe.

Finally, implementations simply fail to respect the above formalism in several
regards. The following two differences are well-known. A relation is, as the
definition says, a set; however, many implementations model relations as a bag
(that is, alowing duplicate elements). Implementations also adduce a “special”
value, denoted `NULL`, which is allowed in any position in any predicate; the
semantics of this value are confusing.

A further discrepancy, less widely known, is that while there are two relations
having an empty relation type (that is, having no columns), there is typically
no representation of these in implementations. (These two relations are the
empty set and the set containing the empty tuple. They are useful in theory as
representations of truth and falsity.)













# Further reading



-  https://doi.org/10.1017/S0956796818000035

- A paper on literate programming

    * Literate programming (1984) Donald E. Knuth
    * Literate statistical practice (2001) K. Hornik, A. J. Rossini
    * A multi-language computing environment for literate programming and
      reproducible research (2012) Eric Schulte, Dan Davison, Thomas Dye,
      Carsten Dominik

- Edgar F Codd. “A relational model of data for large shared data
  banks”. Communications of the ACM 13.6 (1970), pp. 377–387.

https://en.wikipedia.org/wiki/IEEE_754
