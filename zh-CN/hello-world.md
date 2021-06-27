# Hello, World

**[查看本章的所有代码](https://github.com/quii/learn-go-with-tests/tree/master/hello-world)**

按照传统，我们用新语言编写的第一个程序都是 Hello，world。

在之前的[章节](https://github.com/studygolang/learn-go-with-tests/blob/master/zh-CN/install-go.md#Go-环境)中，我们讨论了 Go 死板的文件存放位置。

按照以下路径创建目录 `$GOPATH/src/github.com/{your-user-id}/hello`。

我们假定你使用的是基于 Unix 的系统，你的名字是「bob」并且愿意遵循 Go 关于 `$GOPATH` 的约定（这是最简单的设置方式）。那么你可以执行以下命令 `mkdir -p $GOPATH/src/github.com/bob/hello` 快速创建目录。

对于后面的章节，存放代码的文件夹名称可以按照自己的喜好定义，例如下一节可以用 `$GOPATH/src/github.com/{your-user-id}/integers`。一些读者喜欢将所有代码放到一个文件夹中，比如「learn-go-with-tests/hello」。简而言之，这取决于你如何组织你的文件夹。

在该目录下创建名为 `hello.go` 的文件并编写以下代码，键入 `go run hello.go` 来运行程序。

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world")
}
```

## 它是如何运行的

用 Go 编写程序，你需要定义一个 `main` 包，并在其中定义一个 `main` 函数。包是一种将相关的 Go 代码组合到一起的方式。

`func` 关键字通过函数名和函数体来定义函数。

通过 `import "fmt"` 导入一个包含 `Println` 函数的包，我们用它来打印输出。

## 如何测试

你打算如何测试这个程序？将你「领域」内的代码和外界（会引起副作用）分离开会更好。`fmt.Println` 会产生副作用（打印到标准输出），我们发送的字符串在自己的领域内。[^注1]

[^注1]: 原文: How do you test this? It is good to separate your "domain" code from the outside world \(side-effects\). The `fmt.Println` is a side effect \(printing to stdout\) and the string we send in is our domain.

所以为了更容易测试，我们把这些问题拆分开。

```go
package main

import "fmt"

func Hello() string {
    return "Hello, world"
}

func main() {
    fmt.Println(Hello())
}
```

我们再次使用 `func` 创建了一个新函数，但是这次我们在定义中添加了另一个关键字 `string`。这意味着这个函数将返回一个字符串。

现在创建一个 `hello_test.go` 的新文件，来为 `Hello` 函数编写测试

```go
package main

import "testing"

func TestHello(t *testing.T) {
    got := Hello()
    want := "Hello, world"

    if got != want {
        t.Errorf("got '%q' want '%q'", got, want)
    }
}
```

在解释这个测试之前，让我们先在终端运行 `go test`，它应该通过测试了！为了再次验证，可以尝试改变 `want` 字符串来破坏测试的结果。

注意，你不必在多个测试框架之间进行选择，然后再去理解如何安装它们。你需要的一切都内建在 Go 语言中，语法与你将要编写的其余代码相同。

### 编写测试

编写测试和函数很类似，其中有一些规则

- 程序需要在一个名为 `xxx_test.go` 的文件中编写
- 测试函数的命名必须以单词 `Test` 开始
- 测试函数只接受一个参数 `t *testing.T`

现在这些信息足以让我们明白，类型为 `*testing.T` 的变量 `t` 是你在测试框架中的 hook（钩子），所以当你想让测试失败时可以执行 `t.Fail()` 之类的操作。

我们再讨论一些新的话题：

#### if

Go 的 `if` 语句非常类似于其他编程语言。

#### 声明变量

我们使用 `varName := value` 的语法声明变量，它允许我们在测试中重用一些值使代码更具可读性。

#### `t.Errorf`

我们调用 `t` 的 `Errorf` 方法打印一条消息并使测试失败。`f` 表示格式化，它允许我们构建一个字符串，并将值插入占位符值 `%q` 中。当你的测试失败时，它能够让你清楚是什么错误导致的。

稍后我们将探讨方法和函数之间的区别。

### Go 文档

Go 的另一个高质量特征是文档。通过运行 `godoc -http :8000`，可以在本地启动文档。如果你访问 [localhost:8000/pkg](localhost:8000/pkg)，将看到系统上安装的所有包。

大多数标准库都有优秀的文档和示例。浏览 [http://localhost:8000/pkg/testing/](http://localhost:8000/pkg/testing/) 是非常值得的，去看一下有什么对你有价值的内容。

如果你的 `godoc` 命令不起作用，那么也许你使用的是 Go 的较新版本（1.14或更高），[其中不再包括 `godoc` 命令](https://golang.org/doc/go1.14#godoc)。 你可以运行 `go get golang.org/x/tools/cmd/godoc` 以手动安装。

### Hello, YOU

现在有了测试，就可以安全地迭代我们的软件了。

在上一个示例中，我们在写好代码 *之后* 编写了测试，以便让你学会如何编写测试和声明函数。从此刻起，我们将 *首先编写测试*。

我们的下一个需求是指定问候的接受者。

让我们从在测试中捕获这些需求开始。这是基本的测试驱动开发，可以确保我们的测试用例 *真正* 在测试我们想要的功能。当你回顾编写的测试时，存在一个风险：即使代码没有按照预期工作，测试也可能继续通过。

```go
package main

import "testing"

func TestHello(t *testing.T) {
    got := Hello("Chris")
    want := "Hello, Chris"

    if got != want {
        t.Errorf("got '%q' want '%q'", got, want)
    }
}
```

这时运行 `go test`，你应该会获得一个编译错误

```text
./hello_test.go:6:18: too many arguments in call to Hello
    have (string)
    want ()
```

当使用像 Go 这样的静态类型语言时，*聆听编译器* 是很重要的。编译器理解你的代码应该如何拼接到一起工作，所以你就不必关心这些了。

在这种情况下，编译器告诉你需要怎么做才能继续。我们必须修改函数 `Hello` 来接受一个参数。

修改 `Hello` 函数以接受字符串类型的参数

```go
func Hello(name string) string {
    return "Hello, world"
}
```

如果你尝试再次运行测试，`main.go` 会编译失败，因为你没有传递参数。传入参数「world」让它通过。

```go
func main() {
    fmt.Println(Hello("world"))
}
```

现在，当你运行测试时，你应该看到类似的内容

```text
hello_test.go:10: got 'Hello, world' want 'Hello, Chris''
```

我们最终得到了一个可编译的程序，但是根据测试它并没有达到我们的要求。

为了使测试通过，我们使用 `name` 参数并用 `Hello，` 字符串连接它，

```go
func Hello(name string) string {
    return "Hello, " + name
}
```

现在再运行测试应该就通过了。通常作为 TDD 周期的一部分，我们该着手 *重构* 了。

### 关于版本控制的一点说明

此时，如果你正在使用版本控制（你应该这样做！）我将按原样 **提交** 代码。因为我们拥有一个基于测试的可用版本。

不过我不会推送到主分支上，因为我下一步计划重构。现在提交很合适，你总是可以在重构中陷入混乱时回到这个可用版本。

这里没有太多可重构的，但我们可以介绍一下另一种语言特性 *常量*。

### 常量

通常我们这样定义一个常量

```go
const englishHelloPrefix = "Hello, "
```

现在我们可以重构代码

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
    return englishHelloPrefix + name
}
```

重构之后，重新测试以确保程序无误。

常量应该可以提高应用程序的性能，它避免了每次使用 `Hello` 时创建 `"Hello, "` 字符串实例。

显然，对于这个例子来说，性能提升是微不足道的！但是创建常量的价值是可以快速理解值的含义，有时还可以帮助提高性能。

## 再次回到 Hello, world

下一个需求是当我们的函数用空字符串调用时，它默认为打印 "Hello, World" 而不是 "Hello, "

首先编写一个新的失败测试

```go
func TestHello(t *testing.T) {

    t.Run("saying hello to people", func(t *testing.T) {
        got := Hello("Chris")
        want := "Hello, Chris"

        if got != want {
            t.Errorf("got '%q' want '%q'", got, want)
        }
    })

    t.Run("say hello world when an empty string is supplied", func(t *testing.T) {
        got := Hello("")
        want := "Hello, World"

        if got != want {
            t.Errorf("got '%q' want '%q'", got, want)
        }
    })

}
```

这里我们将介绍测试库中的另一个工具 -- 子测试。有时，对一个「事情」进行分组测试，然后再对不同场景进行子测试非常有效。

这种方法的好处是，你可以建立在其他测试中也能够使用的共享代码。

当我们检查信息是否符合预期时，会有重复的代码。

重构不 *仅仅* 是针对程序的代码！

重要的是，你的测试 *清楚地说明* 了代码需要做什么。

我们可以并且应该重构我们的测试。

```go
func TestHello(t *testing.T) {

    assertCorrectMessage := func(t *testing.T, got, want string) {
        t.Helper()
        if got != want {
            t.Errorf("got '%q' want '%q'", got, want)
        }
    }

    t.Run("saying hello to people", func(t *testing.T) {
        got := Hello("Chris")
        want := "Hello, Chris"
        assertCorrectMessage(t, got, want)
    })

    t.Run("empty string defaults to 'world'", func(t *testing.T) {
        got := Hello("")
        want := "Hello, World"
        assertCorrectMessage(t, got, want)
    })

}
```

我们在这里做了什么？

我们将断言重构为函数。这减少了重复并且提高了测试的可读性。在 Go 中，你可以在其他函数中声明函数并将它们分配给变量。你可以像调用普通函数一样调用它们。我们需要传入 `t *testing.T`，这样我们就可以在需要的时候令测试代码失败。

`t.Helper()` 需要告诉测试套件这个方法是辅助函数（helper）。通过这样做，当测试失败时所报告的行号将在函数调用中而不是在辅助函数内部。这将帮助其他开发人员更容易地跟踪问题。如果你仍然不理解，请注释掉它，使测试失败并观察测试输出。

现在我们有了一个很好的失败测试，让我们使用 `if` 修复代码。

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
    if name == "" {
        name = "World"
    }
    return englishHelloPrefix + name
}
```

如果我们运行测试，应该看到它满足了新的要求，并且我们没有意外地破坏其他功能。

### 回到版本控制

现在我们对代码很满意，我将修改之前的提交，所以我们只提交认为好的版本及其测试。

### 规律

让我们再次回顾一下这个周期

- 编写一个测试
- 让编译通过
- 运行测试，查看失败原因并检查错误消息是很有意义的
- 编写足够的代码以使测试通过
- 重构

从表面上看可能很乏味，但坚持这种反馈循环非常重要。

它不仅确保你有 *相关的测试*，还可以确保你通过重构测试的安全性来 *设计优秀的软件*。

查看测试失败是一个重要的检查手段，因为它还可以让你看到错误信息。作为一名开发人员，如果测试失败时不能清楚地说明问题所在，那么使用这个代码库可能会非常困难。

通过确保你的测试的 *快速*，并设置你的工具，可以使运行测试足够简单，你在编写代码时就可以进入流畅的状态。

如果不写测试，你提交的时候通过运行软件来手动检查你的代码，这会打破你的流畅状态，而且你任何时候都无法将自己从这种状态中拯救出来，尤其是从长远来看。

## 继续前进！更多需求

天呐，我们有更多的需求了。现在需要支持第二个参数，指定问候的语言。如果一种不能识别的语言被传进来，就默认为英语。

通过 TDD 轻松实现这一功能，我们是有信心的！

为使用西班牙语的用户编写测试，将其添加到现有的测试用例中。

```go
    t.Run("in Spanish", func(t *testing.T) {
        got := Hello("Elodie", "Spanish")
        want := "Hola, Elodie"
        assertCorrectMessage(t, got, want)
    })
```

记住不要欺骗自己！*先编写测试*。当你尝试运行测试时，编译器 *应该* 会出错，因为你用两个参数而不是一个来调用 `Hello`。

```text
./hello_test.go:27:19: too many arguments in call to Hello
    have (string, string)
    want (string)
```

通过向 `Hello` 添加另一个字符串参数来解决编译问题

```go
func Hello(name string, language string) string {
    if name == "" {
        name = "World"
    }
    return englishHelloPrefix + name
}
```

当你尝试再次运行测试时，它会报错在其他测试和 `hello.go` 中没有传递足够的参数给 `Hello` 函数

```text
./hello.go:15:19: not enough arguments in call to Hello
    have (string)
    want (string, string)
```

通过传递空字符串来解决它们。现在，除了我们的新场景外，你的所有测试都应该编译并通过

```text
hello_test.go:29: got 'Hello, Elodie' want 'Hola, Elodie'
```

这里我们可以使用 `if` 检查语言是否是「西班牙语」，如果是就修改信息

```go
func Hello(name string, language string) string {
    if name == "" {
        name = "World"
    }

    if language == "Spanish" {
        return "Hola, " + name
    }

    return englishHelloPrefix + name
}
```

测试现在应该通过了。

现在是 *重构* 的时候了。你应该在代码中看出了一些问题，其中有一些重复的「魔术」字符串。自己尝试重构它，每次更改都要重新运行测试，以确保重构不会破坏任何内容。

```go
const spanish = "Spanish"
const helloPrefix = "Hello, "
const spanishHelloPrefix = "Hola, "

func Hello(name string, language string) string {
    if name == "" {
        name = "World"
    }

    if language == spanish {
        return spanishHelloPrefix + name
    }

    return englishHelloPrefix + name
}
```

### 法语

- 编写一个测试，断言如果你传递 `"French"` 你会得到 `"Bonjour, "`
- 看到它失败，检查错误信息是否容易理解
- 在代码中进行最小的合理更改

你可能写了一些看起来大致如此的东西

```go
func Hello(name string, language string) string {
    if name == "" {
        name = "World"
    }

    if language == spanish {
        return spanishHelloPrefix + name
    }

    if language == french {
        return frenchHelloPrefix + name
    }

    return englishHelloPrefix + name
}
```

### `switch`

当你有很多 `if` 语句检查一个特定的值时，通常使用 `switch` 语句来代替。如果我们希望稍后添加更多的语言支持，我们可以使用 `switch` 来重构代码，使代码更易于阅读和扩展。

```go
func Hello(name string, language string) string {
    if name == "" {
        name = "World"
    }

    prefix := englishHelloPrefix

    switch language {
    case french:
        prefix = frenchHelloPrefix
    case spanish:
        prefix = spanishHelloPrefix
    }

    return prefix + name
}
```

编写一个测试，添加用你选择的语言写的问候，你应该可以看到扩展这个 *神奇* 的函数是多么简单。

### 最后一次重构？

你可能会抱怨说也许我们的函数正在变得很臃肿。对此最简单的重构是将一些功能提取到另一个函数中。

```go
func Hello(name string, language string) string {
    if name == "" {
        name = "World"
    }

    return greetingPrefix(language) + name
}

func greetingPrefix(language string) (prefix string) {
    switch language {
    case french:
        prefix = frenchHelloPrefix
    case spanish:
        prefix = spanishHelloPrefix
    default:
        prefix = englishHelloPrefix
    }
    return
}
```

一些新的概念：

- 在我们的函数签名中，我们使用了 *命名返回值*（`prefix string`）。
- 这将在你的函数中创建一个名为 `prefix` 的变量。
  - 它将被分配「零」值。这取决于类型，例如 `int` 是 0，对于字符串它是 `""`。
    - 你只需调用 `return` 而不是 `return prefix` 即可返回所设置的值。
  - 这将显示在 Go Doc 中，所以它使你的代码更加清晰。
- 如果没有其他 `case` 语句匹配，将会执行 `default` 分支。
- 函数名称以小写字母开头。在 Go 中，公共函数以大写字母开始，私有函数以小写字母开头。我们不希望我们算法的内部结构暴露给外部，所以我们将这个功能私有化。

## 总结

谁会知道你可以从 `Hello, world` 中学到这么多东西呢？

现在你应该对以下内容有了一定的理解：

### Go 的一些语法

- 编写测试
- 用参数和返回类型声明函数
- `if`，`const`，`switch`
- 声明变量和常量

### TDD 过程以及步骤的重要性

- *编写一个失败的测试，并查看失败信息*，我们知道现在有一个为需求编写的 *相关* 的测试，并且看到它产生了 *易于理解的失败描述*
- 编写最少量的代码使其通过，以获得可以运行的程序
- *然后* 重构，基于我们测试的安全性，以确保我们拥有易于使用的精心编写的代码

在我们的例子中，我们通过小巧易懂的步骤从 `Hello()` 到 `Hello("name")`，到 `Hello("name", "french")`。

与「现实世界」的软件相比，这当然是微不足道的，但原则依然通用。TDD 是一门需要通过开发去实践的技能，通过将问题分解成更小的可测试的组件，你编写软件将会更加轻松。

---

作者：[Chris James](https://dev.to/quii)
译者：[Donng](https://github.com/Donng)
校对：[polaris1119](https://github.com/polaris1119)，[pityonline](https://github.com/pityonline)

本文由 [GCTT](https://github.com/studygolang/GCTT) 原创编译，[Go 中文网](https://studygolang.com/) 荣誉推出
