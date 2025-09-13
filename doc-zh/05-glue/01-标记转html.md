````markdown
# 将标记转换为 HTML

在我们将所有东西粘合在一起之前，还缺少一个关键部分，那就是将我们的 `Markup` 数据类型转换为 `Html`。

我们将从创建一个新模块并导入 `Markup` 和 `Html` 模块开始。

```hs
module Convert where

import qualified Markup
import qualified Html
```

## 限定导入

这次，我们限定地导入了模块。限定导入意味着我们没有将导入模块中定义的名称暴露给通用模块命名空间，而是必须用模块名称作为前缀。

例如，`parse` 变成了 `Markup.parse`。如果我们限定地导入了 `Html.Internal`，我们就必须写 `Html.Internal.el`，这有点长。

我们也可以使用 `as` 关键字给模块起一个新名字：

```hs
import qualified Html.Internal as HI
```

然后写 `HI.el`。

我喜欢使用限定导入，因为读者不必猜测一个名字来自哪里。有些模块甚至被设计成限定导入。例如，许多容器类型（如映射、集合和向量）的 API 非常相似。如果想在单个模块中使用多个容器，我们几乎必须使用限定导入，这样当我们编写一个像 `singleton` 这样的函数（它创建一个包含单个值的容器）时，GHC 就会知道我们指的是哪个 `singleton` 函数。

有些人更喜欢使用导入列表而不是限定导入，因为限定名称可能有点冗长和嘈杂。我通常更喜欢限定导入而不是导入列表，但你可以随意尝试两种解决方案，看看哪种更适合你。有关导入的更多信息，请参阅这篇 [wiki 文章](https://wiki.haskell.org/Import)。

## 将 `Markup.Structure` 转换为 `Html.Structure`

在这一点上，将标记结构转换为 HTML 结构大多是直截了当的，我们需要对标记结构进行模式匹配，并使用相关的 HTML API。

```hs
convertStructure :: Markup.Structure -> Html.Structure
convertStructure structure =
  case structure of
    Markup.Heading 1 txt ->
      Html.h1_ txt

    Markup.Paragraph p ->
      Html.p_ p

    Markup.UnorderedList list ->
      Html.ul_ $ map Html.p_ list

    Markup.OrderedList list ->
      Html.ol_ $ map Html.p_ list

    Markup.CodeBlock list ->
      Html.code_ (unlines list)
```

请注意，使用 `-Wall` 运行此代码将揭示模式匹配是*非穷尽的*。这是因为我们目前没有办法构建非 `h1` 的标题。有几种方法可以处理这个问题：

-   忽略警告——这很可能有一天会在运行时失败，用户会很难过
-   对其他情况进行模式匹配，并使用 `error` 函数添加一个友好的错误——它具有与上面相同的缺点，但也不会在编译时通知未处理的情况
-   模式匹配并做错误的事情——用户仍然很难过
-   使用 `Either` 在类型系统中编码错误，我们将在后面的章节中看到如何做到这一点
-   限制输入——更改 `Markup.Heading` 以不包含数字，而是包含特定的受支持的标题。这是一个合理的方法
-   实现一个支持任意标题的 HTML 函数。应该很容易做到

> #### 这些 `$` 是什么？
>
> 美元符号 (`$`) 是一个我们可以用来对表达式进行分组的运算符，就像我们使用括号一样。我们可以用不可见的括号替换 `$`，将它左边的表达式和右边的表达式括起来。所以：
>
> ```hs
> Html.ul_ $ map Html.p_ list
> ```
>
> 被理解为：
>
> ```hs
> (Html.ul_) (map Html.p_ list)
> ```
>
> 它是一个函数应用运算符，它将美元右边的参数应用于美元左边的函数。
>
> `$` 是右结合的，并且优先级非常低，这意味着：它向右分组，其他运算符绑定得更紧密。例如，以下表达式：
>
> ```hs
> filter (2<) $ map abs $ [-1, -2, -3] <> [4, 5, 6]
> ```
>
> 被理解为：
>
> ```hs
> (filter (2<)) ((map abs) ([1, -2, 3] <> [-4, 5, 6]))
> ```
>
> 这也等同于以下代码，括号更少：
>
> ```hs
> filter (2<) (map abs ([1, -2, 3] <> [-4, 5, 6]))
> ```
>
> 看到信息如何从右向左流动，以及 `<>` 绑定得更紧密了吗？
>
> 这个运算符在 Haskell 代码中相当普遍，它帮助我们减少一些混乱，但如果你愿意，可以随意避免使用它，转而使用括号，我们甚至没有用 `$` 节省按键次数！

---

练习：实现 `h_ :: Natural -> String -> Structure`，我们将用它来定义任意标题（例如 `<h1>`、`<h2>` 等）。

<details><summary>解决方案</summary>

```hs
import Numeric.Natural

h_ :: Natural -> String -> Structure
h_ n = Structure . el ("h" <> show n) . escape
```

别忘了从 `Html.hs` 中导出它！

</details>


练习：使用 `h_` 修复 `convertStructure`。


<details><summary>解决方案</summary>

```hs
convertStructure :: Markup.Structure -> Html.Structure
convertStructure structure =
  case structure of
    Markup.Heading n txt ->
      Html.h_ n txt

    Markup.Paragraph p ->
      Html.p_ p

    Markup.UnorderedList list ->
      Html.ul_ $ map Html.p_ list

    Markup.OrderedList list ->
      Html.ol_ $ map Html.p_ list

    Markup.CodeBlock list ->
      Html.code_ (unlines list)
```

</details>

---

## 文档 -> Html

要创建 `Html` 文档，我们需要使用 `html_` 函数。这个函数需要两样东西：一个 `Title` 和一个 `Structure`。

对于标题，我们可以直接从外部使用文件名提供。

要将我们的标记 `Document`（这是一个标记 `Structure` 的列表）转换为 HTML `Structure`，我们需要转换每个标记 `Structure`，然后将它们连接在一起。

我们已经知道如何转换每个标记 `Structure`；我们可以使用我们编写的 `convertStructure` 函数和 `map`。这将为我们提供以下函数：

```
map convertStructure :: Markup.Document -> [Html.Structure]
```

要连接所有的 `Html.Structure`，我们可以尝试编写一个递归函数。然而，我们会很快遇到一个基本情况的问题：当列表为空时该怎么办？

我们可以只提供一个表示空 HTML 结构的虚拟 `Html.Structure`。

让我们将这个添加到 `Html.Internal`：

```hs
empty_ :: Structure
empty_ = Structure ""
```

---

现在我们可以编写我们的递归函数了。试试吧！

<details><summary>解决方案</summary>

```hs
concatStructure :: [Structure] -> Structure
concatStructure list =
  case list of
    [] -> empty_
    x : xs -> x <> concatStructure xs
```

</details>

---

还记得我们作为 `Semigroup` 类型类实例实现的 `<>` 函数吗？我们提到 `Semigroup` 是实现 `<> :: a -> a -> a` 的事物的**抽象**，其中 `<>` 是关联的 (`a <> (b <> c) = (a <> b) <> c`)。

事实证明，拥有 `Semigroup` 的实例并且还拥有一个表示“空”值的值是一种相当普遍的模式。例如，一个字符串可以被连接，空字符串可以作为“空”值。这实际上是一个众所周知的**抽象**，称为**幺半群 (monoid)**。

## 幺半群 (Monoids)

实际上，“空”并不是对我们想要的东西的一个很好的描述，作为一种抽象也不是很有用。相反，我们可以将其描述为一个“单位”元素，它满足以下法则：

-   `x <> <identity> = x`
-   `<identity> <> x = x`

换句话说，如果我们尝试使用这个“空”——这个单位值，作为 `<>` 的一个参数，我们将总是得到另一个参数。

对于 `String`，空字符串 `""` 满足这个条件：

```hs
"" <> "world" = "world"
"hello" <> "" = "hello"
```

当然，这对于我们写的任何值都成立，而不仅仅是“world”和“hello”。

实际上，如果我们暂时离开 Haskell 的世界，即使是整数，以 `+` 作为关联二元运算 `+`（代替 `<>`）和 `0` 代替单位元，也构成一个幺半群：

```hs
17 + 0 = 17
0 + 99 = 99
```

所以整数和 `+` 运算一起构成一个半群，和 `0` 一起构成一个幺半群。

我们从中了解到新东西：

1.  幺半群是半群之上更具体的抽象；它通过添加一个新条件（单位元的存在）来构建
2.  这个抽象很有用！我们可以编写一个通用的 `concatStructure`，它可以适用于任何幺半群

事实上，`base` 中存在一个名为 `Monoid` 的类型类，它以 `Semigroup` 作为**超类**。

```hs
class Semigroup a => Monoid a where
  mempty :: a
```

> 注意：这是一个简化版本。
> [实际的](https://hackage.haskell.org/package/base-4.16.4.0/docs/Prelude.html#t:Monoid)
> 由于向后兼容性和性能原因，要复杂一些。
> `Semigroup` 实际上是在 `Monoid` 之后引入 Haskell 的！

我们可以为我们的 HTML `Structure` 数据类型添加一个 `Monoid` 实例：


```hs
instance Monoid Structure where
  mempty = empty_
```

现在，我们可以使用库函数，而不是使用我们自己的 `concatStructure`：

```hs
mconcat :: Monoid a => [a] -> a
```

理论上可以实现为：

```hs
mconcat :: Monoid a => [a] -> a
mconcat list =
  case list of
    [] -> mempty
    x : xs -> x <> mconcat xs
```

请注意，因为 `Semigroup` 是 `Monoid` 的*超类*，我们仍然可以使用 `Semigroup` 类中的 `<>` 函数，而无需在 `=>` 的左侧添加 `Semigroup a` 约束。通过添加 `Monoid a` 约束，我们也隐式地添加了 `Semigroup a` 约束！

这个 `mconcat` 函数与 `concatStructure` 函数非常相似，但这个函数适用于任何 `Monoid`，包括 `Structure`！抽象帮助我们识别通用模式并**重用**代码！

> 旁注：整数与 `+` 和 `0` 在 Haskell 中实际上不是 `Monoid` 的实例。这是因为整数也可以与 `*` 和 `1` 形成一个幺半群！但是**每个类型只能有一个实例**。相反，存在另外两个提供该功能的 `newtype`，[Sum](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Monoid.html#t:Sum) 和 [Product](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Monoid.html#t:Product)。看看它们如何在 `ghci` 中使用：
>
> ```hs
> ghci> import Data.Monoid
> ghci> Product 2 <> Product 3 -- 注意，Product 是一个数据构造函数
> Product {getProduct = 6}
> ghci> getProduct (Product 2 <> Product 3)
> 6
> ghci> getProduct $ mconcat $ map Product [1..5]
> 120
> ```

## 另一个抽象？

我们现在已经两次使用了 `map` 然后 `mconcat`。肯定有一个函数可以统一这种模式。事实上，它被称为 [`foldMap`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Foldable.html#v:foldMap)，它不仅适用于列表，还适用于任何可以“折叠”或“归约”为摘要值的数据结构。这种抽象和类型类被称为 **Foldable**。

为了更简单地理解 `Foldable`，我们可以看看 `fold`：

```hs
fold :: (Foldable t, Monoid m) => t m -> m

-- 比较
mconcat :: Monoid m            => [m] -> m
```

`mconcat` 只是 `fold` 针对列表的专门版本。而 `fold` 可以用于任何实现了 `Foldable` 的数据结构和实现了 `Monoid` 的有效载荷类型的组合。这可以是 `[]` 和 `Structure`，或者 `Maybe` 和 `Product Int`，或者你闪亮的新二叉树和 `String` 作为有效载荷类型。但请注意，`Foldable` 类型的*种类*必须是 `* -> *`。所以，例如 `Html` 不能是 `Foldable`。

`foldMap` 是一个函数，它允许我们在使用 `<>` 函数组合它们之前，将一个函数应用于 `Foldable` 类型的有效载荷类型。

```hs
foldMap :: (Foldable t, Monoid m) => (a -> m) -> t a -> m

-- 与一个专门版本比较：
-- - t ~ []
-- - m ~ Html.Structure
-- - a ~ Markup.Structure
foldMap
  :: (Markup.Structure -> Html.Structure)
  -> [Markup.Structure]
  -> Html.Structure
```

顾名思义，它确实在“折叠”之前“映射”。你可能会在这里停下来想，“我们正在谈论的这个‘map’并不特定于列表；也许那是另一个抽象？”是的。它实际上是一个非常重要和基础的抽象，称为 `Functor`。但我认为我们这一章的抽象已经足够了。我们将在后面的章节中介绍它！

## 完成我们的转换模块

让我们通过编写 `convert` 来完成我们的代码：

```hs
convert :: Html.Title -> Markup.Document -> Html.Html
convert title = Html.html_ title . foldMap convertStructure
```

现在我们有了一个完整的实现，可以将标记文档转换为 HTML：

```hs
-- Convert.hs
module Convert where

import qualified Markup
import qualified Html

convert :: Html.Title -> Markup.Document -> Html.Html
convert title = Html.html_ title . foldMap convertStructure

convertStructure :: Markup.Structure -> Html.Structure
convertStructure structure =
  case structure of
    Markup.Heading n txt ->
      Html.h_ n txt

    Markup.Paragraph p ->
      Html.p_ p

    Markup.UnorderedList list ->
      Html.ul_ $ map Html.p_ list

    Markup.OrderedList list ->
      Html.ol_ $ map Html.p_ list

    Markup.CodeBlock list ->
      Html.code_ (unlines list)
```

## 总结

我们学习了：

-   限定导入
-   处理错误的方法
-   `Monoid` 类型类和抽象
-   `Foldable` 类型类和抽象

接下来，我们将把我们的功能粘合在一起，并学习 Haskell 中的 I/O！

> 你可以查看我们所做更改的 git 提交 [the changes we've made](https://github.com/soupi/learn-haskell-blog-generator/commit/ad34f2264e9114f2d7436ff472c78da47055fcfe) 和 [到目前为止的代码](https://github.com/soupi/learn-haskell-blog-generator/tree/ad34f2264e9114f2d7436ff472c78da47055fcfe)。
````