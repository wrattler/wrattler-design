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
into which our various to-be-developed AI assistants can fit. We anticipate that
these asistants will generate “suggested analyses” whose intent -- roughly
roughly -- is to add “additional information” to the data. We don't know the
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

The current answer is that the cells are executed in a single runtime process --
the “kernel” -- so that the execution of each cell takes place in the
environment created by all the cells executed previously. Cells can be run out
of order and so the environment in which a cell executes might not be that which
would be created by executing the cells lexically preceding the current one.

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
the code is subservient to the needs of exposition. Literate programming is not
commonly used and I don't know why not.


## Risks and challenges

The phrase “common data language” should give us pause. There's no shortage of
ideas out there but frankly none of them seem to work well in practice (at least
none of them grab me as a data scientist). So that's a worry.


## Existing examples of a "common data language"

- The relational model 

  Well-defined theory; implementations tend not to be faithful to the theory. (For
  example, there are two relations with no columns but these are usually not
  supported by RDMSs);

- Types, in the PL sense

  Well-define theory; not implemented widely outside certain programming
  languages. However, *type providers*, which expose data as F# types, are one
  example of where types are used to describe data. 
  
- Ontologies

  The theory here appears to be logic, and particularly "description logics",
  which have an expressive power greater than propositional logic but less than
  first-order logic. For whatever reason, such ontologies are not widely used by
  data scientists, and tend to be used in practice in particular domains, such
  as healthcare or certain industries, whose practitioners acknowledge great
  difficulties in knowledge management.

- “Provenance Graphs”

  I don't know enough about
  these. https://www.usenix.org/legacy/event/tapp10/tech/full_papers/buneman.pdf
  
- "NoSQL" models

  As far as I can tell, these are predominantly key-value stores; there is no
  metadata to speak of, and very little theory of knowledge *per se*.
  
- JSON

  Sometimes called “self-documenting data,” which I think refers to the fact
  that certain hierarchical structures are encoded directly in the
  data. Perhaps better thought of as “*ad hoc* data”.
  

# Desiderata

The following are things that I, as a data scientist, would like to see
supported by the system I use every day. (Much of the following was described in
Tomas' design notes.)

## Tabular data

In our existing prototype we store “tabular” data only, which are data that
consist of a set of rows, each of which “has the same set of fields.” A more
formal definition is given [below](Overview of the Relational Model).

Tabular data covers many use cases and forms the basis of relational
databases. All practical systems, *e.g.*, R or Python Pandas) support some form
of tables (which R calls "data frames" and I think Pandas has copied that
nomenclature) so they are a pre-existing *lingua franca* of data. We should
probably start by building on these, with an eye to where we want to go.

[^vectors]: That's “vector” in the data science sense of “sequence indexed by
that naturals up to some $N$” rather than in the mathematical sense of “things
for which one has the concepts of addition and multiplication by reals.“

## Semantic annotations

The semantics of data frames as understood by R and Python is rather
impoverished. What is known in these systems is roughly that each column has a
particular primitive “type” or “class” -- for example, string, float, integer,
possibly Boolean.

Real-world data contain a greater richness of meaning than these simple
representations allow.

### The meaning of primitive types

At present, many things are left unsaid which should be made explicit in order
to communicate between different languages or assistants. One example is the
size and format of the representation of real numbers (16 bits, 32 bits,
arbitrary precision, arbitrary size, fixed precision decimal, and so on), or
whether non-numeric values such as “`NaN`” and “`+Inf`” are included.

Since we intend to allow communications between different systems, we had be
able to communicate the representations available (and possibly translate
between them).

### Interpretations of data represented as primitive types

Real-world data has an interpretation. A particular float might be a
representation of a *real number*, specifically a *physical measurement*,
specifically a *temperature*, specifically a temperature *in Celsius*.

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

However, most systems (especially databases!) insist that every column is a
single type, apart possibly from a distinguished, catch-all, “out of type”
value, such as R's “`NA`” value, or SQL's “`NULL`”, the semantics of which is
poorly-understood. 

The other kind of complex data occurs when a single “value” is composed of more
than one primitive value, typically from multiple columns. An example is
geographical coordinates, with longitude in one column and latitude in another.

### “User-defined“ annotations

A new system may fail to be used because it is too inflexible and cannot be
extended in ways that the user requires for her particular purpose. On the other
hand, it can also fail because the extension mechanism is *too* open, providing
no semantic guarantees or constraints. Navigating this strait is not trivial.


## Distinction between data and representation.





# Overview of the Relational Model




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
