# Wrattler beta - goals and architecture notes

This document records some of the discussion betwen Charles and Tomas on what we would
like to achieve in the first (beta) version of Wrattler and includes a few notes on
the architecture. 

## Goals - what do we want to do?

 * **JupyterLab integration.** We believe that integrating with JupyterLab is 
   important in order to make Wrattler a viable open-source project and an easy-to-use
   tool. This also gives us a couple of great things for free (see below). 
   For these reasons, Wrattler should be "another kind of notebook" that can be 
   created in JupyterLab. Wrattler components should not be tightly bound to 
   JupyterLab, but we think JupyterLab is primary vehicle for using Wrattler.
   
 * **Sample polyglot data analysis.** We need to create a sample data analysis that
   illustrates some of the capabilities of Wrattler. This means it needs to be 
   polyglot (using Python together with JavaScript and TheGamma). Optionally, it might
   also use an AI assistant such as datadiff or ptype. It will share data via a 
   data store across multiple cells and have parts in TheGamma that will give 
   live preview during editing.
 
 * **JupyterLab friendly kernels.** We need to investigate to what extent can we
   reuse existing JupyterLab kernels. Ideally, Wrattler should be able to build on
   top of existing kernels but insert some shim code when calling the kernel to 
   read/write data from/to data store. This way, we can reuse the existing 
   Jupyter infrastructure for spawning kernels. (Available through Binder.)
 
 * **Datastore prototype.** The beta version should help us clarify requirements for
   the data store. In the prototype, the datastore keeps data as JSON blobs in 
   Azure data storage. Users might want to use other things such as SQL databases.
   We should think about how datastore can provide a thin layer over, say, SQL
   database while still keeping track of data provenance.
   
## Work items - what needs to be done?

There is a number of work items or sub-projects that need to be completed for 
the beta version. I put the names of people most likely involved in parentheses,
but this is just a proposal and we can change that.

 1. **Doing a sample data analysis (Giovani).** We need to identify a suitable 
    data set (preferably a public one and preferably from the list of AIDA data
    sets) and do a data analysis that answers a question about the data set.
    This should mix some Python (for analysis), TheGamma (for data exploration)
    and JavaScript (for data visualization). Most of the work can be done 
    independently of other work items and gradually integrated with Wrattler.
 
 2. **Implementing Python kernel (Nick).** To use Python as the backend, we 
    need a Python kernel. This should provide two endpoints (for getting 
    code snippet imports and exports and for running code) as described in 
    `2018-04-07-prototype-notes.md`. This can start as a stand-alone console
    application, but we should investigate how to do this on top of Jupyter
    kernels (see below). 
 
 3. **Refactoring the Notebook system (May, Tomas).** We will split the current
    client-side codebase into one entry point (which handles the overall GUI)
    and language plugins (that build parts of the dependency graph and create
    language-specific GUI). The main part will be in TypeScript, language 
    plugins can be in F# (for TheGamma) or anything else.
 
 4. **Proper datastore (James).** The data store should provide a way for passing
    dataframes between multiple langauges (as currently), a way of annotating 
    dataframes with meta-data (such as inferred types) and a way of managing 
    access to raw files (downloaded from the internet) or SQL databases (so that
    those can be used, but their usage is recorded in the dependency graph).
 
 5. **JupyterLab integration (TBA).** In order to integrate with JupyterLab, 
    we will need two things: 1) JupyterLab plugin that loads Wrattler when
    a file representing Wrattler notebook is opened, 2) a way of starting 
    Wrattler kernels on top of JupyterLab (possibly by adding some shim code
    to existing Juptyer kernels).
 
## Milestones - how do we get there?

The following breaks the work items into a number of steps. We need to produce a
public demo by June, so STEP 1 lists what is needed to get that. STEP 2 and STEP 3
are next steps, getting towards beta with full JupyterLab integration.

### STEP 1 - Getting started

 - Everything can be hosted in Azure, so that we have a public demo
 - Using (new) Python and (existing) TheGamma and JavaScript kernels
 - Using (perhaps existing simple Azure blob storage based) data store
 - Given (documented) pre-requisites, anyone can run Wrattler on their machine

### STEP 2 - JuptyerLab integration

 - The same as in Step 1 can be done inside JuptyerLab
 - The Python kernel is handled by JupyterLab, possibly using Jupyter Python kernel
 - The Wrattler GUI runs as a JupyterLab plugin (possibly using the F# prototype codebase)
 - The data store is also handled by JupyterLab, but as an independent process 
 - (We might not have a good story for permanent data storage)

### STEP 3 - Public JupyterLab demo

 - We have a public demo that can be launched via Binder 
 - Anyone can click a link, which starts Wrattler inside JupyterLab
 - We now have JuptyerLab integrated Python (and more) kernels
 - We have a good story for permanent data storage (cloud based and/or file based)

## More notes on the architecture

### Wrattler and Jupyter kernels

Wrattler kernels have two end-points - analyze a given code to get imports and exports
and evaluate given code using given inputs (stored in data store). The operations do 
not have side-effect (on the state of the kernel). They might read data from the data
store and create new dataframes in the data store, but they do not mutate anything.

In contrast, Jupyter kernels have a command to run code, which affects the state of
the kernel. However, we should be able to implement Wrattler kernels on top of 
Jupyter kernels - all we need is to load some prelude code when starting the kernel
and then wrapping the calls to the kernel in some code that first loads imports, 
then runs the actual code and then writes the exports.

Say, I want to run `someSmall <- head(3, someLarge)` to get a new data frame with 
top 3 rows from another data frame. We could define functions `getImportsAndExports`
and `loadFromDataStore` with `writeToDataStore` and run the following code to 
get exports/imports of a code:

```
getImportsAndExports("someSmall <- head(3, someLarge)")
```

And run the following code to evaluate the command:

```
someLarge <- loadFromDataStore("http://.../12345")
someSmall <- head(3, someLarge)
writeToDataStore("http://.../67891")
```

These would get evaluated by ordinary Jupyter kernel in an ordinary way, but they
would actually implement the behaviour we need for Wrattler, i.e. return imported
and exported variables in the first case and use the data store in the latter case.

### Data Store

Here are things that we want from the data store:

 * Ability for analysts to transparently store data in a format that makes sense
   to them, e.g. as a relational DB, as a csv files, or what have you.
   This seems better than inventing our own data format, which would be a lot of work
   for little gain, and make   people nervous about trying the system.
   
 * The data store should record provenance. When I load a Wrattler data store that
    someone else has created, I should be able to load the raw data, see any transformations
    that have been made, and see which depend on what.

 * (optional) It could be that adding provenance can be done in a way that makes it easy
    to record other Wrattler specific meta-data, such as free-text comments from the analyst,
    links to Ontologies, etc
    
One way to meet these objectives would be for a "data store" to be a special directory, like an app package on MacOS.
The directory would contain:

  * The current version of the data in its "native format", e.g. a csv file
  * The "original version" of the data
  * The version history, i.e., all transformation that have been applied to the data. This could be in the form of "notebook cells", i.e. a snippet of code that can be run by a Kernel
  * Other annotations of the data.

The data in the data store would be accessed from the kernels.  
There's a question of how to do this --- in order to track provenance, we need to trace reads and writes to the data store.
One way to do this would be to create our own wrappers for standard libraries (`readr`, `read.csv`, and the like),
but this might pose an obstacle to adoption. Perhaps the best way would be to track this at runtime. e.g., We have our
own wrapper code that runs after the kernel has executed each cell, and this checks via hashing whether the data has changed,
updating the provenance graph accordingly.

The provenance information needs to be accessed from the front end in order to support live preview, recomputation, and refactoring.
However, it might still be good to have the "official version" of the provenance in the data store, so that the data store can be 
shared  independent of the front end. The first thought for storing provenance would be for us to create a graph representation, where
each edge of the graph is annotated with the snippet of code that causes the change, and each node corresponds to the part of the data store that is updated. There is a question of what granularity to
use, in two different senses:

* Edge granularity: Should we track "Notebook cell 31412341" updated the data, or precisely which method call, or something in between
* Node granularity: Should we track provenance separately for each cell? For each row? Column? Table?

We probably want to do late binding on both of these issues. Maybe a lot of use cases could be covered by having the edge granularity at the cell level, and allowing the nodes to be rows, columns, or tables. 
But there's a lot of thinking to be done about this (and we can also look at what's going on in the provenance community).


### Why and how of JupterLab integration

There are a couple of social benefits for integrating with JupyterLab - it is a
potentially popular system, with a lot of activity around it and it will make
Wrattler more visible to the data science community.

On the technical side, perhaps the nicest thing we get "for free" is that we will
be able to run Wrattler via Binder - Binder is a tool that spawns JupyterLab on 
a Kubernetes cluster and the people working on Binder even have a free cloud-hosted
service that lets anyone do this (we could run our own on Azure too). Just by
clicking on a link, you get a new instance of JupyterLab running in the cloud
with all the kernels started. This would be super neat for letting people play
with Wrattler.

At the same time, we should keep in mind other options for running Wrattler - it would
be nice if it could also be used as a plugin for Atom or other editors. This
means keeping the JupyterLab integration reasonably separate and the core API
clean.
