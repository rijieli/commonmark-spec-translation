# 附录：一种语法分析策略

在本附录中，我们描述了 CommonMark 参考实现中使用的解析策略的一些特性。 

## 概述

解析有两个阶段： 

1. 在第一阶段，输入的行被解析并且文档的块结构---它被划分为段落、块引用、列表项等等被构建。文本被分配给这些块但未被解析。链接引用定义会被解析并构建一个链接映射表。

2. 在第二阶段，段落原始文本内容和标题等被解析为 Markdown 内联元素序列（字符串、行内代码、链接、强调等），第一阶段构建的链接映射表参与这个解析过程。

处理中的每个点，文档都表示为 **blocks** 树。树的根节点是一个`document`块。`document`可以有任意数量的其他块作为 **子节点**。反过来，这些子节点也可以将其他块作为子节点。一个块最后的子节点通常被认为是 **open**，这意味着后续的输入行可以改变其内容。 （未打开的块是**closed**。）举例而言，下方是一棵可能的文档树，其中打开的块用箭头标记：

``` tree
-> document
  -> block_quote
       paragraph
         "Lorem ipsum dolor\nsit amet."
    -> list (type=bullet tight=true bullet_char=-)
         list_item
           paragraph
             "Qui *quodsi iracundia*"
      -> list_item
        -> paragraph
             "aliquando id"
```

## 阶段 1：块结构 

每一行的解析都有可能对这棵树产生影响。输入的行会被分析，并且根据其内容，可以通过以下一种或多种方式改动根节点： 

1. 一个或多个打开的块被关闭。
2. 可能创建一个或多个新块作为最后一个打开块的子代。 
3. 文本会被添加到树上剩余的最后（深）的一个开放块。

当一行被以这种方式合并到树中，它可以被丢弃，因此可以在流中读取输入。

对于每一行，我们遵循以下过程： 

1. 首先，我们遍历打开的块，从根节点`document`开始，向下遍历最后一个子节点，直到最后一个打开的块。一个块如果要保持打开状态则必须满足一定条件。例如，代表引用的块需要一个`>`字符。一个段落需要至少一个非的空行。在这个阶段，我们可能会匹配所有或部分开放块。但是不能关闭未匹配的块，因为我们可能有一个惰性延续行[[lazy continuation line](https://spec.commonmark.org/0.29/#lazy-continuation-line)]。

2. 接下来，在使用%现有块的继续标记%之后，我们寻找开始标记（例如，代表引用的`>`）。如果我们遇到一个新的开始标记，我们会在%创建新块作为最后一个匹配块的子块之前关闭步骤 1 中未匹配的所有块%。

3. 最后，我们查看该行的其余部分（在块标记如 `>`、列表标记和缩进被消耗之后）。这些剩余的文本可以合并到最后一个打开的块（段落、代码块、标题或原始 HTML）中。

当我们看到段落中的一行是 [setext heading underline] 时，构造一个 Setext 标题块。

段落关闭时会检测参考链接定义；分析累积的行以检查它们是否以一个或多个参考链接定义标记开始。任何剩余部分都成为正常段落。

通过四行 Markdown 文本，我们来了解上面的树是如何生成的：

``` markdown
> Lorem ipsum dolor
sit amet.
> - Qui *quodsi iracundia*
> - aliquando id
```

一开始，我们的文档模型只包含根节点

``` tree
-> document
```

我们文字的第一行

``` markdown
> Lorem ipsum dolor 
```

将创建一个 `block_quote` 节点作为 `document` 的子节点，同时还有一个 `paragraph` 节点作为 `block_quote` 的子节点。剩余的文本被添加到最后一个打开的块节点 `paragraph` 中

``` tree
-> document
  -> block_quote
    -> paragraph
         "Lorem ipsum dolor"
```

下一行，

``` markdown
sit amet.
```

是打开的块节点 `paragraph` 的“惰性延续”，因此它也会被添加到`paragraph`节点的文本中：

``` tree
-> document
  -> block_quote
    -> paragraph
         "Lorem ipsum dolor\nsit amet."
```

第三行， 

``` markdown
> - Qui *quodsi iracundia*
```

导致 `paragraph` 块被关闭，同时还将创建一个新的 `list` 节点作为 `block_quote` 的子节点。一个新的 `list_item` 也被添加为 `list` 的子节点，一个 `paragraph` 节点作为 `list_item` 的子节点被创建。剩余的文本将被添加到这个 `paragraph` 中：

``` tree
-> document
  -> block_quote
       paragraph
         "Lorem ipsum dolor\nsit amet."
    -> list (type=bullet tight=true bullet_char=-)
      -> list_item
        -> paragraph
             "Qui *quodsi iracundia*"
```

第四行

``` markdown
> - aliquando id
```

导致 `list_item`（及其子节点 `paragraph`）被关闭，一个新的 `list_item` 作为 `list` 的子节点被打开。一个 `paragraph` 被添加为新的 `list_item` 的子节点用来保存本行剩余的文本。

我们因此获得了最终的语法树：

``` tree
-> document
  -> block_quote
       paragraph
         "Lorem ipsum dolor\nsit amet."
    -> list (type=bullet tight=true bullet_char=-)
         list_item
           paragraph
             "Qui *quodsi iracundia*"
      -> list_item
        -> paragraph
             "aliquando id"
```

## 阶段 2: 内联结构

当所有的输入都被解析，所有打开的块都被关闭。

我们将“遍历树”，访问每个节点，并将段落和标题的原始文本内容解析为内联节点。至此，我们已经获取了所有的链接引用定义，我们可以随时解析引用链接。

``` tree
document
  block_quote
    paragraph
      str "Lorem ipsum dolor"
      softbreak
      str "sit amet."
    list (type=bullet tight=true bullet_char=-)
      list_item
        paragraph
          str "Qui "
          emph
            str "quodsi iracundia"
      list_item
        paragraph
          str "aliquando id"
```

注意第一个`paragraph`中的换行符被解析为 `softbreak`，第一个列表项中的星号变成了一个 `emph` 节点。 

### 解析嵌套强调和链接的算法

到目前为止，内联解析最棘手的部分是处理强调、加粗、链接和图像。下面的算法将帮助我们实现这个过程。

当我们开始解析内联节点并且遇到

- 一连串 `*` 或 `_` 字符，或者
- 一个 `[` 或 `![` 时

我们插入一个文本节点，并将这些符号作为内容，然后向 delimiter stack [分隔符堆栈](https://spec.commonmark.org/0.29/#delimiter-stack)添加这个节点的指针。

[分隔符堆栈](https://spec.commonmark.org/0.29/#delimiter-stack)是一个双向链表。每个元素都包含一个指向文本节点的指针，以及

- 分隔符的类型（`[`、`![`、`*`、`_`）
- 分隔符的数量，
- 分隔符是否为 *active* 状态（所有的起始标记都是激活状态的），以及
- 分隔符是否为潜在的开启者、潜在的关闭者还是两者（这取决于在分隔符之前和之后的字符）。

当我们遇到 `]` 字符时，我们调用 *查找链接或图像* 过程（见下文）。

当我们到达输入的末尾时，我们调用 *处理强调* 过程（见下文），同时 `stack_bottom` = NULL。 

#### *查找链接或图像*

从分隔符堆栈的顶部开始，我们向下查看堆栈寻找一个开始的 `[` 或 `![` 分隔符。 

- 如果我们没有找到，我们返回一个文字文本节点`]`。

- 如果我们确实找到了一个，但它不是 *active*，我们从堆栈中删除不活动的分隔符，并返回一个文字文本节点 `]`。

- 如果我们找到一个并且它处于活动状态，那么我们会提前解析以查看是否有内嵌链接/图像、参考链接/图像、紧凑参考链接/图像或快捷方式参考链接/图像。

  + 如果我们不这样做，那么我们从分隔符堆栈中删除开始分隔符并返回一个文字文本节点 `]`。
  
  + 如果我们这样做，那么
  
    * 我们返回一个链接或图像节点，其子节点是内联在开始分隔符指向的文本节点之后。 
    
    * 我们在这些内联上运行 *处理强调* 过程，使用`[` 开启者作为 `stack_bottom`。 
    
    * 我们删除了开始分隔符。
    
    * 如果我们有一个链接（而不是图像），我们还将开始分隔符之前的所有 `[` 分隔符设置为 *inactive*。 （这将阻止我们获得链接内的链接。）
    
#### *处理强调*

参数 `stack_bottom` 设置了我们在 *分隔符堆栈* 中下降多远的下限。如果它是 NULL，我们可以一直走到底部。否则，我们会在访问 `stack_bottom` 之前停止。

让 `current_position` 指向 *分隔符堆栈* 上位于 `stack_bottom` 上方的元素（或者如果 `stack_bottom` 为 NULL，则指向第一个元素）。

我们跟踪每个分隔符类型（`*`、`_`）和每个结束分隔符运行的长度（模 3）的 `openers_bottom`。将其初始化为 `stack_bottom`。

然后我们重复以下操作，直到我们用完潜在的关闭者： 

- 移动 `current_position` 在分隔符堆栈中前进（如果需要），直到我们找到第一个潜在的更接近分隔符 `*` 或 `_` 的位置。 （这将是最接近输入开头的潜在可能性——解析顺序中的第一个。） 

- 现在，回顾堆栈（停留在此分隔符类型的 `stack_bottom` 和 `openers_bottom` 上方）中第一个匹配的潜在开启者（“匹配”意味着相同的分隔符）。 

- 如果找到一个：
  
  + 弄清楚我们是强调还是强调强调：如果更近和开场跨度的长度 >= 2，我们有强的，否则是规则的。
  
  + 相应地插入一个 emph 或 strong emph 节点，在对应于 opener 的文本节点之后。从分隔符堆栈中删除开启器和关闭器之间的所有分隔符。
  
  + 从开始和结束文本节点中删除 1（对于常规 emph）或 2（对于强 emph）分隔符。如果它们因此变为空，则删除它们并删除相应的元素 定界符堆栈的。如果关闭节点被移除，则将 `current_position` 重置为堆栈中的下一个元素。 

- 如果没有找到：
  
  + 将 `openers_bottom` 设置为 `current_position` 之前的元素。（我们知道这种接近 并包括这一点没有开启者，因此这为未来的搜索设置了下限。）
  
  + 如果“current_position”的接近者不是潜在的开启者，请将其从分隔符堆栈中删除（因为我们知道它也不能更接近）。将 `current_position` 推进到堆栈中的下一个元素。
  
完成后，我们从分隔符堆栈中删除 `stack_bottom` 上方的所有分隔符。 
