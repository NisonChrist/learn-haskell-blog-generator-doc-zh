````markdown
# 灵活的 HTML 内容（函数）

我们希望能够编写不同的 HTML 页面，而不必一遍又一遍地编写 `<html>` 和 `<body>` 标签的整个结构。我们可以用函数来做到这一点。

要定义一个函数，我们像前面看到的那样创建一个定义，并在名称和等号 (`=`) 之间添加参数名称。所以一个函数定义有以下形式：

```hs
<name> <arg1> <arg2> ... <argN> = <expression>
```

参数名称将在等号右侧（在 `<expression>` 中）的范围内可用，函数名称将是 `<name>`。

我们将定义一个函数，它接受一个字符串（页面的内容），并通过在内容前后连接它们，将其包装在相关的 `<html>` 和 `<body>` 标签中。我们使用 `<>` 运算符来连接两个字符串。

```hs
wrapHtml content = "<html><body>" <> content <> "</body></html>"
```

这个函数 `wrapHtml` 接受一个名为 `content` 的参数，并返回一个在内容前加上 `<html><body>` 并在其后附加 `</body></html>` 的字符串。请注意，在 Haskell 中，名称通常使用驼峰式命名法。

现在我们可以调整上一章中的 `myhtml` 定义：

```hs
myhtml = wrapHtml "Hello, world!"
```

再次注意，调用函数时我们不需要括号。函数调用的形式如下：

```hs
<name> <arg1> <arg2> ... <argN>
```

但是，如果我们想在 `main = putStrLn myhtml` 中用 `myhtml` 绑定的表达式替换 `myhtml`，我们就必须将表达式用括号括起来：

```hs
main = putStrLn (wrapHtml "Hello, world!")
```

如果我们不小心写成这样：

```hs
main = putStrLn wrapHtml "Hello, world!"
```

我们会从 GHC 收到一个错误，指出 `putStrLn` 应用于两个参数，但它只接受一个。这是因为上面是 `<name> <arg1> <arg2>` 的形式，其中，正如我们前面定义的，`<arg1>` 和 `<arg2>` 是 `<name>` 的参数。

使用括号，我们可以按正确的顺序将表达式组合在一起。

> #### 关于运算符优先级和固定性的旁注
>
> 运算符（如 `<>`）是中缀函数，它接受两个参数——每边一个。
>
> 当同一表达式中没有括号的多个运算符时，运算符的*固定性*（左或右）和*优先级*（0 到 10 之间的数字）决定了哪个运算符绑定得更紧密。
>
> 在我们的例子中，`<>` 具有*右*固定性，所以 Haskell 在 `<>` 的右侧添加了一个不可见的括号。所以，例如：
>
> ```hs
> "<html><body>" <> content <> "</body></html>"
> ```
>
> Haskell 将其视为：
>
> ```hs
> "<html><body>" <> (content <> "</body></html>")
> ```
>
> 对于优先级的例子，在表达式 `1 + 2 * 3` 中，运算符 `+` 的优先级为 6，运算符 `*` 的优先级为 7，所以我们让 `*` 优先于 `+`。Haskell 会将此表达式视为：
>
> ```hs
> 1 + (2 * 3)
> ```
>
> 当混合使用具有*相同优先级*但*不同固定性*的不同运算符时，您可能会遇到错误，因为 Haskell 不知道如何对这些表达式进行分组。在这种情况下，我们可以通过显式添加括号来解决问题。

---

练习：

1.  将 `wrapHtml` 的功能分成两个函数：
    1.  一个将内容包装在 `<html>` 标签中
    2.  一个将内容包装在 `<body>` 标签中

    将新函数命名为 `html_` 和 `body_`。
2.  更改 `myhtml` 以使用这两个函数。
3.  为 `<head>` 和 `<title>` 标签添加另外两个类似的函数，并将其命名为 `head_` 和 `title_`。
4.  创建一个新函数 `makeHtml`，它接受两个字符串作为输入：
    1.  一个用于标题的字符串
    2.  一个用于正文内容的字符串

    并使用前面练习中实现的函数构造一个 HTML 字符串。

    对于：

    ```hs
    makeHtml "My page title" "My page content"
    ```

    输出应该是：

    ```html
    <html><head><title>My page title</title></head><body>My page content</body></html>
    ```
5.  在 `myhtml` 中使用 `makeHtml`，而不是直接使用 `html_` 和 `body_`

---

解决方案：

<details>
  <summary>练习 #1 的解决方案</summary>
  
  ```hs
  html_ content = "<html>" <> content <> "</html>"
     
  body_ content = "<body>" <> content <> "</body>"
  ```

</details>

<details>
  <summary>练习 #2 的解决方案</summary>
  
  ```hs
  myhtml = html_ (body_ "Hello, world!")
  ```

</details>

<details>
  <summary>练习 #3 的解决方案</summary>
  
  ```hs
  head_ content = "<head>" <> content <> "</head>"
  
  title_ content = "<title>" <> content <> "</title>"
  ```

</details>

<details>
  <summary>练习 #4 的解决方案</summary>
  
  ```hs
  makeHtml title content = html_ (head_ (title_ title) <> body_ content)
  ```

</details>


<details>
  <summary>练习 #5 的解决方案</summary>
  
  ```hs
  myhtml = makeHtml "Hello title" "Hello, world!"
  ```

</details>


<details>
  <summary>我们最终的程序</summary>
  
  ```hs
  -- hello.hs

  main = putStrLn myhtml

  myhtml = makeHtml "Hello title" "Hello, world!"

  makeHtml title content = html_ (head_ (title_ title) <> body_ content)

  html_ content = "<html>" <> content <> "</html>"
     
  body_ content = "<body>" <> content <> "</body>"

  head_ content = "<head>" <> content <> "</head>"
  
  title_ content = "<title>" <> content <> "</title>"
  ```

   我们现在可以运行我们的 `hello.hs` 程序，将输出管道输送到一个文件，然后在我们的浏览器中打开它：
   
   ```sh
   runghc hello.hs > hello.html
   firefox hello.html
   ```

它应该在页面上显示 `Hello, world!`，在页面标题上显示 `Hello title`。

</details>


---

## 缩进

您可能会问 Haskell 如何知道一个定义是完整的？答案是：Haskell 使用缩进来知道何时应该将事物组合在一起。

Haskell 中的缩进可能有点棘手，但总的来说：应该成为某个表达式一部分的代码应该比该表达式的开头缩进得更远。

我们知道这两个定义是分开的，因为第二个定义没有比第一个定义缩进得更远。


### 缩进技巧

1.  为缩进选择特定数量的空格（2 个空格、4 个空格等）并坚持使用。始终使用空格而不是制表符。
2.  任何时候都不要缩进超过一次。
3.  如有疑问，请根据需要换行并缩进一次。

这里有几个例子：

```hs
main =
    putStrLn "Hello, world!"
```

或者：

```hs
main =
    putStrLn
        (wrapHtml "Hello, world!")
```

__避免以下样式__，它们使用多个缩进步骤，或者完全无视缩进步骤：

```hs
main = putStrLn
        (wrapHtml "Hello, world!")
```

```hs
main = putStrLn
                (wrapHtml "Hello, world!")
```
````