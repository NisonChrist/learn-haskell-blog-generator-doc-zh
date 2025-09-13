````markdown
# 使用 Either 处理错误

在 Haskell 中，有相当多的方法来指示和处理错误。
我们将看一种解决方案：使用
[Either](https://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Either.html) 类型。
Either 的定义如下：

```hs
data Either a b
  = Left a
  | Right b
```

简单地说，`Either a b` 类型的值可以包含 `a` 类型的值，
也可以包含 `b` 类型的值。
我们可以通过使用的构造函数来区分它们。

```hs
Left True :: Either Bool b
Right 'a' :: Either a Char
```

有了这个类型，我们可以使用
`Left` 构造函数来指示失败，并附带一些错误值，
而使用 `Right` 构造函数和一个类型来表示成功，并附带
预期的结果。

由于 `Either` 是多态的，我们可以使用任何两种类型来表示
失败和成功。使用 ADT 来描述失败模式通常很有用。

例如，假设我们想将一个 `Char` 解析为十进制数字的 `Int`。
如果该字符不是数字，此操作可能会失败。
我们可以将此错误表示为一个数据类型：

```hs
data ParseDigitError
  = NotADigit Char
  deriving Show
```

我们的解析函数可以具有以下类型：

```hs
parseDigit :: Char -> Either ParseDigitError Int
```

现在，当我们实现我们的解析函数时，我们可以在出现错误时返回 `Left`，
描述问题，并在成功解析时返回 `Right`，附带解析后的值：


```hs
parseDigit :: Char -> Either ParseDigitError Int
parseDigit c =
  case c of
    '0' -> Right 0
    '1' -> Right 1
    '2' -> Right 2
    '3' -> Right 3
    '4' -> Right 4
    '5' -> Right 5
    '6' -> Right 6
    '7' -> Right 7
    '8' -> Right 8
    '9' -> Right 9
    _ -> Left (NotADigit c)
```

`Either a` 也是 `Functor` 和 `Applicative` 的实例，
所以如果我们想组合这些
类型的计算，我们有一些组合子可以使用。

例如，如果我们有三个字符，并且我们想尝试解析
它们中的每一个，然后找出它们之间的最大值；我们可以使用
应用式接口：

```hs
max3chars :: Char -> Char -> Char -> Either ParseDigitError Int
max3chars x y z =
  (\a b c -> max a (max b c))
    <$> parseDigit x
    <*> parseDigit y
    <*> parseDigit z
```


`Either a` 的 `Functor` 和 `Applicative` 接口允许我们
将函数应用于有效负载值，并将错误处理**延迟**到
稍后的阶段。从语义上讲，按顺序返回 `Left` 的第一个 Either
将是返回值。我们可以在应用式实例的实现中看到这是如何工作的：

```hs
instance Applicative (Either e) where
    pure          = Right
    Left  e <*> _ = Left e
    Right f <*> r = fmap f r
```

在某个时候，有人会真正想要**检查**结果，
看看我们是得到一个错误（使用 `Left` 构造函数）还是预期的值
（使用 `Right` 构造函数），他们可以通过模式匹配结果来做到这一点。

## Applicative + Traversable

`Either` 的 `Applicative` 接口非常强大，可以与
另一个名为
[`Traversable`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Traversable.html#g:1) 的抽象结合使用 -
用于可以从左到右遍历的数据结构，如链表或二叉树。
有了这些，我们可以组合不确定数量的值，例如 `Either ParseDigitError Int`，
只要它们都在一个实现了 `Traversable` 的数据结构中。

让我们看一个例子：

```hs
ghci> :t "1234567"
"1234567" :: String
-- 记住，String 是 Char 列表的别名
ghci> :info String
type String :: *
type String = [Char]
      -- 在 ‘GHC.Base’ 中定义

ghci> :t map parseDigit "1234567"
map parseDigit "1234567" :: [Either ParseDigitError Int]
ghci> map parseDigit "1234_567"
[Right 1,Right 2,Right 3,Right 4,Right 5,Right 6,Right 7]

ghci> :t sequenceA
sequenceA :: (Traversable t, Applicative f) => t (f a) -> f (t a)
-- 将 `t` 替换为 `[]`，将 `f` 替换为 `Either Error` 以获得专门版本

ghci> sequenceA (map parseDigit "1234567")
Right [1,2,3,4,5,6,7]

ghci> map parseDigit "1a2"
[Right 1,Left (NotADigit 'a'),Right 2]
ghci> sequenceA (map parseDigit "1a2")
Left (NotADigit 'a')
```

`map` 然后 `sequenceA` 的模式是另一个名为 `traverse` 的函数：

```hs
ghci> :t traverse
traverse
  :: (Traversable t, Applicative f) => (a -> f b) -> t a -> f (t b)
ghci> traverse parseDigit "1234567"
Right [1,2,3,4,5,6,7]
ghci> traverse parseDigit "1a2"
Left (NotADigit 'a')
```

我们可以在任何两个类型上使用 `traverse`，其中一个实现了 `Applicative`
接口，如 `Either a` 或 `IO`，另一个实现了 `Traversable` 接口，
如 `[]`（链表）和
[`Map k`](https://hackage.haskell.org/package/containers-0.6.5.1/docs/Data-Map-Strict.html#t:Map)
（在其他语言中也称为字典 - 从键到值的映射）。
例如，使用 `IO` 和 `Map`。请注意，我们可以使用
[`fromList`](https://hackage.haskell.org/package/containers-0.6.5.1/docs/Data-Map-Strict.html#v:fromList)
函数从元组列表构造一个 `Map` 数据结构 - 元组中的第一个值是键，第二个是类型。

```hs
ghci> import qualified Data.Map as M -- 来自 containers 包

ghci> file1 = ("output/file1.html", "input/file1.txt")
ghci> file2 = ("output/file2.html", "input/file2.txt")
ghci> file3 = ("output/file3.html", "input/file3.txt")
ghci> files = M.fromList [file1, file2, file3]
ghci> :t files :: M.Map FilePath FilePath -- FilePath 是 String 的别名
files :: M.Map FilePath FilePath :: M.Map FilePath FilePath

ghci> readFiles = traverse readFile
ghci> :t readFiles
readFiles :: Traversable t => t FilePath -> IO (t String)

ghci> readFiles files
fromList [("output/file1.html","I'm the content of file1.txt\n"),("output/file2.html","I'm the content of file2.txt\n"),("output/file3.html","I'm the content of file3.txt\n")]
ghci> :t readFiles files
readFiles files :: IO (Map String String)
```

上面，我们创建了一个函数 `readFiles`，它将接受一个从*输出文件路径*
到*输入文件路径*的映射，并返回一个 IO 操作，当运行时，它将读取输入文件
并将其内容直接替换到映射中！这肯定在以后会很有用。

## 多个错误

注意，由于 `Either` 的种类是 `* -> * -> *`（它接受两个类型
参数），`Either` 不能是 `Functor` 或 `Applicative` 的实例：
这些类型类的实例必须具有
`* -> *` 的种类。
记住，当我们看一个类型类函数签名时，比如：

```hs
fmap :: Functor f => (a -> b) -> f a -> f b
```

如果我们想为特定类型（代替 `f`）实现它，
我们需要能够用目标类型*替换* `f`。如果我们尝试
用 `Either` 来做，我们会得到：

```hs
fmap :: (a -> b) -> Either a -> Either b
```

`Either a` 和 `Either b` 都不是*饱和的*，所以这不会进行类型检查。
出于同样的原因，如果我们尝试用，比如说，`Int` 来替换 `f`，我们会得到：

```hs
fmap :: (a -> b) -> Int a -> Int b
```

这也没有意义。

虽然我们不能使用 `Either`，但我们可以使用 `Either e`，它的种类是
`* -> *`。现在让我们尝试在这个签名中用 `Either e` 替换 `f`：

```hs
liftA2 :: Applicative => (a -> b -> c) -> f a -> f b -> f c
```

我们会得到：

```hs
liftA2 :: (a -> b -> c) -> Either e a -> Either e b -> Either e c
```

这告诉我们，我们只能使用应用式接口来
组合两个*具有相同 `Left` 构造函数类型的 `Either`*。

那么，如果我们有两个可以返回不同错误的函数，我们该怎么办？
有几种方法；最主要的是：

1. 让它们返回相同的错误类型。编写一个包含所有可能
   错误描述的 ADT。这在某些情况下可行，但并不总是理想的。
   例如，调用 `parseDigit` 的用户不应该被迫
   处理输入可能是空字符串的可能情况。
2. 为每种类型使用专门的错误类型，当它们组合在一起时，
   将每个函数的错误类型映射到一个更通用的错误类型。这可以
   通过 `Bifunctor` 类型类中的
   [`first`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Bifunctor.html#v:first)
   函数来完成。

## Monadic 接口

应用式接口允许我们将一个函数提升到多个
`Either` 值（或其他应用式函子实例，如 `IO` 和 `Parser`）上工作。
但更多时候，我们希望在一个可能返回错误的计算中使用
另一个可能返回错误的计算中的值。

例如，像 GHC 这样的编译器分阶段运行，如词法分析、
解析、类型检查等。每个阶段都依赖于
前一个阶段的输出，并且每个阶段都可能失败。我们可以为这些函数编写类型：

```hs
tokenize :: String -> Either Error [Token]

parse :: [Token] -> Either Error AST

typecheck :: AST -> Either Error TypedAST
```

我们希望组合这些函数，使它们以链式方式工作。`tokenize` 的输出
进入 `parse`，`parse` 的输出进入 `typecheck`。

我们知道我们可以将一个函数提升到 `Either`（和其他函子）上，
我们也可以提升一个返回 `Either` 的函数：

```hs
-- 提醒 fmap 的类型
fmap :: Functor f => (a -> b) -> f a -> f b
-- 专门用于 `Either Error`
fmap :: (a -> b) -> Either Error a -> Either Error b

-- 这里，`a` 是 [Token]，`b` 是 `Either Error AST`：

> fmap parse (tokenize string) :: Either Error (Either Error AST)
```

虽然这段代码可以编译，但它并不好，因为我们正在构建
`Either Error` 的层，我们不能再对
`typecheck` 使用这个技巧了！`typecheck` 期望一个 `AST`，但如果我们尝试在
`fmap parse (tokenize string)` 上 fmap 它，`a` 将是 `Either Error AST`。

我们真正想要的是展平这个结构，而不是嵌套它。
如果我们看一下 `Either Error (Either Error AST)` 可能具有的值的种类，
它看起来像这样：

- `Left <error>`
- `Right (Left error)`
- `Right (Right <ast>)`

---

**练习**：如果我们改用模式匹配呢？这会是什么样子？

<details><summary>解决方案</summary>

```hs
case tokenize string of
  Left err ->
    Left err
  Right tokens ->
    case parse tokens of
      Left err ->
        Left err
      Right ast ->
        typecheck ast
```

如果我们在一个阶段遇到错误，我们返回该错误并停止。如果我们成功，我们
在下一个阶段使用该值。

</details>

---

为 `Either` 展平这个结构与最后一部分非常相似 - `Right tokens`
情况的主体：

```hs
flatten :: Either e (Either e a) -> Either e a
flatten e =
  case e of
    Left l -> Left l
    Right x -> x
```

因为我们有这个函数，我们现在可以在
之前的 `fmap parse (tokenize string) :: Either Error (Either Error AST)`
的输出上使用它：

```
> flatten (fmap parse (tokenize string)) :: Either Error AST
```

现在，我们可以再次使用这个函数来与 `typecheck` 组合：

```hs
> flatten (fmap typecheck (flatten (fmap parse (tokenize string)))) :: Either Error TypedAST
```

这个 `flatten` + `fmap` 的组合看起来像一个重复的模式，
我们可以将它组合成一个函数：

```hs
flatMap :: (a -> Either e b) -> Either e a -> Either e b
flatMap func val = flatten (fmap func val)
```

现在，我们可以这样写代码：

```hs
> flatMap typecheck (flatMap parse (tokenize string)) :: Either Error TypedAST

-- 或者使用反引号语法将函数转换为中缀形式：
> typecheck `flatMap` parse `flatMap` tokenize string

-- 或者创建一个自定义的中缀运算符：(=<<) = flatMap
> typeCheck =<< parse =<< tokenize string
```


这个函数，`flatten`（以及 `flatMap`），在 Haskell 中有不同的名称。
它们被称为
[`join`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Control-Monad.html#v:join)
和 [`=<<`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Control-Monad.html#v:-61--60--60-)
（读作“反向绑定”），
它们是 Haskell 中另一个非常有用的抽象的精髓。

如果我们有一个可以实现的类型：

1. `Functor` 接口，特别是 `fmap` 函数
2. `Applicative` 接口，最重要的是 `pure` 函数
3. 这个 `join` 函数

它们可以实现 `Monad` 类型类的实例。

通过函子，我们能够“提升”一个函数以在实现函子类型类的类型上工作：

```hs
fmap :: (a -> b) -> f a -> f b
```

通过应用函子，我们能够将一个多参数函数“提升”
到实现应用函子类型类的类型的多个值上，
并且还能将一个值提升到该类型中：

```hs
pure :: a -> f a

liftA2 :: (a -> b -> c) -> f a -> f b -> f c
```

通过 monad，我们现在可以展平（或在 Haskell 术语中“join”）实现
`Monad` 接口的类型：

```hs
join :: m (m a) -> m a

-- 这是 =<<，参数反转，读作“绑定”
(>>=) :: m a -> (a -> m b) -> m b
```

使用 `>>=`，我们可以从左到右地编写我们之前的编译流水线，
这对于 monad 来说似乎更受欢迎：

```hs
> tokenize string >>= parse >>= typecheck
```

我们之前在谈论 `IO` 时已经遇到过这个函数了。是的，
`IO` 也实现了 `Monad` 接口。`IO` 的 monadic 接口
帮助我们创建了效果的正确排序。

`Monad` 接口的精髓是 `join`/`>>=` 函数，正如我们所见，
我们可以用 `join` 来实现 `>>=`，我们也可以用 `>>=` 来实现 `join`
（试试看！）。

monadic 接口对于不同的类型可能意味着非常不同的东西。对于 `IO`，这
是效果的排序，对于 `Either`，这是提前截止，
对于 [`Logic`](https://hackage.haskell.org/package/logict-0.7.1.0)，这意味着回溯计算等。

再次强调，不要担心类比和隐喻；专注于 API 和
[定律](https://wiki.haskell.org/Monad_laws)。

> 嘿，你检查过 monad 定律了吗？左单位元、右单位元和结合律？我们已经
> 讨论过一个具有完全相同定律的类型类——`Monoid` 类型类。也许这与
> 关于 monad 只是在某某东西中的 monoid 的著名引言有关……

### Do 表示法？

还记得 [do 表示法](../05-glue/02-io.html#do-notation)吗？事实证明，它适用于任何
`Monad` 实例的类型。这有多酷？而不是写：

```hs
pipeline :: String -> Either Error TypedAST
pipeline string =
  tokenize string >>= \tokens ->
    parse tokens >>= \ast ->
      typecheck ast
```

我们可以写：

```hs
pipeline :: String -> Either Error TypedAST
pipeline string = do
  tokens <- tokenize string
  ast <- parse tokens
  typecheck ast
```

它会工作的！不过，在这种特殊情况下，`tokenize string >>= parse >>= typecheck`
是如此简洁，只有使用
[>=>](https://hackage.haskell.org/package/base-4.16.4.0/docs/Control-Monad.html#v:-62--61--62-)
或
[<=<](https://hackage.haskell.org/package/base-4.16.4.0/docs/Control-Monad.html#v:-60--61--60-)
才能超越它：

```hs
(>=>) :: Monad m => (a -> m b) -> (b -> m c) -> a -> m c
(<=<) :: Monad m => (b -> m c) -> (a -> m b) -> a -> m c

-- 与函数组合比较：
(.) ::              (b ->   c) -> (a ->   b) -> a ->   c
```

```hs
pipeline  = tokenize >=> parse >=> typecheck
```

或

```hs
pipeline = typecheck <=< parse <=< tokenize
```

Haskell 使用抽象创建非常简洁的代码的能力，
一旦熟悉了这些抽象，就会变得非常强大。了解了 monad 抽象，
我们现在已经熟悉了许多库的核心组合 API - 例如：

- [并发](https://hackage.haskell.org/package/stm)
  和[异步编程](https://hackage.haskell.org/package/async)
- [Web 编程](https://gilmi.me/blog/post/2020/12/05/scotty-bulletin-board)
- [测试](http://hspec.github.io/)
- [模拟有状态计算](https://hackage.haskell.org/package/mtl-2.3.1/docs/Control-Monad-State-Lazy.html#g:2)
- [在计算之间共享环境](https://hackage.haskell.org/package/mtl-2.3.1/docs/Control-Monad-Reader.html#g:2)
- 等等。

## 总结

使用 `Either` 进行错误处理有两个原因：

1. 我们使用类型来编码可能的错误，并且我们**强制用户承认并处理**它们，从而
   使我们的代码更能抵抗崩溃和不良行为
2. `Functor`、`Applicative` 和 `Monad` 接口为我们提供了
   **组合**可能失败的函数的机制，（几乎）毫不费力 - 减少样板代码，同时
   保持对我们代码的强有力保证，并将处理错误的需要推迟到
   适当的时候

````