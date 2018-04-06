# Wrattler prototype overview

This document gives a brief overview of the initial Wrattler prototype. There are four components:

 * **[Notebook running in the browser](https://github.com/wrattler/wrattler/tree/a4a8331eb077c5a6e8dfdaee0df6a8a90fd1417a)** - 
   this is the client-side code that runs as JavaScript in the web browser. It is
   written in F# and compiled to JavaScript using Fable (F# to JavaScript compiler).
   The browser part creates dependency graph and walks over the graph to evaluate
   notebooks. It handles things like updating the graph when code changes and 
   invoking individual language kernles to run code.
   
 * **[Data store](https://github.com/wrattler/wrattler-data-store/tree/6c6e09ba428e2a84dc743fde09a869c37f39a039)** - 
   the data store stores data frames that are passed between cells of Wrattler notebooks.
   It is called by the browser (to fetch frames for a preview) and by language runtimes 
   (to load data before running code and store results back). Right now, this is 
   super simple - it just stores raw JSON blobs in Azure and loads them back (without
   any clever logic).
   
 * **[R service](https://github.com/wrattler/wrattler-r-service/tree/76199886474906638ec5a2ba7ddc9333e3a09639)** - 
   this is a sample language service. The browser component calls the R service to do
   two things: Analyse R code (get imported and exported data frames of a code snippet) 
   and to run R code (using data from data store). The implementation of the R service
   is in F# and invokes R via .NET interop with R, which is very fragile, so we will
   probably want to do this in a better way, but the current implementation shows
   the right API and some clever tricks (like R code analysis).
   
 * **[Datadiff type provider](https://github.com/wrattler/wrattler-datadiff-service/tree/bc2c738676d747f2b42f7f43f3008b72dc8c2b76)** - 
   this is a type provider for TheGamma that wraps datadiff and exposes it via
   "dot driven" API in TheGamma in Wrattler. Just like the data store and R service,
   this is just a simple HTTP service. When called with a URL of two data frames in 
   a data store, it generates members (shown in TheGamma completion lists) with 
   individual patches returned by datadiff.
   
## Running individual components

The current state of the prototype is that it "works on my machine". That is:

 - The R service works on Windows only. To run it, you can use `build.cmd`
 - The data store should work anywhere (using .NET on Windows and mono on Mac/Linux), 
   but you first need to create `src/config.fs` file with connection string to 
   Azure data storage. Something like:
   
        module Config
        let WrattlerDataStore = "DefaultEndpointsProtocol=https;AccountName=<storage>;AccountKey=<key>;EndpointSuffix=core.windows.net"
 
 - The browser component should also work anywhere, but it has a couple of dependencies.
   It is built using Fable, which has a [good getting started
   guide](http://fable.io/docs/getting-started.html). After you install the dependencies
   and clone the repo, the following should do the trick:
   
        cd wrattler
        yarn install
        dotnet restore
        cd src 
        dotnet fable yarn-start
        
   Once you do this, go to http://localhost:8080/broadband.html.
   
## Communication protocol

#### Data store

The data store is extremely simple and handles only two types of requests - to store data
frame and to load data frame. Assuming http://localhost:7102 is the URL where the data store
service is running, you can make a HTTP PUT request to store a data frame:

    $ curl --data '[{"Col1":123, "Col2":"Abc"},{"Col1":456, "Col2":"Def"}]' 
        -X PUT http://localhost:7102/699b2dff/variablename
    Created
    
And you can make a HTTP GET request to load a data frame:
    
    $ curl -X GET http://localhost:7102/699b2dff/variablename
    [{"Col1":123, "Col2":"Abc"},{"Col1":456, "Col2":"Def"}]

There are two interesting things here.

**Data format.** For now, the data format is just JSON (a list of records). The data stores
saves the data exactly as it gets it - so it does not even check it for correct format. The
values can be usual JSON values (numbers, strings, `null`). This is one thing we need
to improve in future versions!

**URL format.** The URL format is `http://<data-store-url>/<hash>/<variable-name>`. We expect 
that we can host the data store in the cloud or locally so that's what the `<data-store-url>`
will be. 

More interesting is the `<hash>`. This is a hash of a node in the dependency
graph maintained by Wrattler notebook (in the browser) that corresponds to this data 
frame. The hashes are calculated from dependencies in the graph, which guarantees that
if you change your code, you will get a new hash. This also means that the data frames
stored in the data store are immutable - once we store a data frame, it will never 
be changed. If you change code, you will get a new hash and the new data will be stored
as new data frames.

Finally, the URL also contains `<variable-name>` which is the name of a variable in 
a code block that created the data frame - this is useful if we have R code blocks. 
Wrattler does not understand R code blocks on the browser side, so it creates just one
hash for the block. The code block can generate multiple variables and those will be
stored under the same hash.

#### Language kernels

The client-side browser component of Wrattler has a simple plug-in model that lets 
people add support for new programming languages. The interface is mostly 
[defined here](https://github.com/wrattler/wrattler/blob/a4a8331eb077c5a6e8dfdaee0df6a8a90fd1417a/src/langs.fs), 
but this is very rough and we will need to clean this up. The key point is that each
language plugin is responsible for things like parsing code, analyzing it (which is needed
when creating dependency graphs), evaluating it and also rendering editors for it.
How this works is up to the client-side JavaScript component - however, I expect that 
we will have a couple of components for languages like R and Python that will delegate
all the work to some server-side kernel. This is exactly what the current R plugin does.
So for R, we have small amount of client-side code that makes calls to the R service
(running on the server) that does the actual work.

**Code analysis.** The current API for the R service exposes two endpoints. One for 
doing basic code analysis (to figure out what the dependencies are) and another for 
actually running the code. Assuming the R service lives at http://localhost:7101, you 
can make the following request to get imports (dependencies) and exports of a code block:

    $ curl --data 
        '{ "code":"top1 <- head(f1, 1)",
           "hash":"36373db7",
           "frames":["f1", "f2", "f3"]}'
        -X POST http://localhost:7101/exports
    { "exports": [ "top1" ],               
      "imports": [ "f1" ] }

Here, the request is a simple JSON with the following fields:

 - `code` is the source code; in the above demo, it takes top 1 row of a data frame `f1` 
   and stores the result in a data frame `top1`.
 - `hash` is a hash of the code block that is computed
 - `frames` is a list of data frames that are available in scope (exported from 
    previous cells).
    
The result of the call is a JSON with `imports` and `exports`. Imports is a list of
data frame names that are imported (here, just `f1` because the R service analyses
the code and recognizes that we are using just `f1` and not using `f2` or `f3`).
Exports is a list of newly defined data frames - that is `top1` here.    

**Evaluating code.** The second endpoint handles running of code. Assuming the R
service runs at http://localhost:7101 as before, we can make a HTTP POST request to 
the end-point http://localhost:7101/eval to run the previous code:

    $ curl --data 
        '{ "code":"top1 <- head(f1, 1)",
           "hash":"36373db7",
           "frames":[{"name":"f1", "url":"http://localhost:7102/699b2dff/variablename"}]}'
        -X POST http://localhost:7101/eval
    { "output": "",                                     
      "frames": [ { "name": "top1", "url": "http://localhost:7102/36373db7/top1" } ] }
      
The request is similar as before. It has the `code` and `hash` (both are the same as 
before). One change is that now, `frames` is not just a list of names, but it contains
data frame names together with their location in the data store. The above URL is the
one we created above when discussing data store (with a simple two-row data frame).

The result is a JSON with `output`, which is a console output (if the command printed
anything) and `frames` which is a list of frames exported by the code block. In the
above case, the R service runs the code, creates a data frame `top1` and stores it into
the data store (using the hash that was sent in the request and variable name based
on the source code). We can make a quick request to the data store to see that the code,
indeed, created a new data frame with the first row of our original data frame:
      
    $ curl -X GET http://localhost:7102/36373db7/top1
    [ { "Col1": 123, "Col2": "Abc" } ]
                            
## Browser component overview           

In this final section, I will try to give a couple of pointers for navigating around 
the [Wrattler client-side component](https://github.com/wrattler/wrattler/tree/a4a8331eb077c5a6e8dfdaee0df6a8a90fd1417a),
i.e. the F# code (compiled to JavaScript using Fable) that builds dependency graphs,
evaluates code and creates the user interface. This is quite big and undocumented, so 
the pointers might not be all that useful at this stage, but I hope they might help, 
at least a bit.

The following is based on the [current commit](https://github.com/wrattler/wrattler/tree/a4a8331eb077c5a6e8dfdaee0df6a8a90fd1417a),
so if you want to make sure you are looking at a version that matches the description,
open [the respository via this link](https://github.com/wrattler/wrattler/tree/a4a8331eb077c5a6e8dfdaee0df6a8a90fd1417a).

 - `/` - the root directory pretty much contains what the default Fable template generates
   this is configuration for yarn package manager and for webpack that is used to run 
   the F# to JS compilation in the background and runs a development web server.
   
 - `/src` - this is where all the source code is. The `wrattler.fsproj` file is an F#
   project file and defines the file order (in F#, file order matters because earlier
   files cannot see later files, so it makes sense to look at the files in the order
   listed here).
   
 - `/src/bindings` contains Fable bindings for calling various JavaScript libraries
   (not very interesting) and `/src/common` contains assorted helpers (also not very
   interesting) with the exception of `/src/common/datastore.fs` which handles 
   communication with the data store (fetching and storing data frames).

 - `/src/ast/ast.fs` defines the key data types that represent parsed notebook 
    (the `Block` type represents a code block) and the dependency graph 
    (the `Entity` type is a node in the graph and its `Kind` property stores 
    information about what kind of node it is and what are its dependencies) 
    (The `astops.fs` file has various helpers for working with those types).
    
 - `/langs.fs` defines interfaces that language plugins need to implement. This 
   is somewhat messy but `LanguagePlugin` is the key type and it needs to provide
   parsing function, binding operation (that constructs nodes of a dependency graph),
   interpreter and type checker (optional) for running and checking code and 
   editor (which creates all the user interface for a code block).
   
 - `/analyzer/binder.fs` implements some shared functionality that's used when
   constructing the dependency graph - mostly caching of nodes that ensures that
   we do not rerun the whole graph when a small change is made.
   
 - `/builtin/interpreter.fs` implements evaluators for R and JavaScript. Those are
   two functions that are called by the system - the input is a node from the graph
   (marked with the corresponding language). There are a couple of nodes for both
   R and JavaScript, so the evaluators need to handle them all (they represent the
   actual source code, each exported data frame and the whole code block).
   
 - `/rendering.fs` implements some common functionality that is used for rendering
   tables and creating editors. For editors, we're using Microsoft's Monaco editor.
   For all the interactivity, we are using a programming model based on the Elm 
   architecture (with state and events together with render and update functions).
 
 - `/gamma` contains all the code for TheGamma language plugin. This is very complicated
   and contains lots of code, but we can ignore that for now :-).
   
 - `/main.fs` implements the remaining bits of language plugins for R and 
   JavaScript (those are only hundred lines of code, but pretty confusing)
   and also the overall user interface that puts everything together and calls
   the individual language plugins to create editors for each blocks (there is also
   `debug.fs` which draws dependency graphs).
