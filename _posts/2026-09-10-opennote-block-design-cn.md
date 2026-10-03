---
title: "一切皆为块：简化笔记应用的数据模型"
date: "2026-09-10 00:00:00 +0800"
lang: zh-CN
permalink: /cn/articles/posts/opennote-block-design.html
translation_key: opennote-block-design
description: "用统一的 Block 模型和父子关系，替代 OpenNote 中固定的集合、笔记本、笔记三层结构。"
---

最初设计 [OpenNote](https://github.com/opennote-org/opennote) 时，我想当然地把 Collection （集合）包含 Notebook （笔记本），Notebook 包含 Note （笔记）。只是做一种基于表象的抽象。得，这不就等于没抽象嘛！

![集合包含笔记本 A 和笔记本 B；笔记本 A 包含笔记 1、笔记 2，笔记本 B 包含笔记 3。]({{ '/assets/images/opennote-hierarchy-cn.svg' | relative_url }})

这个设计把层级固定为三层。笔记不能包含另一篇笔记；如果要支持子笔记本，就得修改模型，或者增加例外规则。同时，尽管集合、笔记本和笔记很相似，我仍然需要维护三种组织概念。

自己拉的屎当然要自己吃。在使用时，我自然会需要在笔记层级下面增加新的子项目，或者压根用不到3级的归类，只要创建一个笔记就够了。这就产生了我原本设计和使用需求之间的矛盾。因此，我开始借鉴成熟笔记应用的设计模式。当然，商业笔记软件没有开源代码。我就参考了思源笔记（SiYuan）。

“集合包括笔记本、笔记本包括笔记” 这样的设计本质上表达的就是一种固定的层级关系。也就是说，集合里面肯定包括笔记本，笔记本里面肯定包括笔记。当然，现实世界里面的层级关系是**不固定**的。我的笔记可以叫做“学习资料“，里面可以放”电影观后感“和”微积分“，随着时间的推移，假设我建立一个新的笔记本叫”数学一“，那么我就需要把”微积分“给移动到”数学一“里面，甚至还需要在”微积分“下面添加”微分“和”积分“，而”微分“下面还可以添加各个章节。这样，原来的三级分类就不起作用了，无法有效捕捉这一特征。而这经典的问题其实在数据结构里面就有解法，那就是树。

下面 Siyuan 笔记的代码展示如何同时表示标识和关系。它在 SQL 层的 `Block` 记录中包含以下字段。[1][ref1]

```go
type Block struct {
    ID       string
    ParentID string
    RootID   string
    // 其余字段省略。
}
```

每个块都有自己的标识、父节点 ID 和文档根节点 ID。这条记录还通过 `Box` 保留所属笔记本，通过 `Type` 标记内容类型。也就是说，在一个共用模型中，可以分别表达标识、位置和内容角色。[1][ref1]

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
