---
title: "How a Tree Simplified OpenNote's Data Model"
date: "2026-09-10 00:00:00 +0800"
lang: en
permalink: /articles/posts/opennote-block-design.html
translation_key: opennote-block-design
description: "How a shared Block model and parent relationships replaced OpenNote's fixed collection–notebook–note hierarchy."
---

When I first designed OpenNote, I started with a familiar hierarchy: collections contained notebooks, and notebooks contained notes. I did not think carefully enough about what those distinctions would mean for the data model.

![A collection contains Notebook A and Notebook B. Notebook A contains Note 1 and Note 2; Notebook B contains Note 3.]({{ '/assets/images/opennote-hierarchy-en.svg' | relative_url }})

The design imposed three levels. A note could not contain another note, and supporting a sub-notebook would require changing the model or adding an exception. I also had three organizational concepts to maintain, despite their similarities.

Reading SiYuan's code helped me reconsider the underlying idea. At its core, a note is an individual container for text and potentially other media. A notebook application manages those containers and their relationships.

For OpenNote, a collection or notebook could be a note as well. A Mathematics note could hold an overview and act as the parent of more specific notes. Calling it a notebook describes an organizational role; expressing that role only requires a parent relationship. I did not need a separate structural type for it.

That is why a tree fit this case: the same kind of container could appear at every level.

SiYuan provides a concrete example of representing identity and relationships together. Its SQL-layer `Block` record contains these fields, among others. [1][ref1]

```go
type Block struct {
    ID       string
    ParentID string
    RootID   string
    // Other fields omitted.
}
```

The block has its own identity, a parent ID, and a document-root ID. The record also retains notebook membership in `Box` and content classification in `Type`. My takeaway was that identity, position, and content role could be represented separately within a shared model. [1][ref1]

Its `NewTree()` function shows the relationship in action. It creates a document root and a paragraph using the same `ast.Node` type, gives them different `Type` values, and attaches the paragraph with `ret.Root.AppendChild(newPara)`. I adapted this shared-node approach to OpenNote's organizational hierarchy; SiYuan's complete model still has its own notebook and block-type distinctions. [1][ref1] [2][ref2]

In a tree, each item is a node. A root has no parent; every other node has one parent. A node can have several children, and a node with none is called a leaf. Parent–child relationships must not form cycles.

OpenNote calls its common container `Block`. Every box in this illustrative hierarchy is a block, and each arrow points from parent to child. [3][ref3]

![Learning is the root. Mathematics contains Calculus, whose children are Integration and Derivatives. Systems contains Memory management.]({{ '/assets/images/opennote-tree-en.svg' | relative_url }})

Learning is the root. Mathematics is both a child and a parent, and it can carry its own content. Integration is a leaf at the fourth level, but it could gain children later. Being a leaf describes its current position, not a permanent type restriction.

My original hierarchy was already a restricted tree. The redesign removed its fixed roles and three-level ceiling. Multiple top-level blocks form a collection of trees, technically called a *forest*.

Here is OpenNote's Rust structure, with derives and most comments omitted. [3][ref3]

```rust
pub struct Block {
    pub id: Uuid,
    pub parent_id: Option<Uuid>,
    pub is_deleted: bool, // Reserved for soft deletion.
    pub payloads: Vec<Payload>,
}
```

`id` identifies the block. `parent_id` identifies its parent: `None` represents a root, while `Some(id)` refers to another block. `payloads` holds the content. The definition does not assign different types to collections, notebooks, and notes. [3][ref3]

Notice that there is no `children: Vec<Block>` field. A tree does not have to be stored as nested objects. OpenNote stores parent IDs, a representation commonly called an adjacency list. Finding immediate children means looking for blocks with a matching parent ID. The following SQL illustrates the parent filter used in the code; it is not a verbatim application query. [4][ref4]

```sql
SELECT *
FROM blocks
WHERE parent_id = :block_id;
```

OpenNote expresses that filter through its ORM. For `ChildrenOf`, it then repeats the lookup for each successive level, gathering all descendants. A familiar tree traversal becomes a way to retrieve a branch of notebook data. [4][ref4]

Traversal works in the other direction too. `read_block_path()` follows parent IDs upward, then reverses the collected blocks to produce a path from the root to the selected block. [5][ref5]

The payloads inside a block are separate from its child blocks. A payload stores content and its vector representation; a child block establishes an organizational relationship. This lets OpenNote keep the notebook hierarchy separate from the pieces of content prepared for retrieval. [6][ref6]

The smaller model still needs rules. Parent IDs must be valid, moves must not create cycles, and deletion needs a policy for descendants. The reviewed update validation rejects a block being its own parent; that check alone does not prevent longer cycles. [7][ref7]

The outcome was greater flexibility with fewer structural concepts to maintain. Adding another level no longer required inventing another type. Core operations could work on blocks, and traversal could follow the same parent relationship throughout the hierarchy. [8][ref8] [4][ref4] [5][ref5] For me, this made trees a practical design tool: a small set of rules that matched how I wanted people to organize their notes.

**References**

1. [SiYuan's Block record][ref1].
2. [SiYuan's tree construction][ref2].
3. [OpenNote's Block definition][ref3].
4. [Descendant lookup][ref4].
5. [Parent-path traversal][ref5].
6. [Payload definition][ref6].
7. [Block validation][ref7].
8. [OpenNote core block operations][ref8].

[ref1]: https://github.com/siyuan-note/siyuan/blob/8641553a1f07374001902d3ce773285db1292b2d/kernel/sql/block.go#L39-L62
[ref2]: https://github.com/siyuan-note/siyuan/blob/8641553a1f07374001902d3ce773285db1292b2d/kernel/treenode/tree.go#L70-L83
[ref3]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-models/src/block.rs#L9-L20
[ref4]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-data/src/database/sqlite.rs#L271-L318
[ref5]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-data/src/database/sqlite.rs#L225-L247
[ref6]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-models/src/payload.rs#L12-L31
[ref7]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-data/src/database/sqlite.rs#L80-L101
[ref8]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-core-logics/src/block.rs#L14-L107

