---
layout: documentation
doctitle: Transactional cursor based API
---

### Maven artifacts

First, you have to get the decency on our Sirix core project. At this stage of development please use the latest SNAPSHOT artifacts from the OSS snapshot repository. Just add the following repository section to your POM file:

```xml
<repository>
  <id>sonatype-nexus-snapshots</id>
  <name>Sonatype Nexus Snapshots</name>
  <url>https://oss.sonatype.org/content/repositories/snapshots</url>
  <releases>
    <enabled>false</enabled>
  </releases>
  <snapshots>
    <enabled>true</enabled>
  </snapshots>
</repository>
```

Or for Gradle:
```gradle
apply plugin: 'java'
apply plugin: 'maven'

repositories {
    maven {
          url "https://oss.sonatype.org/content/repositories/snapshot"
    }
}
```

Maven artifacts are deployed to the central maven repository (however please use the SNAPSHOT-variants as of now). Currently the following artifacts are available. Make sure that snapshots are getting updated with newer versions in your IDE.

Core project:

```xml
<dependency>
  <groupId>io.sirix</groupId>
  <artifactId>sirix-core</artifactId>
  <version>0.9.0-SNAPSHOT</version>
</dependency>
```

To add the dependency in Gradle:
```gradle
dependencies {
  compile 'io.sirix:sirix-core:0.9.0-SNAPSHOT'
}
```

### Tree-Encoding in Sirix
Our encoding is pretty simple. We do not use range-encodings or hierarchical labels (the latter optionally for imported XML-documents in order to provide fast document order determination). Instead, every node once stored in Sirix references other nodes by a `firstChild`/`leftSibling`/`rightSibling`/`parent`/`nodeKey` encoding. Think of it as a persistent DOM.

<div class="img_container">
![encoding](images/encoding.png)
</div>

The numbers are auto-generated unique, stable node-IDs or nodeKeys. Every structural node might have a first child, a left sibling, a right sibling and a parent node. Namespace- and attributes- are the only non-structural nodes in XML/XDM data. They just have a parent pointer. In our JSON to tree mapping however, every node is a structural node.

Note that we're able to store both XML- and JSON-documents in the same encoding.

### Create a database with a single resource file
First, we want to show how to create a database with a single resource (the resource is going to be imported from an XML-document and shredded into our internal format).

```java
// XML-file to import.
final var pathToXmlFile = Paths.get("xmlFile");

// Create database configuration.
final var databaseFile = Paths.get("database");
final var dbConfig = new DatabaseConfiguration(databaseFile);

// Create a new lightweight database structure.
Databases.createXdmDatabase(dbConfig);

// Open the database.
try (final var database = Databases.openXdmDatabase(databaseFile)) {
  // Create a first resource without text-value compression but with DeweyIDs which are hierarchical node labels.
  database.createResource(ResourceConfiguration.builder("resource").useTextCompression(false).useDeweyIDs(true).build());

  try (// Open a resource manager.
       final var manager = database.openResourceManager("resource");
       // Open only write transaction on the resource (transaction provides a cursor for navigation
       // through moveToX-methods).
       final var wtx = manager.beginNodeTrx();
       final var fis = new FileInputStream(pathToXmlFile.toFile())) {
       
       // Import an XML document.
       wtx.insertSubtreeAsFirstChild(XmlShredder.createFileReader(fis));

       // Commit and persist the changes.
       wtx.commit();
  }
}
```

In order to import a single JSON file we almost do the same:

```java
// JSON-file to import.
final var pathToJsonFile = Paths.get("jsonFile");

// Create database configuration.
final var databaseFile = Paths.get("database");
final var dbConfig = new DatabaseConfiguration(databaseFile);

// Create a new lightweight database structure.
Databases.createJsonDatabase(dbConfig);

// Open the database.
try (final var database = Databases.openJsonDatabase(databaseFile)) {
  // Create a first resource without text-value compression but with DeweyIDs which are hierarchical node labels.
  database.createResource(ResourceConfiguration.builder("resource").useTextCompression(false).useDeweyIDs(true).build());

  try (// Open a resource manager.
       final var manager = database.openResourceManager("resource");
       // Open only write transaction on the resource (transaction provides a cursor for navigation
       // through moveToX-methods).
       final var wtx = manager.beginNodeTrx();
       final var fis = new FileInputStream(pathToJsonFile.toFile())) {
       
       // Import a JSON-document.
       wtx.insertSubtreeAsFirstChild(XmlShredder.createFileReader(fis));

       // Commit and persist the changes.
       wtx.commit();
  }
}
```

### Open the database and the resource manager again

Now, that we have imported a first resource and persisted it in our binary-structure, we are able to open it again at any time (alternatively the single node `read/write-transaction` handle can be reused after issuing the commit).

```java
// Open the database.
try (final var database = Databases.openXdmDatabase(databaseFile);
     final var manager = database.openResourceManager("resource");
     // Now open a read-only transaction again.
     final var rtx = manager.beginNodeReadOnlyTrx()) {
    
  // Use the descendant axis to iterate over all structural descendant nodes (each node with the exception of namespace- and attribute-nodes) in pre-order (depth-first).
  new DescendantAxis(rtx, IncludeSelf.YES).forEach((unused) -> {
    // The transaction-cursor is moved to each structural node (all nodes, except for namespace- and attributes in preorder).
    switch (rtx.getKind()) {
      case ELEMENT:
        // In order to process namespace-nodes we could do the following and log the full qualified name of each node.
        for (int i = 0, nspCount = rtx.getNamespaceCount(); i < nspCount; i++) {
          rtx.moveToNamespace(i);
          LOGGER.info(rtx.getName());
          rtx.moveToParent();
        }

        // In order to process attribute-nodes we could do the following and log the full qualified name of each node. 
        for (int i = 0, attrCount = rtx.getAttributeCount(); i < attrCount; i++) {
          rtx.moveToAttribute(i);
          LOGGER.info("Attribute name:" + rtx.getName());
          LOGGER.info("Attribute value:" + rtx.getValue());
          rtx.moveToParent();
        }
        break;
        
        LOGGER.info(rtx.getDescendantCount());
        LOGGER.info(rtx.getChildCount());
        /* 
         * Hash of a node, build bottom up for all nodes (depends on descendant hashes, however only
         * ancestor nodes are updated during a normal edit-operation. During bulk inserts with 
         * insertSubtree(...) the hashes are generated during a postorder-traversal, just like the 
         * descendant-count of each structural node.
         */
        LOGGER.info(rtx.getHash());
        break;
      case COMMENT:
        LOGGER.info(rtx.getValue());
        break;
      case TEXT:
        // Log the text-value.
        LOGGER.info(rtx.getValue());
        break;
      // Other node types omitted.
      default:
        // Do nothing.
    };
  });
}
```

In JSON we obvisouly have no namespace- or attribute-nodes, but with this exception the axis can be used in the same way:

```java
// Open the database.
try (final var database = Databases.openJsonDatabase(databaseFile);
     final var manager = database.openResourceManager("resource");
     // Now open a read-only transaction again.
     final var rtx = manager.beginNodeReadOnlyTrx()) {
    
  // Use the descendant axis to iterate over all structural descendant nodes (each node with the exception of namespace- and attribute-nodes) in pre-order (depth-first).
  new DescendantAxis(rtx, IncludeSelf.YES).forEach((unused) -> {
    // The transaction-cursor is moved to each structural node (all nodes, except for namespace- and attributes in preorder).
    switch (rtx.getKind()) {
      case OBJECT_KEY:
        LOGGER.info(rtx.getName());
        break;
      case STRING_VALUE:
      case BOOLEAN_VALUE:
      case NUMBER_VALUE:
      case NULL_VALUE:
        LOGGER.info(rtx.getValue());
        break;
      default:
        // ARRAY- and OBJECT-nodes omitted.
    }
  });
```

### Axis to navigate in space and in time
However as this is such a common case to iterate over structual and non-structural nodes as for instance namespace- and attribute-nodes we also provide a simple wrapper axis:

```java
new NonStructuralWrapperAxis(new DescendantAxis(rtx))
```

For sure we also have a `NamespaceAxis` and an `AttributeAxis`.

As it's very common to do something based on the different node-types we implemented the visitor pattern. As such you can simply plugin a visitor in another `descendant-axis` called `VisitorDescendantAxis`, which is a special axis taking care of the return types from the visit-methods. A visitor must implement methods as follows for each node:

```java
/**
 * Do something when visiting a {@link ImmutableElement}.
 * 
 * @param node the {@link ImmutableElement}
 */
VisitResult visit(ImmutableElement node);
```

The only implementation of the `VisitResult` interface is the following enum:

```java
/**
 * The result type of an {@link XdmNodeVisitor} implementation.
 * 
 * @author Johannes Lichtenberger, University of Konstanz
 */
public enum VisitResultType implements VisitResult {
  /** Continue without visiting the siblings of this node. */
  SKIPSIBLINGS,

  /** Continue without visiting the descendants of this node. */
  SKIPSUBTREE,

  /** Continue traversal. */
  CONTINUE,

  /** Terminate traversal. */
  TERMINATE
}
```

The `VisitorDescendantAxis` takes care and skips a whole subtree if the return type is `VisitResultType.SKIPSUBTREE`, or skips the traversal of all further right-siblings of the current node (`VisitResultType.SKIPSIBLINGS`). You can also terminate the whole traversal with `VisitResultType.TERMINATE`.

The default implementation of each method in the `Visitor`-interface returns `VisitResultType.CONTINUE` for each node-type, such that you only have to implement the methods (for the nodes), which you're interested in. If you've implemented a class called `MyVisitor` you can use the `VisitorDescendantAxis` in the following way:

```java
// Executes a modification visitor for each descendant node.
final var axis = VisitorDescendantAxis.newBuilder(rtx).includeSelf().visitor(new MyVisitor()).build();
     
while (axis.hasNext()) axis.next();
```

We provide all possible `XPath` axis. Note, that the `PrecedingAxis` and the `PrecedingSiblingAxis` do not deliver nodes in document order, but in the natural encountered order. Furthermore a `PostOrderAxis` is available, which traverses the tree in a postorder traversal. Similarly a `LevelOrderAxis` traverses the tree in a breath first manner.

We provide several filters, which can be plugged in through a `FilterAxis`. The following code for instance traverses all children of a node and filters them for nodes with the local name "a".

```java
new FilterAxis<XdmNodeReadOnlyTrx>(new ChildAxis(rtx), new NameFilter(rtx, new QNm("a")))
```

The `FilterAxis` optionally takes more than one filter. The filter either is a `NameFilter`, to filter for names as for instance in elements and attributes, a value filter to filter text nodes or a node kind filter (`AttributeFilter`, `NamespaceFilter`, `CommentFilter`, `DocumentRootNodeFilter`, `ElementFilter`, `TextFilter` or `PIFilter` to filter processing instruction nodes).

It can be used as follows:

```java
// Filter by name (first argument is the axis, next arguments are filters (which implement org.sirix.axis.filter.Filter).
for (final var axis = new FilterAxis<XdmNodeReadOnlyTrx>(new VisitorDescendantAxis.Builder(rtx).includeSelf().visitor(myVisitor).build(), new NameFilter(rtx, "foobar")); axis.hasNext();) {
  axis.next();
}
```

Alternatively you could simply stream over your axis (without using the `FilterAxis` at all) and then filter by predicate. `rtx` is a `NodeReadOnlyTrx` in the following example:

```java
final var axis = new PostOrderAxis(rtx);
final var axisStream = StreamSupport.stream(axis.spliterator(), false);

axisStream.filter(unusedNodeKey -> new NameFilter(rtx, new QNm("a"))).forEach((unused) -> /* Do something with the transactional cursor */);
```

In order to achieve much more query power you can chain several axis with the `NestedAxis`. The following example shows how a simple XPath query can be processed. However, we think it's much more convenient simply use the XPath query with our Brackit binding.

```java
// XPath expression /p:a/b/text()
// Part: /p:a
final var childA = new FilterAxis(new ChildAxis(rtx), new NameFilter(rtx, "p:a"));
// Part: /b
final var childB = new FilterAxis(new ChildAxis(rtx), new NameFilter(rtx, "b"));
// Part: /text()
final var text = new FilterAxis(new ChildAxis(rtx), new TextFilter(rtx));
// Part: /p:a/b/text()
final var axis = new NestedAxis(new NestedAxis(childA, childB), text);
```

Simple examples with the `PostOrderAxis`:

whole bunch of axis are usable (all XPath axis and a few more):

```java
// A postorder-axis which iterates in postorder through a (sub)tree.
final var axis = new PostOrderAxis<XdmNodeReadOnlyTrx>(rtx); 
while (axis.hasNext()) {
  // Unique node identifier (nodeKey) however not needed here.
  final long nodeKey = axis.next();
  // axis.getTrx() or directly use rtx.
  switch(axis.getTrx().getKind()) {
  case TEXT:
    // Do something.
    break;
  }
}
```

or more elegantly:

```java
// Iterate and use a visitor implementation to describe the behavior for the individual node types.
final var visitor = new MyVisitor(rtx);
final var axis = new PostOrderAxis<XdmNodeReadOnlyTrx>(rtx); 
while (axis.hasNext()) {
  axis.next();
  rtx.acceptVisitor(visitor);
}
```

or with the foreach-loop:
```java
// Iterate and use a visitor implementation to describe the behavior for the individual node types.
final var visitor = new MyVisitor(rtx);
for (final long nodeKey : new PostOrderAxis(rtx)) {
  rtx.acceptVisitor(visitor);
}
```

#### Concurrent Axis
We also provide a ConcurrentAxis to fetch nodes concurrently. In order to execute an XPath-query as for instance `//regions/africa//location` it would look like that:

```java
final Axis axis = new NestedAxis(
        new NestedAxis(
            new ConcurrentAxis(firstConcurrRtx,
                new FilterAxis(new DescendantAxis(firstRtx, IncludeSelf.YES),
                    new NameFilter(firstRtx, "regions"))),
            new ConcurrentAxis(secondConcurrRtx,
                new FilterAxis(new ChildAxis(secondRtx), new NameFilter(secondRtx, "africa")))),
        new ConcurrentAxis(thirdConcurrRtx, new FilterAxis(
            new DescendantAxis(thirdRtx, IncludeSelf.YES), new NameFilter(thirdRtx, "location"))));
```

Note and beware of the different transactional cursors as constructor parameters (all opened on the same revision). We also provide a `ConcurrentUnionAxis`, a `ConcurrentExceptAxis` and a `ConcurrentIntersectAxis`.

#### Predicate Axis
In order to test for a predicate for instance select all nodes which have a child element with name "foo" you could use:

```java
final var childAxisFilter = new FilterAxis(new ChildAxis(rtx), new ElementFilter(rtx), new NameFilter("foo"));
final var descendantAxis = new DescendantAxis();
final var predicateAxisFilter = new PredicateAxis(rtx, childAxisFilter);
final var nestedAxis = new NestedAxis(descendantAxis, predicateAxisFilter);
```

#### Time Travel axis
However, we not only support navigational axis within one revision, we also allow navigation on the time axis.

For instance you can use one of the following axis to navigate in time:
`FirstAxis`, `LastAxis`, `PreviousAxis`, `NextAxis`, `AllTimeAxis`, `FutureAxis`, `PastAxis`.

Each of the constructors of these time-travel axis takes a transactional cursor as the only parameter and opens the node, the cursor currently points to in each of the revisions (if it exists).

In order to use time travel axis, however first a few more revisions have to be created through committing a bunch of changes.

### Open and modify a resource in a database
First, we have to open the resource again. For XDM (with XML-data) databases it is:
```java
// Open the database.
try (final var database = Databases.openXdmDatabase(databaseFile);
     final var manager = database.openResourceManager("resource");
     // Now open a read/write transaction again.
     final var wtx = manager.beginNodeTrx()) {
  ...
}
```

For JSON databases it is:

```java
// Open the database.
try (final var database = Databases.openJsonDatabase(databaseFile);
     final var manager = database.openResourceManager("resource");
     // Now open a read/write transaction again.
     final var wtx = manager.beginNodeTrx()) {
  ...
}
```

Note, that we now have to use a transaction, which is able to modify stuff (`NodeTrx`) instead of a read-only transaction.

We can then navigate to a specific node, either via axis, filters and so on or if we know the node key simply through the method `moveTo(long)` whereas the long parameter is the node key of the node we want to select.

We provide several navigational primitives/methods. After the resource/document is opened the cursor sits at a document root node, which is a node, which is present after bootstrapping a resource. We are then able to navigate to it's first child which is the XML root element via `moveToFirstChild()`. Similar, we can move to a right sibling with `moveToRightSibling()`, or move to the left sibling (`moveToLeftSibling()`). Furthermore many more methods to navigate through the tree are available. For instance `moveToLastChild()` or `moveToAttribute(int)`/`moveToAttributeByName(new QNm("foobar"))`/`moveToNamespace(int)` if we reside an element node. Further more we added the ability to move to the next node in preorder (`moveToNext()`) or to the previous node in preorder (`moveToPrevious()`). Or for instance to the next node on the XPath `following::`-axis. A simple example is this:

```java
// A fluent call would be if you know a node has a right sibling and there's a first child of the right sibling.
rtx.moveToRightSibling().getCursor().moveToFirstChild().getCursor();

// Can be tested before.
if (rtx.hasRightSibling()) {
  rtx.moveToRightSibling();
}

// Or test afterwards.
if (rtx.moveToRightSibling().hasMoved()) {
  // Do something.
}

// Move to next node in the XPath following::-axis.
rtx.moveToNextFollowing();

// Move to previous node in preorder.
rtx.moveToPrevious();

// Move to next node in preorder.
rtx.moveToNext();
```

The API is fluent:

```java
// getCursor() returns the transaction cursor currently used. However in this case the caller must be sure that a right sibling of the node denoted by node-key 15 and his right sibling and the right sibling's first child exists.
wtx.moveTo(15).getCursor().moveToRightSibling().getCursor().moveToFirstChild().getCursor().insertCommentAsFirstChild("foo");
```

Once we navigated to the node, we are able to either update for instance the name or the value (depending on the node type).

```java
// Cursor for instance points to an element node.
if (wtx.isElement()) wtx.setName(new QNm("foo"))

// Or a text node
if (wtx.isText()) wtx.setValue("foo")
```

Or we can insert new elements via `insertElementAsFirstChild(new QNm("foo"))`/`insertElementAsLeftSibling(new QNm("foo"))`/`insertElementAsRightSibling(new QNm("foo"))`. Similar methods exist for all other node types. We for sure always check for consistency and if calling the method on a specific node type should be allowed or not.

Attributes for instance can only be inserted (`insertAttribute(new QNm("name", "value"))`), if the cursor is located on an element node.

More sophisticated bulk insertion methods exist, too (as you have already seen when we imported an XML-document). We provide a method to insert an XML-fragment as a first child (`XdmNodeTrx insertSubtreeAsFirstChild(XMLEventReader)`), as a left sibling (`XdmNodeTrx insertSubtreeAsLeftSibling(XMLEventReader)`) and as a right sibling (`XdmNodeTrx insertSubtreeAsRightSibling(XMLEventReader)`).

To insert a new subtree based on a String you can simply use

```java
wtx.insertSubtreeAsFirstChild(XmlShredder.createStringReader("<foo>bar<baz/></foo>"))`
```

Updating methods can be chained:

```java
// Assertion: wtx is located at element node.
wtx.insertAttribute(new QNm("foo"), "bar", Move.PARENT).insertElementAsRightSibling(new QNm("baz"));

// Copy subtree of the node the read-transaction is located at as a new right sibling.
wtx.copySubtreeAsRightSibling(rtx);
```

Changes are always done in-memory and only ever flushed to disk or the flash drive on a transaction commit. You can either commit or rollback the transaction:

```java
wtx.commit() or wtx.rollback()
```

Note that the transaction handle simply can be reused after a `commit()` or `rollback()` method call.

You can also start an auto-commit transactional cursor:

```java
// Auto-commit every 30 seconds.
resourceManager.beginNodeTrx(TimeUnit.SECONDS, 30);
// Auto-commit after every 1000th modification.
resourceManager.beginNodeTrx(1000);
// Auto-commit every 30 seconds and every 1000th modification.
resourceManager.beginNodeTrx(1000, TimeUnit.SECONDS, 30);
```

You're also able to start a read/write Transaction and then revert to a former revision:

```java
// Open a read/write transaction on the most recent revision, then revert to revision two and commit as a new revision.
resourceManager.beginNodeTrx().revertTo(2).commit()
```
Once we have committed more than one revision we can open it either by specifying the exact revision number or by a timestamp in which case the revision, which has been stored closest to the given timestamp is opened.

```java
// To open a transactional read-only cursor on revision two.
final var rtx = resourceManager.beginNodeReadOnlyTrx(2)

// Or by a timestamp:
final var dateTime = LocalDateTime.of(2019, Month.JUNE, 15, 13, 39);
final var instant = dateTime.atZone(ZoneId.of("Europe/Berlin")).toInstant();
final var rtx = resourceManager.beginNodeReadOnlyTrx(instant)
```

### Seialize as XML

In order to serialize the (most recent) revision as XML pretty printed to STDOUT:

```java
final var serializer = XmlSerializer.newBuilder(manager, System.out).prettyPrint().build();
serializer.call();
```

Or write it to string:

```java
final var baos = new ByteArrayOutputStream();
final var writer = new PrintStream(baos);
final var serializer = XmlSerializer.newBuilder(manager, writer).prettyPrint().build();
serializer.call();
final var content = baos.toString(StandardCharsets.UTF8);
```

In order to serialize revision 1, 2 and 3 of a resource with an XML declaration and the internal node keys for element nodes:

```java
final var serializer = XmlSerializer.newBuilder(manager, out, 1, 2, 3).emitXMLDeclaration().emitIds().build();
serialize.call()
```

### Import differences between an initially stored revision and a second version of a XML-document

To update a resource with algorithmically found differences between two tree-structures, use something like the following:

```java
// Old Sirix resource to update.
final var resOldRev = Paths.get("resource");

// XML document which should be imported as the new revision.
final var resNewRev = Paths.get("foo.xml");

// Determine and import differences between the sirix resource and the
// provided XML document.
FMSEImport.xdmDataImport(resOldRev, resNewRev);
```
