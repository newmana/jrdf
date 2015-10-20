# Persistent JRDF Graphs #

JRDF has evolved to support basic persistence of graphs.  Graphs can be created using the PersistentJRDFFactoryImpl.  This requires a DirectoryHandler in order to be created.

## Lifecycle ##

The addition of persistence requires some extra care when using graphs.  If any graph operations are performed they are usually persisted immediately, although this is not necessarily guaranteed.  To ensure that all operations are persisted you must close all open resources related to a graph.

### Up to JRDF 0.5.6 ###
The lifecycle of a JRDF graph is currently tied to the factory that creates it.  Typically, the code to use is:
```
DirectoryHandler handler = new TempDirectoryHandler();
PersistentJRDFFactory factory = PersistentJRDFFactoryImpl.getFactory(handler);
try {
  Graph graph = factory.getGraph("graph1");
  ...
  graph operations
  ...
} finally {
  factory.close();
}
```

Until the factory is closed, the operations made on the graphs cannot be considered persisted to disk.  Graphs are not treated individually.  The persistence lifecycle still follows the original lifecyce, that is, load RDF, do some work, then save.

### JRDF 0.5.6.1 and above ###

The lifecycle of a graph changed with JRDF 0.5.6.1.  The graph is now responsible for closing it's own resources and factories no longer track open resources.  The state of a graph is only in a consistent state when you close all the resources being used.  The change is that the responsibility has moved from the factory to the graph.

The typical use of a graph becomes:
```
DirectoryHandler handler = new TempDirectoryHandler();
PersistentJRDFFactory factory = PersistentJRDFFactoryImpl.getFactory(handler);
Graph graph = factory.getGraph("graph1");
try {
  ...
  graph operations
  ...
} finally {
  graph.close();
}
```

Graphs are not completely independent to each other, as they share a database used to map the numbers in their indexes to string values.  Graphs have their own indexes for triples (which are just 3 numbers) but share a database for mapping to their representation (for examples, in the index 1,1,1 represents the triple 

&lt;urn:foo&gt;

, 

&lt;urn:foo&gt;

, 

&lt;urn:foo&gt;

 - the map turns 1 into 

&lt;urn:foo&gt;

).

Allowing graphs to be persisted individually, where you can take move individual graphs around, is the eventual aim.

## Directory Handler ##
The DirectoryHandler interface is designed to manage the location where the indexes and system graph are stored.  It is used by the persistent graph factories.

Currently, there is only one DirectoryHandler implementation called TempDirectoryHandler.  It uses the current user name and system temporary directory property ("java.io.tmpdir") to create a directory.  For example, with a user name of "andrew" on Windows it creates a "C:\temp\jrdf\_andrew" directory.

## System Graph ##

The system graph is an NTriples file, "graphs.nt", that is stored with the other indexes.  Each named graph adds three statements to the file, for example:
```
_:a <http://jrdf.sf.net/name> "graph1" .
_:a <http://jrdf.sf.net/id> "1"^^<http://www.w3.org/2001/XMLSchema#long> .
_:a <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://jrdf.sf.net/graph> .
```

This indicates that the resources for "graph1" uses the id 1.  So there should be files associated with this id such as the indexes "spo1", "osp1" and "pos1".

All persistent graph implementations track the creation and deletion of named graphs.  The interface has the following methods that operate on the system graph:
  * hasGraph(String), which returns true if a graph exists with that name.
  * getNewGraph(String), which creates a new graph with the given name or fails if a graph with that name already exists.
  * getExistingGraph(String), which gets an existing graph or fails if the graph doesn't exist.
  * getGraph(String), which gets an existing graph or creates a new one if it doesn't exist.
  * getGraph(), which gets the graph "default" if it exists or creates it new.