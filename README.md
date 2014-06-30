PigeonKeeper
============

Installation
------------

    $ npm install pigeonkeeper


Quick Start
-----------

    var PigeonKeeper = require("pigeonkeeper");
    var GenericService = require("./node_modules/pigeonkeeper/lib/genericService");

    var mySharedData = {a:1, b:2};

    function finalCallback(err, data)
    {
        console.log("Current PK version: " + pk.getVersion());

        if(err)
        {
            console.log("In app.finalCallback, error object: " + JSON.stringify(err));
            console.log(pk.overallStateAsString());
            console.log(pk.getResults());
        }
        else
        {
            console.log("In app.finalCallback: all tasks are complete!");
            console.log(pk.overallStateAsString());
            console.log(pk.getResults());
        }
    }

    // Create generic services...
    var svc7 = new GenericService("7");
    var svc5 = new GenericService("5");
    var svc3 = new GenericService("3");
    var svc11 = new GenericService("11");
    var svc8 = new GenericService("8");
    var svc2 = new GenericService("2");
    var svc9 = new GenericService("9");
    var svc10 = new GenericService("10");


    var pk = new PigeonKeeper("MyPigeonKeeper", finalCallback, false, 5);

    // The digraph we're about to create matches the example given at http://en.wikipedia.org/wiki/Topological_sort
    // Create vertices and associate processes
    pk.addVertex("7", svc7, svc7.doStuff);
    pk.addVertex("5", svc5, svc5.doStuff);
    pk.addVertex("3", svc3, svc3.doStuff);
    pk.addVertex("11", svc11, svc11.doStuff);
    pk.addVertex("8", svc8, svc8.doStuff);
    pk.addVertex("2", svc2, svc2.doStuff);
    pk.addVertex("9", svc9, svc9.doStuff);
    pk.addVertex("10", svc10, svc10.doStuff);

    // Now add edges
    pk.addEdge("7", "11");
    pk.addEdge("7", "8");
    pk.addEdge("5", "11");
    pk.addEdge("3", "8");
    pk.addEdge("3", "10");
    pk.addEdge("11", "2");
    pk.addEdge("11", "9");
    pk.addEdge("11", "10");
    pk.addEdge("8", "9");

    // Kick it!
    pk.start(mySharedData);


Introduction
------------

### What is PK? ###

* A JS object for orchestrating the execution of processes or services


### What PK is NOT ###

* A workflow system
* A visual programming language


Concepts
--------

### Digraphs ###

* Short for "directed graphs"
* Consists of vertices and directed edges

![Example of a Digraph](https://developers.adp.com/PK/Digraph01.png "Example of a Digraph")

### Vertices ###
Each vertex has…

* An ID
* A data object
* A state (more about this later)


### Directed Edges ###

* Think: an arrow between two vertices
* Specified by a start vertex and an end vertex
* Parallel edges are NOT allowed


### Directed Acyclic Graph ###

* A digraph without cycles or self-loops
* Abbreviated DAG

![Example of a DAG](https://developers.adp.com/PK/AcyclicDigraph01.png "Example of a DAG")

From: [http://en.wikipedia.org/wiki/Topological_sort](http://en.wikipedia.org/wiki/Topological_sort "Wikipedia article on Topological Sort")

### Topological Sort ###

* An ordering of the vertices so that any vertex appears after its parents
* Not possible in digraphs with cycles (hence our focus on DAGs)
* Topological ordering need not be unique!

### Example ###
<table border="1">
<tr>
<td style="vertical-align: top"><img src="https://developers.adp.com/PK/AcyclicDigraph01.png" /></td>
<td style="vertical-align: top">Some topological orderings of this DAG include:
<ul>
<li>7, 5, 3, 11, 8, 2, 9, 10 (visual left-to-right, top-to-bottom)</li>
<li>3, 5, 7, 8, 11, 2, 9, 10 (smallest-numbered available vertex first)</li>
<li>3, 7, 8, 5, 11, 10, 2, 9 (because we can)</li>
<li>5, 7, 3, 8, 11, 10, 9, 2 (fewest edges first)</li>
<li>7, 5, 11, 3, 10, 8, 9, 2 (largest-numbered available vertex first)</li>
<li>7, 5, 11, 2, 3, 8, 9, 10 (attempting top-to-bottom, left-to-right)</li>
</ul>
</td>
</tr>
</table>
From: http://en.wikipedia.org/wiki/Topological_sort

### Kahn's Algorithm ###

* Used by PK for performing topological sort
* Run time is O(|V| + |E|) where:
*     |V| is number of vertices
*     |E| is number of directed edges
* Operates by removing parentless vertices (think: onion peeling)
* Finite acyclic digraphs must always have at least one parentless vertex, if |V| > 1
* Detects cycles!
* Is "destructive," so digraph must be deep-copied


### Processes ###

* A process is a JS EventEmitter
* Things that can happen inside a process:
*     HTTP requests, e.g. getting data from SOR
*     MongoDB or Solr/Lucene interactions
*     File operations
*     Synchronous operations, too
* PK is agnostic about how the results of the process are stored


### More About Processes ###

* Processes have a "start" or "run" or "doStuff" method
* As it goes, PK will call that method...
* ...and will pass in a “SharedData” object
* Processes emit two types of events:
*     "success" – includes data generated by the process
*     "error" – includes error object


### Processes and Vertices ###

* We associate a vertex with a process
* Vertex tells the process when to start running
* Process updates the vertex when the process completes


### Processes and Edges ###

* Process B depends on Process A means...A must be completed in order for B to start
* Example: B uses data fetched by A
* Since we associate processes with vertices, we describe dependencies between the processes with directed edges!


### SharedData ###

* SharedData is an object
* Intended to store config info...
* ...but can be used however the developers want
* Passed to PK when PK is started


### Processes and Shared Data ###

* PK passes (by reference) the SharedData object to each process when PK runs it
* Process can modify that object…
*     When process starts, or...
*     When process ends successfully, or...
*     When process ends in failure
* Idea is that processes can send config info  around from one process to another


### Again, What is PigeonKeeper? PigeonKeeper... ###

* Is a JS object for orchestrating the execution of processes
* Maintains association between vertices and processes
* Maintains dependencies between processes
* Performs topological sort
* Maintains a SharedData object
* Applies state transition rules (more in a sec)
* Runs processes based on state of associated vertices


### Available States ###

Each vertex can be in one of 5 states:

* NOT_READY
* READY
* IN_PROGRESS
* SUCCESS
* FAIL

Distinction between READY and IN_PROGRESS must be maintained if we wish to limit number of processes running in parallel.



### Lifecycle of a Vertex State ###
![Lifecycle of a Vertex State](https://developers.adp.com/PK/StateTransitionRules.png "Lifecycle of a Vertex State")


### Transition Rules ###
NOT_READY → READY when either...

* the vertex has no parents,
* or all parents are in SUCCESS state

READY → IN_PROGRESS

* controlled by PigeonKeeper

IN_PROGRESS → SUCCESS or FAIL

* depends on the process associated with the vertex

FAIL propagates to all children

### Example State Transitions ###

####Step 0####
![Initially, all states are NOT_READY](https://developers.adp.com/PK/StateTransitions00.png "Initially, all states are NOT_READY")

Initially, all states are NOT_READY


####Step 1####
![States without parents become READY](https://developers.adp.com/PK/StateTransitions01.png "States without parents become READY")

States without parents become READY


####Step 2####
![Processes for the READY states are executed, and so states become IN_PROGRESS](https://developers.adp.com/PK/StateTransitions02.png "Processes for the READY states are executed, and so states become IN_PROGRESS")

Processes for the READY states are executed, and so states become IN_PROGRESS


####Step 3####
![Some processes return OK, others fail](https://developers.adp.com/PK/StateTransitions03.png "Some processes return OK, others fail; Corresponding vertices’ states become SUCCESS or FAIL")

Some processes return OK, others fail; corresponding vertices’ states become SUCCESS or FAIL


####Step 4####
![Vertices with SUCCESSful parents become READY; Vertices with FAILed parents also FAIL](https://developers.adp.com/PK/StateTransitions04.png "Vertices with SUCCESSful parents become READY; Vertices with FAILed parents also FAIL")

Vertices with SUCCESSful parents become READY; vertices with FAILed parents also FAIL


####Step 5####
![READY vertices become IN_PROGRESS; FAILure propagates to children (sort of like the welfare state)](https://developers.adp.com/PK/StateTransitions05.png "READY vertices become IN_PROGRESS; FAILure propagates to children (sort of like the welfare state)")

READY vertices become IN_PROGRESS; FAILure propagates to children (sort of like the welfare state)


####Step 6####
![Process associated with vertex 11 is SUCCESSful...](https://developers.adp.com/PK/StateTransitions06.png "Process associated with vertex 11 is SUCCESSful...")

Process associated with vertex 11 is SUCCESSful...


####Step 7####
![...Child vertex 2 becomes READY...](https://developers.adp.com/PK/StateTransitions07.png "...Child vertex 2 becomes READY...")

...Child vertex 2 becomes READY...


####Step 8####
![...then IN_PROGRESS...](https://developers.adp.com/PK/StateTransitions08.png "...then IN_PROGRESS...")

...then IN_PROGRESS...


####Step 9####
![...and finishes SUCCESSfully.](https://developers.adp.com/PK/StateTransitions09.png "...and finishes SUCCESSfully.")

...and finishes SUCCESSfully.


### About FAIL... ###

PigeonKeeper has two ways of handling failure – specified in constructor:

1. Entire PigeonKeeper stops when one vertex FAILs, or...
2. Vertices which depend on a FAILed process are also marked as FAIL, but other processes can continue

Above example used option 2!

Usage Steps
-----------

### Overview ###

* Write processes as event emitters
* Write “finalProcess” method
* Create a PK instance
* Add vertices and associated processes
* Add directed edges
* Start PK!


Important Methods
-----------------

### Constructor ###

    PigeonKeeper(pkName, finalCallback, quitOnFailure, maxNumRunningProcesses, logger, userObject)

* pkName: instance name, which will be modified into a GUID
* finalCallback: function to be called when PK quits
* quitOnFailure: Boolean
*     When true, PK quits when a single process fails
*     When false, PK tries to execute all processes that don’t depend on failed processes
* maxNumRunningProcesses
* logger: a logging mechanism (optional)
* userObject: object for use by logging mechanism (optional)


### Commonly Used Methods ###

To create the digraph and associate processes with vertices, use...

    addVertex(vertexId, service, serviceStart)

    addEdge(startVertexId, endVertexId)

To start PK a'runnin', use...

    start(sharedData)

* Kicks-off the PK (begins applying state transition rules, starts processes, etc)
* sharedData is passed to processes when they are ran
* sharedData is intended to make config info available to the processes
* Processes can modify sharedData


License
-------

The MIT License (MIT)

Copyright (c) 2014 ADP, Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
