---
title: "树结构如何简化 OpenNote 的数据模型"
date: "2026-09-10 00:00:00 +0800"
lang: zh-CN
permalink: /cn/articles/posts/opennote-block-design.html
translation_key: opennote-block-design
description: "用统一的 Block 模型和父子关系，替代 OpenNote 中固定的集合、笔记本、笔记三层结构。"
---

最初设计 OpenNote 时，我采用了一种熟悉的层级结构：集合包含笔记本，笔记本包含笔记。当时，我没有仔细想过这些区分会给数据模型带来什么影响。

![集合包含笔记本 A 和笔记本 B；笔记本 A 包含笔记 1、笔记 2，笔记本 B 包含笔记 3。]({{ '/assets/images/opennote-hierarchy-cn.svg' | relative_url }})

这个设计把层级固定为三层。笔记不能包含另一篇笔记；如果要支持子笔记本，就得修改模型，或者增加例外规则。同时，尽管集合、笔记本和笔记很相似，我仍然需要维护三种组织概念。

阅读思源笔记（SiYuan）的代码后，我开始重新思考这个模型的出发点。从本质上说，笔记是一个独立的容器，用来存放文本，也可以容纳其他媒体。笔记应用管理的是这些容器，以及它们之间的关系。

对 OpenNote 来说，集合或笔记本也可以是一篇笔记。例如，一篇「数学」笔记可以包含概述，同时作为更具体的笔记的父节点。称它为笔记本，描述的是它在组织内容时承担的角色；而表达这个角色，只需要一个父子关系，不必为它定义单独的结构类型。

这就是树结构适合这个场景的原因：同一种容器可以出现在任意层级。

思源提供了一个具体的例子，展示如何同时表示标识和关系。它在 SQL 层的 `Block` 记录中包含以下字段。[1][ref1]

```go
type Block struct {
    ID       string
    ParentID string
    RootID   string
    // 其余字段省略。
}
```

每个块都有自己的标识、父节点 ID 和文档根节点 ID。这条记录还通过 `Box` 保留所属笔记本，通过 `Type` 标记内容类型。我的收获是：在一个共用模型中，可以分别表达标识、位置和内容角色。[1][ref1]

`NewTree()` 函数展示了这种关系如何建立。它使用同一种 `ast.Node` 类型创建文档根节点和段落，赋予它们不同的 `Type` 值，再通过 `ret.Root.AppendChild(newPara)` 将段落挂到根节点下。我把这种共用节点的思路用到了 OpenNote 的组织层级中；思源的完整模型仍然保留着自己的笔记本和块类型区分。[1][ref1] [2][ref2]

在树中，每个元素都是一个节点。根节点没有父节点，其他每个节点都有且只有一个父节点。一个节点可以有多个子节点，没有子节点的节点称为叶节点。父子关系不能形成环。

OpenNote 将这种通用容器称为 `Block`。下面这个示意层级中的每个方框都是一个块，箭头由父节点指向子节点。[3][ref3]

![学习是根节点；数学下有微积分，微积分下有积分和导数；系统下有内存管理。]({{ '/assets/images/opennote-tree-cn.svg' | relative_url }})

「学习」是根节点。「数学」既是子节点，也是父节点，而且可以拥有自己的内容。「积分」是第四层的叶节点，但以后也可以有子节点。叶节点描述的是它当前的位置，而不是一种永久的类型限制。

我最初的层级结构其实已经是一棵受限的树。重新设计后，我去掉了固定角色和三层的上限。多个顶层块组成多棵树的集合，术语上称为「森林」（forest）。

下面是 OpenNote 的 Rust 结构体，省略了派生属性和大部分注释。[3][ref3]

```rust
pub struct Block {
    pub id: Uuid,
    pub parent_id: Option<Uuid>,
    pub is_deleted: bool, // 为软删除预留。
    pub payloads: Vec<Payload>,
}
```

`id` 标识当前块。`parent_id` 标识它的父节点：`None` 表示根节点，`Some(id)` 则指向另一个块。`payloads` 存放内容。这个定义没有为集合、笔记本和笔记分配不同的类型。[3][ref3]

注意，这里没有 `children: Vec<Block>` 字段。树不一定要存成嵌套对象。OpenNote 存储父节点 ID，这种表示方式通常称为邻接表（adjacency list）。查找直接子节点，就是查找父节点 ID 匹配的块。下面的 SQL 用来说明代码中的父节点筛选条件，并非应用查询的原文。[4][ref4]

```sql
SELECT *
FROM blocks
WHERE parent_id = :block_id;
```

OpenNote 通过 ORM 表达这个筛选条件。对于 `ChildrenOf`，它会逐层重复查找，收集全部后代节点。熟悉的树遍历，就这样成为获取笔记数据某个分支的方法。[4][ref4]

遍历也可以反过来进行。`read_block_path()` 沿着父节点 ID 向上查找，再将收集到的块反转，得到从根节点到所选块的路径。[5][ref5]

块内部的载荷（payload）与它的子块是两回事。载荷存储内容及其向量表示，子块则建立组织关系。这样，OpenNote 就能把笔记的层级结构与为检索准备的内容片段分开。[6][ref6]

模型变小了，规则仍然不可少：父节点 ID 必须有效，移动节点不能产生环，删除节点时也需要明确如何处理后代节点。所查看代码中的更新校验会拒绝将块本身设为父节点，但仅靠这项检查，还不足以阻止更长的环。[7][ref7]

最终，这个模型在减少结构概念的同时，也带来了更大的灵活性。增加一层不再需要发明一种新类型，核心操作可以统一面向块，遍历也可以在整个层级中沿用同一种父子关系。[8][ref8] [4][ref4] [5][ref5] 对我而言，这让树成为一个实用的设计工具：用一小组规则，表达我希望用户组织笔记的方式。

**参考资料**

1. [思源的 Block 记录][ref1]。
2. [思源的树构建过程][ref2]。
3. [OpenNote 的 Block 定义][ref3]。
4. [后代节点查找][ref4]。
5. [父节点路径遍历][ref5]。
6. [Payload 定义][ref6]。
7. [Block 校验][ref7]。
8. [OpenNote 的核心块操作][ref8]。

[ref1]: https://github.com/siyuan-note/siyuan/blob/8641553a1f07374001902d3ce773285db1292b2d/kernel/sql/block.go#L39-L62
[ref2]: https://github.com/siyuan-note/siyuan/blob/8641553a1f07374001902d3ce773285db1292b2d/kernel/treenode/tree.go#L70-L83
[ref3]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-models/src/block.rs#L9-L20
[ref4]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-data/src/database/sqlite.rs#L271-L318
[ref5]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-data/src/database/sqlite.rs#L225-L247
[ref6]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-models/src/payload.rs#L12-L31
[ref7]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-data/src/database/sqlite.rs#L80-L101
[ref8]: https://github.com/opennote-org/opennote/blob/254fd32510b214539ecc500192f549c1fcf2576b/crates/opennote-core-logics/src/block.rs#L14-L107
