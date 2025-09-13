````markdown
# IO 与 Either？

当我们创建可能需要 I/O 的 `IO` 操作时，我们可能会遇到各种错误。
例如，当我们使用 `writeFile` 时，我们可能会在写入过程中耗尽磁盘空间，
或者文件可能是写保护的。虽然这些情况不是很常见，但它们绝对是
可能的。

我们本可以潜在地将像 `readFile` 和 `writeFile` 这样的 Haskell 函数编码为返回 `Either` 的 `IO` 操作，
例如：

```hs
readFile :: FilePath -> IO (Either ReadFileError String)
writeFile :: FilePath -> String -> IO (Either WriteFileError ())
```

然而，这里有几个问题；首先是组合 `IO` 操作
变得更加困难。以前我们可以写：

```hs
readFile "input.txt" >>= writeFile "output.html"
```

但现在类型不再匹配 - `readFile` 执行时将返回一个 `Either ReadFileError String`，
但 `writeFile` 希望接受一个 `String` 作为输入。我们被迫在调用 `writeFile` 之前处理错误。

## 使用 ExceptT 组合 IO + Either

处理这个问题的一种方法是使用**monad 变换器**。Monad 变换器提供了一种
将 monad 功能相互叠加的方法。它们被称为变换器，因为
**它们接受一个具有 monad 实例的类型作为输入，并返回一个实现 monad 接口的新类型，在其之上叠加一个新的功能**。

例如，如果我们想组合类型等同于 `IO (Either Error a)` 的值，
使用 monadic 接口（函数 `>>=`），我们可以使用一个名为
[`ExceptT`](https://hackage.haskell.org/package/mtl-2.3.1/docs/Control-Monad-Except.html#g:2)
的 monad 变换器，并将其叠加在 `IO` 之上。
让我们看看 `ExceptT` 是如何定义的：

```hs
newtype ExceptT e m a = ExceptT (m (Either e a))
```

记住，`newtype` 是现有类型的新名称。如果我们用 `Error` 替换
`e`，用 `IO` 替换 `m`，我们就会得到我们想要的 `IO (Either Error a)`。
我们可以使用
`runExceptT` 函数将 `ExceptT Error IO a` 转换为 `IO (Either Error a)`：

```hs
runExceptT :: ExceptT e m a -> m (Either e a)
```

`ExceptT` 以一种结合了
`Either` 和它所接受的任何 `m` 的功能的方式实现了 monadic 接口。因为 `ExceptT e m` 有一个 `Monad` 实例，
`>>=` 的一个专门版本会是这样的：

```hs
-- 通用版本
(>>=) :: Monad m => m a -> (a -> m b) -> m b

-- 专门版本，用 `ExceptT e m` 替换上面的 `m`。
(>>=) :: Monad m => ExceptT e m a -> (a -> ExceptT e m b) -> ExceptT e m b
```

请注意，专门版本中的 `m` 仍然需要是 `Monad` 的一个实例。

---

不确定这是如何工作的？尝试为 `IO (Either Error a)` 实现 `>>=`：

```hs
bindExceptT :: IO (Either Error a) -> (a -> IO (Either Error b)) -> IO (Either Error b)
```

<details><summary>解决方案</summary>

```hs
bindExceptT :: IO (Either Error a) -> (a -> IO (Either Error b)) -> IO (Either Error b)
bindExceptT mx f = do
  x <- mx -- `x` 的类型是 `Either Error a`
  case x of
    Left err -> pure (Left err)
    Right y -> f y
```

请注意，我们实际上没有使用 `Error` 或 `IO` 的实现细节，
`Error` 根本没有被提及，对于 `IO`，我们只使用了带有
do 表示法的 monadic 接口。我们可以用一个更通用的类型签名来编写相同的函数：

```hs
bindExceptT :: Monad m => m (Either e a) -> (a -> m (Either e b)) -> m (Either e b)
bindExceptT mx f = do
  x <- mx -- `x` 的类型是 `Either e a`
  case x of
    Left err -> pure (Left err)
    Right y -> f y
```

因为 `newtype ExceptT e m a = ExceptT (m (Either e a))`，我们可以直接
打包和解包那个 `ExceptT` 构造函数，得到：


```hs
bindExceptT :: Monad m => ExceptT e m a -> (a -> ExceptT e m b) -> ExceptT e m b
bindExceptT mx f = ExceptT $ do
  -- `runExceptT mx` 的类型是 `m (Either e a)`
  -- `x` 的类型是 `Either e a`
  x <- runExceptT mx
  case x of
    Left err -> pure (Left err)
    Right y -> runExceptT (f y)
```

</details>

---

> 请注意，当叠加 monad 变换器时，我们叠加它们的顺序很重要。
> 对于 `ExceptT Error IO a`，我们有一个 `IO` 操作，当运行时将返回 `Either`
> 一个错误或一个值。

`ExceptT` 可以两全其美 - 我们可以使用 `throwError` 函数返回错误值：

```hs
throwError :: e -> ExceptT e m a
```

我们也可以“提升”返回底层 monadic 类型 `m` 的值的函数，
使其改为返回 `ExceptT e m a` 的值：

```hs
lift :: m a -> ExceptT e m a
```

例如：

```hs
getLine :: IO String

lift getLine :: ExceptT e IO String
```

> 实际上，`lift` 也是 `MonadTrans`（monad 变换器的类型类）中的一个类型类函数。所以技术上，`lift getLine :: MonadTrans t => t IO String`，
> 但为了具体起见，我们进行了专门化。


现在，如果我们有：

```hs
readFile :: FilePath -> ExceptT IOError IO String

writeFile :: FilePath -> String -> ExceptT IOError IO ()
```

我们可以毫无问题地再次组合它们：

```hs
readFile "input.txt" >>= writeFile "output.html"
```

但请记住 - 错误类型 `e`（在 `Either` 和 `Except` 的情况下）
在组合的函数之间必须是相同的！这意味着 `readFile` 和 `writeFile` 的错误表示类型必须相同 - 这也会
迫使任何使用这些函数的人处理这些错误 - 调用 `writeFile` 的用户是否应该被要求处理“文件未找到”错误？调用 `readFile` 的用户是否应该被要求处理“磁盘空间不足”错误？
还有很多很多可能的 IO 错误！“网络不可达”、“内存不足”、
“线程已取消”，我们不能要求用户处理所有这些错误，或者
甚至在数据类型中全部覆盖它们。

那么我们该怎么办呢？

我们放弃这种**用于 IO 代码**的方法，并使用另一种方法：异常，
我们将在下一章中看到。

> 注意 - 当我们将 `ExceptT` 叠加在另一个名为
> [`Identity`](https://hackage.haskell.org/package/base-4.16.4.0/docs/Data-Functor-Identity.html)
> 的类型之上时，该类型也实现了 `Monad` 接口，我们会得到一个与 `Either`
> 完全相同的类型，称为 [`Except`](https://hackage.haskell.org/package/transformers-0.6.0.2/docs/Control-Monad-Trans-Except.html#t:Except)
> （末尾没有 `T`）。你有时可能想使用 `Except` 而不是 `Either`，
> 因为它有一个比 `Either` 更合适的名称和更好的错误处理 API。

````