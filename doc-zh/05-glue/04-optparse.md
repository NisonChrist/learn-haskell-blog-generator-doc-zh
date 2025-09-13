````markdown
# 优雅的选项解析

我们希望为我们的程序定义一个更友好的界面。虽然我们可以通过 `getArgs` 和模式匹配自己实现一些功能，但使用库可以更容易地获得好的结果。我们将使用一个名为
[optparse-applicative](https://hackage.haskell.org/package/optparse-applicative) 的包。

`optparse-applicative` 为我们提供了一个 EDSL（是的，又一个）来构建
命令行参数解析器。像命令、开关和标志这样的东西可以被构建
并组合在一起，为命令行参数创建一个解析器，而无需像我们编写标记解析器时那样实际编写字符串操作，并且将
提供其他好处，例如自动生成用法行、帮助屏幕、
错误报告等等。

虽然 `optparse-applicative` 的依赖足迹不是很大，
但在这种特殊情况下，我们库的用户很可能不需要命令行解析，因此将此依赖项添加到 `.cabal` 文件的 `executable` 部分
（而不是 `library` 部分）是有意义的：

```diff
 executable hs-blog-gen
   import: common-settings
   hs-source-dirs: app
   main-is: Main.hs
   build-depends:
       base
+    , optparse-applicative
     , hs-blog
   ghc-options:
     -O
```

## 构建命令行解析器

optparse-applicative 包有相当不错的
[文档](https://hackage.haskell.org/package/optparse-applicative-0.16.1.0#optparse-applicative)，
但我们将在本章中介绍一些需要注意的重要事项。

总的来说，我们需要做四件重要的事情：

1. 定义我们的模型 - 我们希望定义一个 ADT 来描述我们程序的各种选项
   和命令

2. 定义一个解析器，当运行时将产生我们模型类型的值

3. 在我们的程序参数输入上运行解析器

4. 对模型进行模式匹配，并根据选项调用正确的操作

### 定义模型

让我们先设想一下我们的命令行界面；它应该
是什么样子？

我们希望能够将单个文件或输入流转换为文件
或输出流，或者我们希望处理整个目录并创建一个新目录。
我们可以用这样的 ADT 来建模：

```hs
data Options
  = ConvertSingle SingleInput SingleOutput
  | ConvertDir FilePath FilePath
  deriving Show

data SingleInput
  = Stdin
  | InputFile FilePath
  deriving Show

data SingleOutput
  = Stdout
  | OutputFile FilePath
  deriving Show
```

> 请注意，我们技术上也可以使用 `Maybe FilePath` 来编码 `SingleInput`
> 和 `SingleOutput`，但那样我们就必须记住 `Nothing` 在
> 每种上下文中的含义。通过为每个选项创建一个具有适当命名构造函数的新类型，
> 我们可以让代码的读者更容易理解
> 我们代码的含义。

在接口方面，我们可以决定当用户想要转换
单个输入源时，他们将使用 `convert` 命令，并提供可选的标志
`--input FILEPATH` 和 `--output FILEPATH` 来从文件中读取或写入。
当用户不提供一个或两个标志时，我们将相应地从
标准输入/输出读取或写入。

如果用户想要转换一个目录，他们可以使用 `convert-dir`
命令，并提供两个强制性标志 `--input FILEPATH` 和
`--output FILEPATH`。

### 构建解析器

这是这个过程中最有趣的部分。我们如何构建一个
适合我们模型的解析器？

`optparse-applicative` 库引入了一个名为 `Parser` 的新类型。
`Parser`，类似于 `Maybe` 和 `IO`，其种类为 `* -> *` - 当它
被提供一个饱和（或具体）类型，如 `Int`、`Bool` 或
`Options` 时，它可以成为一个饱和类型（一个有值的类型）。

 `Parser a` 表示一个命令行选项解析器的规范，
当命令行参数被
成功解析时，它会产生一个 `a` 类型的值。
这类似于 `IO a` 如何表示一个可以产生 `a` 类型值的
程序的描述。这两种
类型之间的主要区别在于，虽然我们不能将 `IO a` 转换为 `a`
（我们只是链接 IO 操作并让 Haskell 运行时执行它们），
但我们*可以*将 `Parser a` 转换为一个函数，该函数接受一个表示
程序参数的字符串列表，如果它能
解析参数，则产生一个 `a`。

正如我们之前在 EDSL 中看到的那样，这个库也使用了*组合子模式*
。我们需要考虑构建
解析器的基本原语以及将小解析器组合成大
解析器的方法。

让我们看一个小解析器的例子：

```hs
inp :: Parser FilePath
inp =
  strOption
    ( long "input"
      <> short 'i'
      <> metavar "FILE"
      <> help "Input file"
    )

out :: Parser FilePath
out =
  strOption
    ( long "output"
      <> short 'o'
      <> metavar "FILE"
      <> help "Output file"
    )
```

`strOption` 是一个解析器构建器。它是一个函数，接受一个组合的
*选项修饰符*作为参数，并返回一个将解析字符串的解析器。
我们可以将类型指定为 `FilePath`，因为 `FilePath` 是
`String` 的别名。解析器构建器描述了如何解析值，
而修饰符描述了它的属性，例如标志名称、
标志名称的简写，以及它在用法
和帮助消息中的描述方式。

> 实际上，`strOption` 可以返回任何实现了 `IsString` 接口的
> 字符串类型。有几种这样的类型，
> 例如，`Text`，一个来自 `text` 包的更高效的 Unicode 文本类型。
> 它比 `String` 更高效，因为 `String` 是作为 `Char` 的
> 链表实现的，而 `Text` 是作为字节数组实现的。
> `Text` 通常是我们应该用于文本值的类型，而不是 `String`。我们到目前为止
> 没有使用它，因为它比 `String` 使用起来稍微不那么符合人体工程学。
> 但它通常是用于文本的首选类型！

如你所见，修饰符可以使用 `<>` 函数进行组合，
这意味着修饰符实现了 `Semigroup` 类型类的实例！

有了这样的接口，我们不必提供所有的修饰符
选项，只需提供相关的即可。所以如果我们不想要
一个缩短的标志名，我们就不必添加它。

#### Functor

对于我们定义的数据类型，拥有 `Parser FilePath` 让我们
朝着正确的方向迈出了一大步，但这并不是我们对 `ConvertSingle` 所需要的。
我们需要一个 `Parser SingleInput` 和一个
`Parser SingleOutput`。如果我们有一个 `FilePath`，我们可以通过使用 `InputFile` 构造函数将其转换为
`SingleInput`。
记住，`InputFile` 也是一个函数：

```hs
InputFile :: FilePath -> SingleInput
OutputFile :: FilePath -> SingleOutput
```

然而，要转换一个解析器，我们需要具有这些类型的函数：

```hs
f :: Parser FilePath -> Parser SingleInput
g :: Parser FilePath -> Parser SingleOutput
```

幸运的是，`Parser` 接口为我们提供了一个函数来“提升”
像 `FilePath -> SingleInput` 这样的函数以在解析器上工作，使其
成为一个类型为 `Parser FilePath -> Parser SingleInput` 的函数。
当然，这个函数适用于任何输入和输出，
所以如果我们有一个类型为 `a -> b` 的函数，我们可以将它传递给
那个函数，并得到一个类型为 `Parser a -> Parser b` 的新函数。

这个函数叫做 `fmap`：

```hs
fmap :: (a -> b) -> Parser a -> Parser b

-- 或者它的中缀版本
(<$>)  :: (a -> b) -> Parser a -> Parser b
```

我们之前在其他类型的接口中见过 `fmap`：

```hs
fmap :: (a -> b) -> [a] -> [b]

fmap :: (a -> b) -> IO a -> IO b
```

`fmap` 是一个类型类函数，就像 `<>` 和 `show` 一样。它属于
[`Functor`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Functor.html#t:Functor) 类型类：

```hs
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

它有以下定律：

```hs
-- 1. 恒等律：
--    如果我们不改变值，任何东西都不应该改变
fmap id = id

-- 2. 组合律：
--    组合提升后的函数与在 fmap 之后
--    组合它们是相同的
fmap (f . g) == fmap f . fmap g
```

任何能够实现 `fmap` 并遵循这些定律的类型 `f` 都可以是 `Functor` 的有效
实例。

> 注意 `f` 的种类是 `* -> *`，我们可以通过查看 `fmap` 类型签名中的其他类型来推断 `f` 的种类：
>
> 1. `a` 和 `b` 的种类是 `*`，因为它们被用作函数的参数/返回
> 类型
> 2. `f a` 的种类是 `*`，因为它被用作函数的参数，因此
> 3. `f` 的种类是 `* -> *`

让我们选择一个数据类型，看看我们是否可以实现一个 `Functor` 实例。
我们需要选择一个种类为 `* -> *` 的数据类型。`Maybe` 符合要求。
我们需要实现一个函数 `fmap :: (a -> b) -> Maybe a -> Maybe b`。
这是一个非常简单（且错误）的实现：

```hs
mapMaybe :: (a -> b) -> Maybe a -> Maybe b
mapMaybe func maybeX = Nothing
```

自己检查一下！它编译成功了！但不幸的是，它不
满足第一条定律。`fmap id = id` 意味着
`mapMaybe id (Just x) == Just x`，然而，从定义中，我们可以
清楚地看到 `mapMaybe id (Just x) == Nothing`。

这是一个很好的例子，说明 Haskell 如何不帮助我们确保定律
得到满足，以及为什么它们很重要。不合法的 `Functor` 实例
的行为将与我们期望 `Functor` 的行为不同。
让我们再试一次！

```hs
mapMaybe :: (a -> b) -> Maybe a -> Maybe b
mapMaybe func maybeX =
  case maybeX of
    Nothing -> Nothing
    Just x -> Just (func x)
```

这个 `mapMaybe` 将满足函子定律。这可以通过
代数来证明——如果我们可以通过替换在
每个定律的等式两边达到另一边，那么定律就成立。

Functor 是一个非常重要的类型类，许多类型都实现了这个接口。
我们知道，`IO`、`Maybe`、`[]` 和 `Parser` 的种类都是 `* -> *`，
并且都允许我们映射它们的“有效载荷”类型。

> 人们常常试图寻找关于类型类含义的类比和隐喻，
> 但像 `Functor` 这样名字有趣的类型类通常没有
> 在所有情况下都适用的类比或隐喻。放弃
> 隐喻，把它看作它本身——一个有定律的接口——会更容易。

我们可以对 `Parser` 使用 `fmap`，使一个返回 `FilePath` 的解析器
改为返回 `SingleInput` 或 `SingleOutput`：

```hs
pInputFile :: Parser SingleInput
pInputFile = fmap InputFile parser
  where
    parser =
      strOption
        ( long "input"
          <> short 'i'
          <> metavar "FILE"
          <> help "Input file"
        )

pOutputFile :: Parser SingleOutput
pOutputFile = OutputFile <$> parser -- fmap 和 <$> 是相同的
  where
    parser =
      strOption
        ( long "output"
          <> short 'o'
          <> metavar "FILE"
          <> help "Output file"
        )
```

#### Applicative

现在我们有两个解析器，
`pInputFile :: Parser SingleInput`
和 `pOutputFile :: Parser SingleOutput`，
我们想将它们*组合*成 `Options`。同样，如果我们只有
`SingleInput` 和 `SingleOutput`，我们可以使用构造函数 `ConvertSingle`：

```hs
ConvertSingle :: SingleInput -> SingleOutput -> Options
```

我们能像之前用 `fmap` 那样玩个类似的把戏吗？
是否存在一个函数可以将一个二元函数提升到
`Parser` 上工作？一个具有这种类型签名的函数：

```
???
  :: (SingleInput -> SingleOutput -> Options)
  -> (Parser SingleInput -> Parser SingleOutput -> Parser Options)
```

是的。这个函数叫做 `liftA2`，它来自 `Applicative`
类型类。`Applicative`（也称为 applicative functor）有三个
主要函数：

```hs
class Functor f => Applicative f where
  pure :: a -> f a
  liftA2 :: (a -> b -> c) -> f a -> f b -> f c
  (<*>) :: f (a -> b) -> f a -> f b
```

[`Applicative`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Control-Applicative.html#t:Applicative)
是另一个非常流行的类型类，有很多实例。

就像任何 `Monoid` 都是 `Semigroup` 一样，任何 `Applicative`
都是 `Functor`。这意味着任何想要实现
`Applicative` 接口的类型也应该实现 `Functor` 接口。

除了常规函子可以做的事情，即在
某个 `f` 上提升一个函数，应用函子允许我们将一个函数应用于
某个 `f` 的*多个实例*，以及将任何 `a` 类型的值“提升”为一个 `f a`。

你应该已经熟悉 `pure` 了，我们在
谈论 `IO` 时见过它。对于 `IO`，`pure` 让我们创建一个
具有特定返回值的 `IO` 动作，而无需执行 IO。
对于 `Parser` 的 `pure`，我们可以创建一个 `Parser`，当运行时，
它将返回一个特定的值作为输出，而无需进行任何解析。

`liftA2` 和 `<*>` 是两个可以相互实现的函数。
`<*>` 实际上是两者中更有用的一个。因为当与 `fmap`（或者更确切地说是中缀版本 `<$>`）
结合使用时，它可以用于应用具有多个参数的函数，而不仅仅是两个。

要将我们的两个解析器组合成一个，我们可以使用 `liftA2` 或
`<$>` 和 `<*>` 的组合：

```hs
-- 使用 liftA2
pConvertSingle :: Parser Options
pConvertSingle =
  liftA2 ConvertSingle pInputFile pOutputFile

-- 使用 <$> 和 <*>
pConvertSingle :: Parser Options
pConvertSingle =
  ConvertSingle <$> pInputFile <*> pOutputFile
```

请注意，`<$>` 和 `<*>` 都是左结合的，
所以我们有像这样的隐式括号：

```hs
pConvertSingle :: Parser Options
pConvertSingle =
  (ConvertSingle <$> pInputFile) <*> pOutputFile
```

让我们更深入地看一下
我们这里的子表达式的类型，以证明这可以进行类型检查：

```hs
pConvertSingle :: Parser Options

pInputFile :: Parser SingleInput
pOutputFile :: Parser SingleOutput

ConvertSingle :: SingleInput -> SingleOutput -> Options

(<$>) :: (a -> b) -> Parser a -> Parser b
  -- 具体来说，这里的 `a` 是 `SingleInput`
  -- `b` 是 `SingleOutput -> Options`，

ConvertSingle <$> pInputFile :: Parser (SingleOutput -> Options)

(<*>) :: Parser (a -> b) -> Parser a -> Parser b
  -- 具体来说，这里的 `a -> b` 是 `SingleOutput -> Options`
  -- 所以 `a` 是 `SingleOutput`，`b` 是 `Options`

-- 所以我们得到：
(ConvertSingle <$> pInputFile) <*> pOutputFile :: Parser Options
```

通过 `<$>` 和 `<*>`，我们可以链接任意数量的解析器（或任何应用函子）。
这是因为两件事：柯里化和参数多态性。
因为 Haskell 中的函数只接受一个参数并返回一个，
所以任何多参数函数都可以表示为 `a -> b`。

> 你可以在这篇名为
> [Typeclassopedia](https://wiki.haskell.org/Typeclassopedia#Laws_2) 的文章中找到应用函子的定律，
> 这篇文章讨论了各种有用的类型类及其定律。

应用函子是一个非常重要的概念，它将出现在各种
解析器接口中（不仅是命令行参数，还有 JSON
解析器和通用解析器）、I/O、并发、非确定性等等。
这个库之所以被称为 optparse-applicative，是因为
它使用 `Applicative` 接口作为
构建解析器的主要 API。

---

**练习**：为 `Options` 的 `ConvertDir` 构造函数创建一个类似的接口。

<details><summary>解决方案</summary>

```hs
pInputDir :: Parser FilePath
pInputDir =
  strOption
    ( long "input"
      <> short 'i'
      <> metavar "DIRECTORY"
      <> help "Input directory"
    )

pOutputDir :: Parser FilePath
pOutputDir =
  strOption
    ( long "output"
      <> short 'o'
      <> metavar "DIRECTORY"
      <> help "Output directory"
    )

pConvertDir :: Parser Options
pConvertDir =
  ConvertDir <$> pInputDir <*> pOutputDir
```

</details>

---

#### Alternative

我们忘记了一件事，那就是 `ConvertSingle` 的每个输入和输出
也可能使用标准输入和输出。
到目前为止，我们只提供了一个选项：通过指定 `--input` 和 `--output` 标志
从文件读取或写入文件。
然而，我们希望使这些标志成为可选的，当它们
未指定时，使用备选的标准 I/O。我们可以使用
`Control.Applicative` 中的 `optional` 函数来做到这一点：

```hs
optional :: Alternative f => f a -> f (Maybe a)
```

`optional` 适用于实现了
[`Alternative`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Control-Applicative.html#t:Alternative) 类型类实例的类型：

```hs
class Applicative f => Alternative f where
  (<|>) :: f a -> f a -> f a
  empty :: f a
```

`Alternative` 看起来与 `Monoid` 类型类非常相似，
但它适用于应用函子。据我所知，这个类型类
不是很常见，主要用于解析库。
它为我们提供了一个接口来组合两个 `Parser`——
如果第一个解析失败，则尝试另一个。
它还提供了其他有用的函数，例如 `optional`，
这将有助于我们的情况：

```hs
pSingleInput :: Parser SingleInput
pSingleInput =
  fromMaybe Stdin <$> optional pInputFile

pSingleOutput :: Parser SingleOutput
pSingleOutput =
  fromMaybe Stdout <$> optional pOutputFile
```

请注意，使用 `fromMaybe :: a -> Maybe a -> a`，我们可以通过为 `Nothing` 情况提供一个值来从 `Maybe` 中提取
`a`。

现在我们可以在 `pConvertSingle` 中使用这些更合适的函数：

```hs
pConvertSingle :: Parser Options
pConvertSingle =
  ConvertSingle <$> pSingleInput <*> pSingleOutput
```

#### 命令和子解析器

我们目前在接口中有两个可能的操作，
转换单个源，或转换目录。一个用于
选择正确操作的良好接口是通过命令。
如果用户想要转换单个源，他们可以使用
`convert`，对于目录，可以使用 `convert-dir`。

我们可以使用 `subparser` 和 `command`
函数创建一个带有命令的解析器：

```hs
subparser :: Mod CommandFields a -> Parser a

command :: String -> ParserInfo a -> Mod CommandFields a
```

`subparser` 接受*命令修饰符*（可以使用
`command` 函数构造）作为输入，并产生一个 `Parser`。
`command` 接受命令名称（在我们的例子中是“convert”或“convert-dir”）
和一个 `ParserInfo a`，并产生一个命令修饰符。正如我们
之前看到的，这些修饰符有一个 `Monoid` 实例，它们可以被
组合，这意味着我们可以附加多个命令作为备选项。

一个 `ParserInfo a` 可以用 `info` 函数构造：

```hs
info :: Parser a -> InfoMod a -> ParserInfo a
```

这个函数用一些额外的信息包装一个 `Parser`，
例如帮助消息、描述等等，以便程序
本身和每个子命令都可以打印一些额外的信息。

让我们看看如何构造一个 `ParserInfo`：

```hs
pConvertSingleInfo :: ParserInfo Options
pConvertSingleInfo =
  info
    (helper <*> pConvertSingle)
    (progDesc "Convert a single markup source to html")
```
请注意，`helper` 在解析器失败时会添加一个帮助输出屏幕。

让我们也构建一个命令：

```hs
pConvertSingleCommand :: Mod CommandFields Options
pConvertSingleCommand =
  command "convert" pConvertSingleInfo
```

尝试使用 `subparser` 组合这两个选项来创建一个 `Parser Options`。

<details><summary>解决方案</summary>

```hs
pOptions :: Parser Options
pOptions =
  subparser
    ( command
      "convert"
      ( info
        (helper <*> pConvertSingle)
        (progDesc "Convert a single markup source to html")
      )
      <> command
      "convert-dir"
      ( info
        (helper <*> pConvertDir)
        (progDesc "Convert a directory of markup files to html")
      )
    )
```

</details>

#### ParserInfo

既然我们已经完成了构建解析器，我们应该将它包装在一个 `ParserInfo` 中
并添加一些信息，使其准备好运行：

```hs
opts :: ParserInfo Options
opts =
  info (helper <*> pOptions)
    ( fullDesc
      <> header "hs-blog-gen - a static blog generator"
      <> progDesc "Convert markup files or directories to html"
    )
```

### 运行解析器

`optparse-applicative` 提供了一个非 `IO` 接口来解析参数，
但使用它的最方便的方法是让它负责获取
程序参数，尝试解析它们，并在失败时抛出错误和帮助消息。
这可以通过函数 `execParser :: ParserInfo a -> IO a` 来完成。

我们可以将所有这些选项解析的东西放在一个新模块中，
然后从 `app/Main.hs` 中导入它。让我们这样做。
这是我们到目前为止所拥有的：

<details><summary>app/OptParse.hs</summary>

```hs
-- | 命令行选项解析

module OptParse
  ( Options(..)
  , SingleInput(..)
  , SingleOutput(..)
  , parse
  )
  where

import Data.Maybe (fromMaybe)
import Options.Applicative

------------------------------------------------
-- * 我们的命令行选项模型

-- | 模型
data Options
  = ConvertSingle SingleInput SingleOutput
  | ConvertDir FilePath FilePath
  deriving Show

-- | 单个输入源
data SingleInput
  = Stdin
  | InputFile FilePath
  deriving Show

-- | 单个输出接收器
data SingleOutput
  = Stdout
  | OutputFile FilePath
  deriving Show

------------------------------------------------
-- * 解析器

-- | 解析命令行选项
parse :: IO Options
parse = execParser opts

opts :: ParserInfo Options
opts =
  info (pOptions <**> helper)
    ( fullDesc
      <> header "hs-blog-gen - a static blog generator"
      <> progDesc "Convert markup files or directories to html"
    )

-- | 所有选项的解析器
pOptions :: Parser Options
pOptions =
  subparser
    ( command
      "convert"
      ( info
        (helper <*> pConvertSingle)
        (progDesc "Convert a single markup source to html")
      )
      <> command
      "convert-dir"
      ( info
        (helper <*> pConvertDir)
        (progDesc "Convert a directory of markup files to html")
      )
    )

------------------------------------------------
-- * 单源到接收器转换解析器

-- | 单源到接收器选项的解析器
pConvertSingle :: Parser Options
pConvertSingle =
  ConvertSingle <$> pSingleInput <*> pSingleOutput

-- | 单输入源的解析器
pSingleInput :: Parser SingleInput
pSingleInput =
  fromMaybe Stdin <$> optional pInputFile

-- | 单输出接收器的解析器
pSingleOutput :: Parser SingleOutput
pSingleOutput =
  fromMaybe Stdout <$> optional pOutputFile

-- | 输入文件解析器
pInputFile :: Parser SingleInput
pInputFile = fmap InputFile parser
  where
    parser =
      strOption
        ( long "input"
          <> short 'i'
          <> metavar "FILE"
          <> help "Input file"
        )

-- | 输出文件解析器
pOutputFile :: Parser SingleOutput
pOutputFile = OutputFile <$> parser
  where
    parser =
      strOption
        ( long "output"
          <> short 'o'
          <> metavar "FILE"
          <> help "Output file"
        )

------------------------------------------------
-- * 目录转换解析器

pConvertDir :: Parser Options
pConvertDir =
  ConvertDir <$> pInputDir <*> pOutputDir

-- | 输入目录的解析器
pInputDir :: Parser FilePath
pInputDir =
  strOption
    ( long "input"
      <> short 'i'
      <> metavar "DIRECTORY"
      <> help "Input directory"
    )

-- | 输出目录的解析器
pOutputDir :: Parser FilePath
pOutputDir =
  strOption
    ( long "output"
      <> short 'o'
      <> metavar "DIRECTORY"
      <> help "Output directory"
    )
```

</details>

### 对 Options 进行模式匹配

在运行命令行参数解析器之后，我们可以对
我们的模型进行模式匹配并调用正确的函数。目前，我们的程序
没有暴露这种 API。所以让我们去我们的 `src/HsBlog.hs`
模块并更改 API。我们可以从该文件中删除 `main` 并
添加两个新函数：

```hs
convertSingle :: Html.Title -> Handle -> Handle -> IO ()

convertDirectory :: FilePath -> FilePath -> IO ()
```

[`Handle`](https://hackage.haskell.org/package/base-4.16.4.0/docs/System-IO.html#t:Handle)
是文件系统对象（包括 `stdin` 和 `stdout`）的 I/O 抽象。
之前，我们使用 `writeFile` 和 `getContents` - 这些函数要么
获取一个 `FilePath` 来打开和操作，要么它们假定 `Handle` 是标准 I/O。
我们可以使用 `System.IO` 中接受 `Handle` 的显式版本：

```hs
convertSingle :: Html.Title -> Handle -> Handle -> IO ()
convertSingle title input output = do
  content <- hGetContents input
  hPutStrLn output (process title content)
```

我们将暂时不实现 `convertDirectory`，并在下一章中实现它。

在 `app/Main.hs` 中，我们需要对 `Options` 进行模式匹配，并
准备从 `HsBlog` 调用正确的函数。

让我们看一下我们完整的 `app/Main.hs` 和 `src/HsBlog.hs`：

<details><summary>app/Main.hs</summary>

```hs
-- | hs-blog-gen 程序的入口点

module Main where

import OptParse
import qualified HsBlog

import System.Exit (exitFailure)
import System.Directory (doesFileExist)
import System.IO

main :: IO ()
main = do
  options <- parse
  case options of
    ConvertDir input output ->
      HsBlog.convertDirectory input output

    ConvertSingle input output -> do
      (title, inputHandle) <-
        case input of
          Stdin ->
            pure ("", stdin)
          InputFile file ->
            (,) file <$> openFile file ReadMode

      outputHandle <-
        case output of
          Stdout -> pure stdout
          OutputFile file -> do
            exists <- doesFileExist file
            shouldOpenFile <-
              if exists
                then confirm
                else pure True
            if shouldOpenFile
              then
                openFile file WriteMode
              else
                exitFailure

      HsBlog.convertSingle title inputHandle outputHandle
      hClose inputHandle
      hClose outputHandle

------------------------------------------------
-- * 实用工具

-- | 确认用户操作
confirm :: IO Bool
confirm =
  putStrLn "Are you sure? (y/n)" *>
    getLine >>= \answer ->
      case answer of
        "y" -> pure True
        "n" -> pure False
        _ -> putStrLn "Invalid response. use y or n" *>
          confirm
```

</details>

<details><summary>src/HsBlog.hs</summary>

```hs
-- HsBlog.hs
module HsBlog
  ( convertSingle
  , convertDirectory
  , process
  )
  where

import qualified HsBlog.Markup as Markup
import qualified HsBlog.Html as Html
import HsBlog.Convert (convert)

import System.IO

convertSingle :: Html.Title -> Handle -> Handle -> IO ()
convertSingle title input output = do
  content <- hGetContents input
  hPutStrLn output (process title content)

convertDirectory :: FilePath -> FilePath -> IO ()
convertDirectory = error "Not implemented"

process :: Html.Title -> String -> String
process title = Html.render . convert title . Markup.parse
```

</details>

我们需要对 `.cabal` 文件做一些小的改动。

首先，我们需要将依赖项 `directory` 添加到 `executable` 中，
因为我们在 `Main` 中使用了 `System.Directory` 库。

其次，我们需要在 `executable` 的模块列表中列出 `OptParse`。

```diff
 executable hs-blog-gen
   import: common-settings
   hs-source-dirs: app
   main-is: Main.hs
+  other-modules:
+    OptParse
   build-depends:
       base
+    , directory
     , optparse-applicative
     , hs-blog
   ghc-options:
     -O
```

## 总结

我们学习了一个名为 `optparse-applicative` 的新奇库，
并用它以声明的方式创建了一个更花哨的命令行界面。
看看运行 `hs-blog-gen --help`（或我们在上一章中讨论的等效的
`cabal`/`stack` 命令）的结果：

```
hs-blog-gen - a static blog generator

Usage: hs-blog-gen COMMAND
  Convert markup files or directories to html

Available options:
  -h,--help                Show this help text

Available commands:
  convert                  Convert a single markup source to html
  convert-dir              Convert a directory of markup files to html
```

在此过程中，我们学习了两个强大的新抽象，`Functor`
和 `Applicative`，并重温了一个名为
`Monoid` 的抽象。通过这个库，我们看到了另一个
这些抽象在构建 API 和 EDSL 方面的用处。

我们将在本书的其余部分继续遇到这些抽象。

---

**附加练习**：添加另一个名为 `--replace` 的标志，以指示
如果输出文件或目录已存在，则可以替换它们。

---

> 你可以查看
> [我们所做更改](https://github.com/soupi/learn-haskell-blog-generator/commit/d0d76aad632fe3abd8701e44db5ba687e0c7ac96)
> 的 git 提交和
> [到目前为止的代码](https://github.com/soupi/learn-haskell-blog-generator/tree/d0d76aad632fe3abd8701e44db5ba687e0c7ac96)。

````