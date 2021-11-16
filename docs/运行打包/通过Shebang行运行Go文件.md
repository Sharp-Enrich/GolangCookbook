## 问题

我如何在Go文件的顶部添加一个**#!行**[^1]，以便直接运行它，就像它是一个解释脚本一样

## 解决方案

按照Bash脚本的形式，直观上Go脚本应该要像下面这样写：
```go title="fail.go"
#!/usr/local/bin/go run
package main

import "fmt"

func main() {
    fmt.Println("你好，世界!")
}
```
想象中运行它应该会在终端中输出`你好，世界！`。但是Go语言规范不允许将`#`作为注释，真正运行这个脚本会出现下面的语法错误：
```text
$ ./fail.go
fail.go:1:1: illegal character U+0023 '#'
```
为了使其工作，Go规范必须进行修改，以接受#作为第一行注释。然而，围棋的作者Rob Pike非常固执己见。当(反复)要求使用此功能时，他[说](https://groups.google.com/d/msg/golang-nuts/iGHWoUQFHjg/_dbLKomrPmUJ):
> I have never said it cannot be done. I have always said it should not be done, and I have explained why.

> “Useful” is not an argument for a feature. All features are useful; otherwise they would not be features. The issue is whether the feature justifies its cost. That is a judgement call, and my judgement is, no. Running compilers and linkers, doing megabytes of I/O, and creating temporary binary files is not a justifiable expense for running a small program. For large programs, the amortization is even more in favor of not doing this.

> I am firmly against adding a feature to Go that encourages abuse of resources.

> If you want the feature, use gorun or an equivalent wrapper program. That’s what it’s for. I think believe \[sic\] gorun is a mistake, but I’m not stopping you from installing it and using it as you see fit.

### gorun

What’s this gorun that has Rob up in arms? The Ubuntu Linux distribution produced the gorun utility to solve the shebang issue. As listed in the homepage, gorun will:

+ write files under a safe directory in `$TMPDIR `(or `/tmp`), so that the actual script location isn’t touched (may be read-only)
+ avoid races between parallel executions of the same file
+ automatically clean up old compiled files that remain unused for some time (without races)
+ replace the process rather than using a child
+ pass arguments to the compiled application properly
+ handle well `GOROOT`, `GOROOT_FINAL` and the location of the toolchain

Importantly, gorun will also strip the shebang line from the executed file before running it via `go run`.

Because gorun is a separate tool, and not included in the Go distribution, it must first be installed via `go get launchpad.net/gorun`. Once it’s installed, you can make use of it quite simply:

```go
#!/usr/bin/gorun

package main

func main() {
    println("Hello world!")
}
```

This file is now executable on the command line (after setting the execution bit via chmod +x, of course). However, it’s also technically invalid Go. This means you can’t compile the same code via go build that you would run with gorun.

As we said earlier, this solution also requires that your users install gorun, making it less portable than ideal.

### 通过Bash模仿Shebang

In UNIX systems, the shebang line is handled by the family of `exec` functions inside the fundamental libc. The `exec` functions use what’s referred to as magic numbers in the first bits of a file to determine what kind of executable it is. The characters `#!` are simply hard coded as a magic number that corresponds to being run as a script.

But we can make clever use of the fact that `//` is both a comment in Go, and interpreted as a valid part of a file path in Bash, in order to trick Bash into running the file for us instead of exec. This isn’t technically a shebang line, but serves the same purpose quite nicely. The resulting file looks like such:

```go
//usr/bin/go run $0 $@; exit $?
package main

func main() {
    println("Hello world!")
}
```
This file is both a valid Bash script and valid Go source code. When Bash runs this file, it executes `/usr/bin/go run $0 $@`, and then immediately exits. If the file were named “/path/to/foo.go”, and we ran it like `./foo.go bar baz`, then Bash would execute` /usr/bin/go run ./foo.go "bar" "baz"`.

At this point, the file is parsed by `go run`, which ignores the first line as a comment, and dutifully compiles and executes the rest of the file.

This trick doesn’t require a separately installed command, and is relatively portable. However, all is not roses:

+ The file must end in `.go`, since `go run` will happily ignore it otherwise.
+ This only works when executed via a shell. It won’t work if executed directly by other programs.

### 权衡取舍

Sadly, unless Rob changes his mind and modifies the Go spec, there is no one-size-fits-all solution in sight. Both of the recipes above have their drawbacks, and choosing which one to use is entirely about your specific needs and tastes.



[^1]: 在计算领域中，Shebang（也称为Hashbang）是一个由井号和叹号构成的字符序列#!，其出现在文本文件的第一行的前两个字符。 在文件中存在Shebang的情况下，类Unix操作系统的程序加载器会分析Shebang后的内容，将这些内容作为解释器指令，并调用该指令，并将载有Shebang的文件路径作为该解释器的参数。例如，以指令#!/bin/sh开头的文件在执行时会实际调用/bin/sh程序（通常是Bourne shell或兼容的shell，例如bash、dash等）来执行。这行内容也是shell脚本的标准起始行。更多相关信息请访问[维基百科-Shebang](https://www.wikiwand.com/zh-sg/Shebang)了解。

