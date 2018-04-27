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

  - Relational databases 

    Well-defined theory, in which the notion of “tabular data” corresponds
    reasonably well with that of “relation.” Essentially all modern databases
    (up until the recent interest in so-called “NoSQL” models) are based around
    the relational model. In practice, implementations (Relational Database
    Management Systems, or RDMSs) tend not to be completely faithful to the
    theory. (For example, there are two relations having no columns but these are
    usually not supported.)

    Many concepts arising in the practice of “data wrangling” come from database
    practice: filters, selections, and joins, for example.

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

    I don't know enough about these [see, for example, @Acar:2010:GMD:1855795.1855803].
    
  

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
Boolean. Real-world data contain a greater richness of meaning than these
$\nu$ are the identity maps)representations
support.

FIXME: Say something about R class mechanism. Is there something equivalent in Pandas?

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
usually poorly-specified.

The other kind of complex data occurs when a single “value” is composed of more
than one primitive value, typically from multiple columns. An example is
geographical coordinates, with longitude in one column and latitude in
another. One often wishes to treat such data as a whole, for example, when
making a map, converting to an address, or converting between coordinate
systems. 

These two kinds of complex data sound a lot like sum and product types in typed
programming languages.


### “User-defined“ annotations

A new system may fail to be used because it is too inflexible and cannot be
extended in ways that the user requires for her particular purpose. On the other
hand, it can also fail because the extension mechanism is *too* open, providing
few semantic guarantees or constraints. Navigating this strait is not trivial.


## Distinction between data, representation, and interpretation.

TODO: I presume there is some existing terminology for this. For any particular
value (eg, in a csv file) there are three meanings which we might be talking
about:

FIXME: I think there's something about this in the OWL spec.

1. The kind of this value, thought of as a real-world quantity (eg, a real
   number);
   
2. The approximation of that quantity in our necessarily-finite computer (eg, as
   a 32-bit IEEE 754 floating point number);
   
3. The representation of that approximation as, for example, a sequence of UTF8
   characters. 
   


# Appendix A: Overview of the Relational Model

FIXME: Someone who knows this subject should check this! The following formalism
is what sense I have been able to make of this subject. It's somewhat different
to the usual presentation, which uses names to do a lot of the work I have done
with maps.

## Formalism

### Relations and databases

Fix, once and for all, a finite set of *domains*, $\mathcal{D} =
\{\mathcal{D}_1, \mathcal{D}_2, \dotsc, \mathcal{D}_n\}$. Each domain
$\mathcal{D}_i$ is a (possibly infinite) set.

A *relation schema* is a finite tuple of domains. (Recall that a tuple differs
from a set in being ordered and allowing duplicates.) The *carrier* of a
relation schema is the cartesian product of the domains in that relation
schema.[^carrier] By a *relation* is meant a relation schema and a finite subset
of the carrier of that schema. The *header* of a relation is the corresponding
relation schema.

[^carrier]: As far as I am aware, the nomenclature of “carrier” is not common
    but it proves convenient in stating certain definitions later on. FIXME:
    Probably the notion of carrier should apply to a relation, rather than a
    relation schema.

A *database instance* consists of the following data:
 
  1. A finite set of *names*;
  2. For each name, a relation schema; and
  3. For each name, a relation, whose header is the given relation schema.

(That, at any rate, is what I understand the formalism to be.)

### Integrity constraints and normal forms

FIXME: What is going on here?

*Integrity constraints* are restrictions on the actual relations that might be
present in a database instance. *Normal forms* are restrictions on the sorts of
“functional dependencies” that may exist. I am not convinced I understand how
they fit into this formalism.

Perhaps integrity constraints are modelled by giving a set of projections (see
below) that must hold in all database instances. 


### The relational algebra

There is also given an algebra on the set of all relations.[^all-relations] There are, or appear
to be, three kinds of operations in this algebra: set-theoretic--type operations
on pairs of “compatible” relations; “joins” of pairs of relations; and
“projections” from one relation to another.

[^all-relations]: Presumably the algebra is defined on the set of all relations
over a fixed set of domains.

If two relations have identical headers, then they are subsets of the same set
(specifically, they are subsets of the same carrier). For such relations the
union, intersection, and difference are defined as those operations on those
subsets.

Projections “restrict the relation to particular columns”, but the definition is
not entirely straightforward because of the need to keep the domains straight.

Given two relation schemas, $S$ and $T$, let $\mu:T\to S$ be a map that
preserves domains (note that the arrow goes the “wrong way”). For example, if $S
= (\mathcal{D}_3, \mathcal{D}_1, \mathcal{D}_1)$ and $T = (\mathcal{D}_1)$, then
there are two possible maps $T\to S$: one taking the single domain in $T$ to the
second domain in $S$, and one taking it to the third domain. Let $r_T$ be a
relation with header $T$. We can extend $\mu$ to a map on elements of $r_T$:
given an element of $r_T$ (*i.e.*, a tuple) we construct an element of the
carrier of $S$ by the obvious action of $\mu$ on that tuple.

A *projection*, $\pi_\mu(r_S)$ on some relation $r_S$ of relation schema $S$ (this
time the arrow goes the right way!) is that relation $r_T$ (as a subset of the
carrier of $T$) whose elements are such that the action of $\mu$ produces an
element of $r_S$. Informally, it is the “restriction of $r_S$ to the domains in
$T$.” Alternatively, it is “the set of tuples in the carrier of $T$ for which
there exists some values of the domains of $S$ that are not in $T$ that makes
the values in the other domains equal.”

Note that the operation “re-order the domains in a relation” is a projection.

Finally, let $r_S$ and $r_T$ be two relations having headers $S$ and $T$
respectively. Suppose there is given a relation schema $U$ and two projections
$\pi_\mu:U\to S$ and $\pi_\nu:U\to T$ respectively. A *join* of $r_S$ and $r_T$
is the maximal relation $r_U$ whose carrier is $U$ and for which $\pi_\mu(r_U) =
r_S$ and $\pi_\nu(r_U) = r_T$. (Here “maximal” means that there is no relation,
having the same property, for which this one is a subset.)

Note that intersection of relations can be defined in terms of joins (it is the
join of two relations having the same header, where the maps $\mu$ and $\nu$,
corresponding to the projections, are the identity maps).

The join is often written $r_S\bowtie r_T$, though note that in our version one
has to specify the maps $\mu$ and $\nu$.

The conventional story tries to get rid of these maps between the headers. It
does this by considering a relation schema to be a set (not a tuple) of
*attributes*, where an attribute is a pair of a *name* and a domain. It is then
required, in the definitions, that maps of relation schemas preserve attributes.
Effectively, the allowed maps are specified precisely by the attribute
names. (One also introduces auxiliary operators whose job is to allow renaming.)

Descriptions of the relational algebra often include a plethora of other
operations, including *equijoin*, *semijoin*, *antijoin*, and *division*. I'm
pretty sure these are all definable in terms of the operations described above.

It strikes me that the definition of a join given above bears a strong
resemblance to the universal construction of a product (as in category
theory). Perhaps there's a better version of all this which starts with the
projections, thought of as morphisms, and gets the other gadgetry for free.


## Interpretation

The intended interpretation, I think, is as follows. Each domain represents the
“allowed primitive values of a certain type.” One could imagine that the domains
are things like “the real numbers,”, “the natural numbers”, “the set {true,
false}.” In practice, the domains are in fact such things as: “strings of ASCII
characters of at length at most 1024,” “floating point numbers,” “fixed
precision decimals with at most 10 digits before the decimal point and two
digits after,” and so on.

Each relation represents “a set of facts”, each fact being the truth of some
“proposition” $P(C_1, \dotsc, C_n)$ where $P$ is an $n$-ary predicate and the
$C_i$ are elements of the domains in the header of the relation.

The “closed-world assumption” is the assumption that a proposition is false if
it could be represented by an element of a relation but does not in fact occur
in that relation.

Finally, the operations of the algebra allow one to “deduce other facts from the
ones given.” In RDMSs, the language SQL roughly describes the algebra
above.

## Problems

The formalism above is somewhat unsatisfactory as a practical model of
data. (Although it's significantly better than a lot of other proposals!) In
part, it may be that I have some of the formalism wrong. However, it's also true
that in practice one frequently sees large variations on this model---such as
“object-oriented databases”---which suggests that there is some unfulfilled
need.[^date]

[^date]: It's true that some practitioners, most notably Date and
    Darwen [@the-third-manifesto], assert that there would be no need for such
    extensions if only the relational theory were implemented correctly. For
    whatever reason, they have not been able to persuade the database community
    of this.

For example, one might imagine that the language of facts would be (first-order)
logic. But (a model of a particular) first-order logic doesn't talk about
multiple domains, just a single “domain of discourse.” The arguments of a
predicate may be filled with *any* term, not just an element of a specific
domain. So perhaps the relational model is some kind of typed logic?

The nature of identity is perennially perplexing (at least to me). In
first-order logic, there are constant symbols, standing for individuals. It is
individuals (and functions of individuals) that appear in the arguments of
predicates, rather than elements from a domain, as in the relational
model. Thus, if, in the relational model, one wants to assert the existence of a
particular person, such as Fred Flintstone, one must create a relation (of
persons, perhaps) whose domains are collectively sufficient uniquely to identify
Fred Flintstone amongst all other persons. In practice, such domains are not
always available (or at least obvious) and it is common to find onself having
two persons whom the known facts fail to individuate.

Suppose there are two persons about whom the known facts are identical. Such
persons cannot be distinct tuples in a relation expressing the known facts,
since relations are a set, not a bag. If I have understood Date correctly, I
think he would argue that if the facts about two individuals are identical, then
there is no observation (or query) that would distinguish them, and so nothing
is gained *by* distinguishing them. 

The interpretation that is commonly given in textbooks of a tuple in a “Persons”
table is something like: “*There exists* a person having the following
characteristics...” and that assertion is no less true if there are two persons
satisfying the given condition. But I do not believe that this is the
interpretation that database practitioners have in mind when they create a
Persons table: it seems to me that, if it were explicit, the interpretation
would be “There exists a person *and* that person has the following
characteristics.”

In support of my argument that this interpretation is common I note that
database designers will frequently invent an “id” column, typically valued in
the domain of integers, whose purpose is precisely to identify individuals. “id”
columns are everywhere, with much attendent argument about what to call them.

Perhaps what one ought to do is to specify a new domain---the domain of Persons,
say---and value the “id” columns in *that*. But now one needs a model of how
domains are to be created. The only coherent model that I can see is that in
fact domains *are* relations but, if they are, that relationship is not part of
the formalism.[^domains-as-relations]

[^domains-as-relations]: Although, as noted, perhaps Date and Darwen have solved
    this problem.

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
semantics of this value are confusing. Sometimes its intended interpretation is
to indicate a violation of the closed-world assumption: that there does exist
some value that would make this proposition true but its value is
unknown. (Example: a table of names and ages, where some people's ages are not
known.) Sometimes it is intended to indicate that there is *no* value that would
make the corresponding proposition true. (Example: a table of people and
employers, where some people don't have employers.)

A further discrepancy, less widely known, is that while there are two relations
having an empty relation type (that is, having no columns), there is typically
no representation of these in implementations. (These two relations are the
empty set and the set containing the empty tuple. They are useful in theory as
representations of truth and falsity.)

## Support for the desiderata

Tables are clearly supported; as are, in some sense, the meanings of primitive
types. However, interpretations are not available, nor are complex types or
user-defined annotations. 


# Appendix B: Overview of Ontology

I have at least worked with databases professionally. By contrast, my experience
of ontologies consists mostly of kvetching to actual experts that I don't
understand what an ontology is. So you should probably treat my attempts to
define one with scepticism.

What I think is meant by an ontology is: (a) a set of sentences in a logic; and
(b) a representation of the domain of interest as model of that logic. The
representation captures the meanings of the things of which we wish to speak of;
the sentences say what we know about the world.

In this game, I gather that propositional logic is too weak to be of much use,
but first-order logic has the problem of being undecidable. Instead one chooses
a particular logic from a class of *description logics*, whose expressive power
is less than that of first-order logic but is nonetheless decidable.

Essentially all my understanding (such as it is) comes from
[@DBLP:journals/corr/abs-1201-4089], [@nardi2003introduction], and
[@cameron1999sets].

Almost certainly I am misusing the terms “model” and “interpretation.”

Recall that logic is an abstract game of syntax, consisting of: a description of
how to construct certain strings of symbols; a certain starting set of strings;
and rules for transforming sets of strings into other sets of strings. A *model*
for a logic is an interpretation of the strings in some mathematical structure,
for which the interpretations of the axioms are true.

In first-order logic, the strings are made up of two kinds of symbols. The first
kind are the logical symbols: $=$, $\neg$, $\vee$, $\wedge$, $\to$,
$\leftrightarrow$, $\exists$, and $\forall$; and parentheses and commas.
 
The second kind, perhaps more interesting to us, are ones that relate to the
particular domain of interest. There is supposed to be given a set of *constant
symbols*, a set of *function symbols*, and a set of *relation symbols*. The
latter two come with a number called their *arity* which describes how many
arguments they have.

To specify a model one must specify a set (called the “domain of discourse“),
and give, for each constant symbol, an element of the set; for each function
symbol, a function on the set; and for each relation symbol, a relation on the
set; the latter two must have the appropriate arities. The idea is that this set
captures all the “things” in the world: the constants are individuals, the
one-place relations are predicates, and so on.

For example, in the database world, to denote that Fred's age is 42, we would
consider a relation *Ages* with relation schema, say, (*People*, $\mathbb{N}$),
and include, as an element of that relation, the tuple (Fred, 42). If we wanted
to encode the fact that every person has an age, we might make this table the
authoritative table of people (and insist that no value is
`NULL`). Alternatively, if some other table, *Persons*, with relation schema
(*People*), contained the definitive list of people, then we would insist on
there being a projection *from* this table *to* the *Ages* table (as well as
*vice versa*).

In the ontology world, `Fred` and the element 42 would be constant symbols,
there would be a relation `AgeOf` having arity 2, and we would write
`AgeOf`(`Fred`, 42).[^age-of] If we wanted to encode the same fact as above, we
would introduce predicates `Person?` and `Natural?` and write 
$$ 
\bigl(\forall x\bigr) 
    \bigl(\text{\texttt{Person?}}(x) \to 
    (\exists y) (\text{\texttt{Natural?}}(y) \wedge \text{\texttt{AgeOf}}(x, y)\bigr).
$$

[^age-of]: We might also write `Person`(`Fred`), but that is not necessary for
    the example. 

One might have defined an ontology as this “vocabulary”, together with a
collection of sentences in the logic; these sentences express truths about the
world. However, as noted, reasoning in first-order logics is computationally
difficult. So one considers a more restricted version. 

There are multiple varieties of these restricted versions but they share certain
common features. In paricular, the non-logical symbols are restricted to
constant symbols (which are now known as *individuals*), one-place predicates
(known as *concepts*), and two-place relations (known as *roles*). Furthermore,
the kinds of sentences one is allowed to construct are limited. For example,
supposing $P$ and $Q$ are concepts (*i.e.*, predicates), the rule that
“everything that is $P$ is also $Q$” is written
$$
P \sqsubseteq Q,
$$
instead of, as in first-order logic,
$$
(\forall x)(P(x) \to Q(x)).
$$

As another example, suppose $P$ is a predicate and $R$ is a role (*i.e.*, a
two-place predicate). Then the predicate whose first-order logic form is
written:
$$
(\exists y)(P(y) \wedge Q(x, y)
$$
is written in 



## Thoughts and confusions

I think the following confusions are well-rehearsed but I am not sufficiently
knowledgeable about the subject to say to what extent they have been resolved.









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
