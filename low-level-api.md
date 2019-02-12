---
layout: default
---

## Transaction cursor based API

Some API examples:

```java
// Create database configuration.
final Path file = Paths.get("db");
final DatabaseConfiguration dbConfig = new DatabaseConfiguration(file);

// Create a new lightweight database structure.
Databases.createXdmDatabase(dbConfig);

// Open the database.
try (final Database database = Databases.openXdmDatabase(file)) {
  // Create a first resource without text-value compression but with DeweyIDs.
  database.createResource(ResourceConfiguration.builder("shredded").useTextCompression(false).useDeweyIDs(true).build());

  try (
    // Open a resource manager.
    final XdmResourceManager manager = database.getResourceManager("resource");
    // Open only write transaction on the resource (transaction provides a cursor for navigation
    // through moveToX-methods).
    final XdmNodeTrx wtx = manager.beginNodeTrx()) {
      // Import an XML document.
      wtx.insertSubtree(XmlShredder.createFileReader(xml), Insert.AS_FIRST_CHILD);

      // Commit and persist the changes.
      wtx.commit();

      // Transaction handle can be reused.
      wtx.moveToDocumentRoot();

      // Executes a modification visitor for each descendant node.
      for (final long nodeKey :
        DescendantAxis.builder(wtx).includeSelf().visitor(
          Optional.of(new ModificationVisitor(wtx, wtx.getNodeKey()))).build()) {
        ...
      }

      // Commit second version.
      wtx.commit();

      // Transaction handle is relocated at the document node of the new revision; iterate over "normal" descendant axis.
      final Axis axis = new DescendantAxis(wtx);
      if (axis.hasNext()) {
        axis.next();

        switch (wtx.getKind()) {
        case ELEMENT:
          // Do something.
          break;
        default:
          // Do nothing.
      }
      if (wtx.moveTo(axis.peek()).get().isComment()) {
        LOGGER.info(wtx.getValue());
      }

      // Begin a reading transaction on revision 0 concurrently to the write-transaction on revision 1 (the very first commited revision).
      try (final var rtx = resource.beginNodeReadOnlyTrx(0);) {
        // moveToX-methods returns either Moved or NotMoved whereas you can query if it has been moved or not, for instance via
        rtx.moveToFirstChild();

        if (rtx.moveToFirstChild().hasMoved())
          // Do something.

        // A fluent call would be if you know a node has a right sibling and there's a first child of the right sibling.
        rtx.moveToRightSibling().get().moveToFirstChild().get();

        // Can be tested before.
        if (rtx.hasRightSibling()) {
          rtx.moveToRightSibling();
        }

        // Move to next node in the XPath following::-axis.
        rtx.moveToNextFollowing();

        // Move to previous node in preorder.
        rtx.moveToPrevious();

        // Move to next node in preorder.
        rtx.moveToNext();

        /* 
         * Or simply within the move-operation and a postcondition check. If hasMoved() returns false, the transaction isn't moved.
         */
        if (rtx.moveToRightSibling().hasMoved()) {
          // Do something.
        }

      // Instead of the following, a visitor is useable!
      switch (rtx.getKind()) {
      case ELEMENT:
        for (int i = 0, nspCount = rtx.getNamespaceCount(); i < nspCount; i++) {
          rtx.moveToNamespace(i);
          LOGGER.info(rtx.getName());
          rtx.moveToParent();
        }

        for (int i = 0, attrCount = rtx.getAttributeCount(); i < attrCount; i++) {
          rtx.moveToAttribute(i);
          LOGGER.info(rtx.getName());
          rtx.moveToParent();
        }

        // Move to the specified attribute by name.
        rtx.moveToAttributeByName(new QNm("foobar"));
        rtx.moveToParent();
        
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
      case TEXT:
        LOGGER.info(rtx.getValue());
        break;
      case COMMENT:
        LOGGER.info(rtx.getValue());
        break;
      default:
        throw new IllegalStateException("Node kind not known!");
    }
  }
```
For printing the whole XML document:
```java
final var serializer = XmlSerializer.newBuilder(manager, System.out).prettyPrint().build();
serializer.call();
```
Or write it to string:
```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();
PrintStream writer = new PrintStream(baos);
final XmlSerializer serializer = XmlSerializer.newBuilder(manager, writer).prettyPrint().build();
serializer.call();
String content = baos.toString(StandardCharsets.UTF8);
```

Note that we aim to support all the Guava flavor. Just imagine how nice the following is:
```java
final Iterator<Long> results = FluentIterable.from(new DescendantAxis(rtx)).filter(new ElementFilter(rtx)).limit(2).iterator();
```

to filter all element nodes and skip after the first 2 elements are found. The resulting iterator contains at most 2 resulting unique node-keys (IDs) to which we can navigate through rtx.moveTo(long).

Furthermore a FilterAxis(Axis, Filter, Filter...) is usable. It's first parameter is the axis to use, the second parameter is a filter. Optionally further filters are usable (third varargs parameter).

To update a resource with algorithmically found differences between two tree-structures, use something like the following:

```java
// Old Sirix resource to update.
final var resOldRev = Paths.get(args[0]);

// XML document which should be imported as the new revision.
final var resNewRev = Paths.get(args[1]);

// Determine and import differences between the sirix resource and the
// provided XML document.
FMSEImport.xdmDataImport(resOldRev, resNewRev);
```

Method chaining for insertions:

```java
// Setup everything omitted... write transaction opened. Assertion: wtx is located at element node.
wtx.insertAttribute(new QNm("foo"), "bar", Move.PARENT).insertElementAsRightSibling(new QNm("baz"));

// Copy subtree of the node the read-transaction is located at as a new right sibling.
wtx.copySubtreeAsRightSibling(rtx);
```

Similarly moveToX()-methods are usable:

```java
// Get returns the transaction cursor currently used. However in this case the caller must be sure that a right sibling of the node denoted by node-key 15 and his right sibling and the right sibling's first child exists.
wtx.moveTo(15).getNodeCursor().moveToRightSibling().getNodeCursor().moveToFirstChild().getNodeCursor().insertCommentAsFirstChild("foo");
```

A whole bunch of axis are usable (all XPath axis and a few more):

```java
// Simple postorder-axis which iterates in postorder through a (sub)tree.
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

Furthermore a special filter-axis is provided:

``` 
// Filter by name (first argument is the axis, next arguments are filters (which implement org.sirix.axis.filter.Filter).
for (final var axis = new FilterAxis<XdmNodeReadOnlyTrx>(new VisitorDescendantAxis.Builder(rtx).includeSelf().visitor(Optional.of(visitor)).build(), new NameFilter(rtx, "foobar")); axis.hasNext();) {
  axis.next();
}
```

Further filters can be specified. All XPath axis are also available, plus a LevelOrderAxis, a ConcurrentAxis which executes the specified axis concurrently. Furthermore a ConcurrentUnionAxis, ConcurrentExceptAxis, ConcurrentIntersectAxis are provided. To allow chained axis, a `NestedAxis` is available which takes two axis as arguments.

The VisitorDescendantAxis above is especially useful as it executes a visitor as one of the first things in the hasNext() iterator-method. The return value of the visitor is used to guide the preorder traversal:

```java
/**
 * The result type of a {@link Visitor} implementation.
 * 
 * @author Johannes Lichtenberger, University of Konstanz
 */
public enum VisitResult {
  /** Continue without visiting the siblings of this structural node. */
  SKIPSIBLINGS,

  /** Continue without visiting the descendants of this element. */
  SKIPSUBTREE,

  /** Continue traversal. */
  CONTINUE,

  /** Terminate traversal. */
  TERMINATE,
}
```

Temporal axis to navigate not only in space, but also in time are also available (for instance to iterate over all future revisions, all past revisions, the last revision, the first revision, a specific revision, the previous revision, the next revision...)

