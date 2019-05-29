---
title: Linux 命令 —— 编写Shell脚本
date: 2018-12-04 20:36:52
tags:
  - Linux
---

## 编写第一个 Shell 脚本

1. 编写一个脚本
2. 使脚本可执行
3. 把脚本放到 `Shell` 能找到的地方

<!-- more -->

### 格式

```bash
#!/bin/bash
# This is our first script.
echo 'Hello World!'
```

`#` 代表注释，可以在单独一行，也可以在一行的末尾。`#!` 字符序列是一种特殊的结构叫做 `shebang`。 这个 `shebang` 被用来告诉操作系统将执行此脚本所用的解释器的名字。**每个 `shell` 脚本都应该把这一文本行作为它的第一行**。

### 可执行权限

对于脚本文件，有两个常见的权限设置；权限为 755 的脚本，则每个人都能执行，和权限为 700 的脚本，只有文件所有者能够执行。注意为了能够执行脚本，脚本必须是可读的。

### 脚本文件位置

直接执行脚本文件是会报错的：

```bash
[me@linuxbox ~]$ hello_world
bash: hello_world: command not found
```

如果没有给出可执行程序的明确路径名，那么系统每次都会搜索一系列的目录，来查找此可执行程序。这个 `/bin` 目录就是其中一个系统会自动搜索的目录。这个目录列表被存储在一个名为 `PATH` 的环境变量中。这个 `PATH` 变量包含一个由冒号分隔开的目录列表。

所以如果我们创建了一个 `bin` 目录，并把我们的脚本放在这个目录下，那么这个脚本就应该像其它程序一样开始工作了。如果这个 `PATH` 变量不包含这个目录，我们能够轻松地添加它，通过在我们的 `.bashrc` 文件中包含下面这一行文本：

```bash
export PATH=~/bin:"$PATH"
```

可以通过 “sourcing” .bashrc 文件来使得修改立即生效，这个 `.` 和 `source` 是同样的作用，一个 `shell` 内建命令，用来读取一个指定的 `shell` 命令文件。

## Shell 基本语法

### 变量和常量

**变量的使用**

```bash
# 赋值
foo="yes"
# 使用
echo $foo
```

> 注意如果名字错的话输出空，因为当 `shell` 遇到错误名字（这里是 `fool`）的时候, 它很高兴地创建了变量 `fool` 并且赋给 `fool` 一个空的默认值。

变量名有以下 3 条限制：

- 变量名可由字母数字字符（字母和数字）和下划线字符组成。
- 变量名的第一个字符必须是一个字母或一个下划线。
- 变量名中不允许出现空格和标点符号。

**将命令结果赋值给变量**

```bash
var=`command`

var=$(command)
```

**常量的使用**

`shell` 不能辨别变量和常量；它们大多数情况下是为了方便程序员。一个常用惯例是指定大写字母来表示常量，小写字母表示真正的变量。

**其他使用技巧**

在参数展开过程中，变量名可能被花括号 “{}” 包围着。

```bash
mv $filename ${filename}1
```

如上操作就能在文件名后面加 “1”。

**here documents**

`command << token` 被称为 `here documents`。这里的 `command` 是一个可以接受标准输入的命令名，`token` 是一个用来指示嵌入文本结束的字符串。

`here documents` 很大程度上和 `echo` 一样，除了默认情况下，`here documents` 中的单引号和双引号会失去它们在 `shell` 中的特殊含义。如果我们把重定向操作符从 “<<” 改为 “<<-”，`shell` 会忽略在此 `here document` 中开头的 tab 字符。

### if 分支

`if` 语法如下：

```bash
if commands; then
     commands
[elif commands; then
     commands...]
[else
     commands]
fi
```

#### 退出状态

当命令执行完毕后，命令（包括我们编写的脚本和 shell 函数）会给系统发送一个值，叫做退出状态。 这个值是一个 0 到 255 之间的整数，说明命令执行成功或是失败。按照惯例，一个零值说明成功，其它所有值说明失败。而 `$?` 则是上一条命令执行的退出状态。

#### 测试文件表达式（经常在 `if` 中使用）

| 表达式          | 条件为真则返回 True                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------- |
| file1 -ef file2 | file1 和 file2 拥有相同的索引号（通过硬链接两个文件名指向相同的文件）。                        |
| file1 -nt file2 | file1 新于 file2。                                                                             |
| file1 -ot file2 | file1 早于 file2。                                                                             |
| -b file         | file 存在并且是一个块（设备）文件。                                                            |
| -c file         | file 存在并且是一个字符（设备）文件。                                                          |
| -d file         | file 存在并且是一个目录。                                                                      |
| -e file         | file 存在。                                                                                    |
| -f file         | file 存在并且是一个普通文件。                                                                  |
| -g file         | file 存在并且设置了组 ID。                                                                     |
| -G file         | file 存在并且由有效组 ID 拥有。                                                                |
| -k file         | file 存在并且设置了它的“sticky bit”。                                                          |
| -L file         | file 存在并且是一个符号链接。                                                                  |
| -O file         | file 存在并且由有效用户 ID 拥有。                                                              |
| -p file         | file 存在并且是一个命名管道。                                                                  |
| -r file         | file 存在并且可读（有效用户有可读权限）。                                                      |
| -s file         | file 存在且其长度大于零。                                                                      |
| -S file         | file 存在且是一个网络 socket。                                                                 |
| -t fd f         | d 是一个定向到终端／从终端定向的文件描述符 。 这可以被用来决定是否重定向了标准输入／输出错误。 |
| -u file         | file 存在并且设置了 setuid 位。                                                                |
| -w file         | file 存在并且可写（有效用户拥有可写权限）。                                                    |
| -x file         | file 存在并且可执行（有效用户有执行／搜索权限）。                                              |

#### 字符串表达式

| 表达式                                    | 条件为真则返回 True                                              |
| ----------------------------------------- | ---------------------------------------------------------------- |
| string                                    | string 不为 null。                                               |
| -n string                                 | 字符串 string 的长度大于零。                                     |
| -z string                                 | 字符串 string 的长度为零。                                       |
| string1 = string2 <br> string1 == string2 | string1 和 string2 相同。 单或双等号都可以，不过双等号更受欢迎。 |
| string1 != string2                        | string1 和 string2 不相同。                                      |
| string1 > string2                         | sting1 排列在 string2 之后。                                     |
| string1 < string2                         | string1 排列在 string2 之前。                                    |

> 注意：当与 `test` 一块使用的时候， > 和 < 表达式操作符必须用引号引起来（或者是用反斜杠转义）。如果不这样，它们会被 `shell` 解释为重定向操作符，造成潜在的破坏结果。例如 `[ ab < bc ]` 是会报错的，读者可自行尝试。

#### 测试整数表达式

| 表达式                | 条件为真则返回 True            |
| --------------------- | ------------------------------ |
| integer1 -eq integer2 | integer1 等于 integer2。       |
| integer1 -ne integer2 | integer1 不等于 integer2。     |
| integer1 -le integer2 | integer1 小于或等于 integer2。 |
| integer1 -lt integer2 | integer1 小于 integer2。       |
| integer1 -ge integer2 | integer1 大于或等于 integer2。 |
| integer1 -gt integer2 | integer1 大于 integer2。       |

#### [[ ]]命令

`[[ expression ]]` 类似于 `test`，`expression` 是一个表达式，其计算结果为真或假。这个`[[ ]]`命令非常 相似于 `test` 命令（它支持所有的表达式），但是增加了一个重要的新的字符串表达式：

```bash
string1 =~ regex
```

如果 `string1` 匹配扩展的正则表达式 `regex` 则返回真。

`[[ ]]` 添加的另一个功能是==操作符支持类型匹配，正如路径名展开所做的那样。例如：

```bash
FILE=foo.bar
if [[ $FILE == foo.* ]]; then echo "$FILE matches pattern 'foo.*'";fi
```

#### (())命令（只针对整数）

`(())` 可以非常方便的操作整数，可以正常使用 `>`、`<`、`==` 等很直观的操作。例如 `if ((INT == 0)); then echo "INT is zero."`。

#### 结合表达式

| 操作符 | 测试 | [[ ]] and (( )) |
| ------ | ---- | --------------- |
| AND    | -a   | &&              |
| OR     | -o   | &#124;&#124;    |
| NOT    | !    | !               |

我们也可以对表达式使用圆括号，为的是分组。如果不使用括号，那么否定只应用于第一个表达式，而不是两个组合的表达式。用 `test` 可以这样来编码：

```bash
if [ ! \( $INT -ge $MIN_VAL -a $INT -le $MAX_VAL \) ]; then
    echo "$INT is outside $MIN_VAL to $MAX_VAL."
else
    echo "$INT is in range."
fi
```

> 可以发现圆括号是要转义的！！！

#### 总结

1.`test` 中可用的比较运算符只有 `==` 和 `!=`，两者都是用于字符串比较的，不可用于整数比较，整数比较只能使用 `-eq`, `-gt` 这种形式。无论是字符串比较还是整数比较都千万不要使用大于号小于号。当然，如果你实在想用也是可以的，对于字符串比较可以使用尖括号的转义形式， 如果比较"ab"和"bc"：`[ ab \< bc ]`，结果为真，也就是返回状态为 0。

2.字符串比较时可以把右边的作为一个模式（这是右边的字符串不加双引号的情况下。如果右边的字符串加了双引号，则认为是一个文本字符串。），而不仅仅是一个字符串，比如 `[[ hello == hell? ]]`，结果为真（当然只针对 `[[]]`，普通 `test` 和 `[]` 不支持正则模式比较）。

3.`(( ))` 结构扩展并计算一个算术表达式的值。如果表达式值为 0，会返回 1 或假作为退出状态码。一个非零值的表达式返回一个 0 或真作为退出状态码。这个结构和先前 `test` 命令及 `[]` 结构的讨论刚好相反。

4.`[ ... ]`为 `shell` 命令，所以在其中的表达式应是它的命令行参数，所以串比较操作符 `>` 与 `<` 必须转义，否则就变成 `IO` 改向操作符了。在 `[[` 中 `<` 与 `>` 不需转义；
由于 "[[" 是关键字，不会做命令行扩展，因而相对的语法就稍严格些。例如在[ ... ]中可以用引号括起操作符，因为在做命令行扩展时会去掉这些引号，而在[[ ... ]]则不允许这样做。

### while/until 循环

#### while

语法：

```bash
while commands; do commands; done
```

`break` 命令被用来退出循环，`continue` 命令用来直接执行下一个循环。

#### until

语法：

```bash
until commands; do commands; done
```

### case

语法：

```bash
case word in
    [pattern [| pattern]...) commands ;;]...
esac
```

注意这里的模式（`pattern`）是有格式要求的，可以发现都是以 `)` 结尾！而每一条命令后都是**两个分号**！！

| 模式         | 描述                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| a)           | 若单词为 “a”，则匹配                                                                                                                                    |
| [[:alpha:]]) | 若单词是一个字母字符，则匹配                                                                                                                            |
| ???)         | 若单词只有 3 个字符，则匹配                                                                                                                             |
| \*.txt)      | 若单词以 “.txt” 字符结尾，则匹配                                                                                                                        |
| \*)          | 匹配任意单词。把这个模式做为 `case` 命令的最后一个模式，是一个很好的做法， 可以捕捉到任意一个与先前模式不匹配的数值；也就是说，捕捉到任何可能的无效值。 |

添加的 “;;&” 的语法允许 `case` 语句继续执行下一条测试，而不是简单地终止运行。如下：

```bash
case $REPLY in
    [[:upper:]])    echo "'$REPLY' is upper case." ;;&
    [[:lower:]])    echo "'$REPLY' is lower case." ;;&
    [[:alpha:]])    echo "'$REPLY' is alphabetic." ;;&
esac
```

### 位置参数

`shell` 提供了一个称为位置参数的变量集合，这个集合包含了命令行中所有独立的单词。这些变量按照从 0 到 9 给予命名。

位置参数 `$0` 总会包含命令行中出现的第一个单词，也就是已执行程序的路径名。而 `$1`、`$2`、`$i`……等等都是第 i 个参数。另外 `shell` 还提供了一个名为 `$#`，可以得到命令行参数个数的变量。

> 注意： 实际上通过参数展开方式你可以访问的参数个数多于 9 个。只要指定一个大于 9 的数字，用花括号把该数字括起来就可以。 例如 `${10}`、 `${55}`、`${211}`等等。

**shift 访问多个参数**

执行一次 `shift` 命令，就会导致所有的位置参数“向下移动一个位置”。事实上，用 `shift` 命令也可以处理只有一个参数的情况（除了其值永远不会改变的变量 `$0`）：

```bash
#!/bin/bash
# posit-param2: script to display all arguments
count=1
while [[ $# -gt 0 ]]; do
    echo "Argument $count = $1"
    count=$((count + 1))
    shift
done
```

### for 循环

语法：

```bash
for variable [in words]; do
    commands
done
```

例子：

```bash
# 普通用法
for i in A B C D; do echo $i; done
# 花括号
for i in {A..D}; do echo $i; done
# 路径名展开
for i in distros*.txt; do echo $i; done
```

`C` 语言格式语法，我们熟知的循环：

```bash
for (( expression1; expression2; expression3 )); do
    commands
done
```

## 错误追踪

`bash` 提供了一种名为追踪的方法，这种方法可通过 `-x` 选项和 `set` 命令加上 `-x` 选项两种途径实现。

1.`-x` 选项。给脚本第一行加上 `-x`:

```bash
#!/bin/bash -x
```

2.`set` 命令加上 `-x` 选项,为脚本中的一块选择区域，而不是整个脚本启用追踪。

```bash
set -x # Turn on tracing
if [ $number = 1 ]; then
    echo "Number is equal to 1."
else
    echo "Number is not equal to 1."
fi
set +x # Turn off tracing
```

## 字符串和数字

### 参数展开

#### 基本参数

`$a` 会变成变量 a 所包含的值。简单参数也可能用花括号引起来：`${a}`，虽然这对展开没有影响，但若该变量 a 与其它的文本相邻，可能会把 `shell` 搞糊涂了，例如上文有提到的 `${filename}1` 可以在文件名后加 “1”。通过把数字包裹在花括号中，可以访问大于 9 的位置参数。例如，访问第十一个位置参数可以使用 `${11}`。

#### 管理空变量的展开

- `${parameter:-word}`，若 parameter 没有设置（例如，不存在）或者为空，展开结果是 word 的值。若 parameter 不为空，则展开结果是 parameter 的值。
- `${parameter:=word}`，若 parameter 没有设置或为空，展开结果是 word 的值。另外，**word 的值会赋值给 parameter**。 若 parameter 不为空，展开结果是 parameter 的值。
- `${parameter:?word}`，若 parameter 没有设置或为空，**这种展开导致脚本带有错误退出，并且 word 的内容会发送到标准错误**。若 parameter 不为空，展开结果是 parameter 的值。
- `${parameter:+word}`，若 parameter 没有设置或为空，展开结果为空。若 parameter 不为空， 展开结果是 word 的值会替换掉 parameter 的值；然而，**parameter 的值不会改变**。

#### 返回变量名的参数展开

- `${!prefix*}` 或 `${!prefix@}` 展开会返回以 `prefix` 开头的已有变量名。
- `{{"${#parameter}"}}` 展开成由 `parameter` 所包含的字符串的长度。通常，`parameter` 是一个字符串；然而，如果 `parameter` 是 `@` 或者是 `*` 的话，则展开结果是位置参数的个数，例如 `{{"${#@}"}}` 返回位置参数的个数。
- `${parameter:offset}` 或 `${parameter:offset:length}`，这些展开用来从 `parameter` 所包含的字符串中提取一部分字符。提取的字符始于第 `offset` 个字符（从字符串开头算起）直到字符串的末尾，除非指定提取的长度。
- `${parameter#pattern}`、`${parameter##pattern}`，这些展开会从 `paramter` 所包含的字符串中清除开头一部分文本，这些字符要匹配定义的 `pattern`。`pattern` 是通配符模式，就如那些用在路径名展开中的模式。这两种形式的差异之处是该 `#` 形式清除最短的匹配结果，而该 `##` 模式清除最长的匹配结果。
- `${parameter%pattern}`、`${parameter%%pattern}`，这些展开和上面的 `#` 和 `## 展开一样，除了它们清除的文本从`parameter` 所包含字符串的末尾开始，而不是开头。
- `${parameter/pattern/string}`、`${parameter//pattern/string}`、`${parameter/#pattern/string}`、`${parameter/%pattern/string}`，这种形式的展开对 `parameter` 的内容执行查找和替换操作。如果找到了匹配通配符 `pattern` 的文本，则用 `string` 的内容替换它。在正常形式下，只有第一个匹配项会被替换掉。在该 `//` 形式下，所有的匹配项都会被替换掉。该 `/#` 要求匹配项出现在字符串的开头，而 `/%` 要求匹配项出现在字符串的末尾。`/string` 可能会省略掉，这样会导致删除匹配的文本

### 大小写转换

1. `declare -u/-l variable` 强制变量 `variable` 总是为大写（-u）或小写（-l）。
2. 参数展开大小写转换。

| 格式             | 结果                                                          |
| ---------------- | ------------------------------------------------------------- |
| `${parameter,,}` | 把 `parameter` 的值全部展开成小写字母。                       |
| `${parameter,}`  | 仅仅把 `parameter` 的第一个字符展开成小写字母。               |
| `${parameter^^}` | 把 `parameter` 的值全部转换成大写字母。                       |
| `${parameter^}`  | 仅仅把 `parameter` 的第一个字符转换成大写字母（首字母大写）。 |

## 数组

### 创建

```bash
# 自动创建
a[1]=foo
# declare 显示声明
declare -a a
```

### 赋值

```bash
# 单个赋值
name[subscript]=value
# 多个值赋值
name=(value1 value2 ...)e
```

### 遍历

```bash
for i in $a; do
  echo $i
done
```

> 建议用引号括起来遍历，这样能正常访问字符串。`for i in "${!foo[@]}";do command;done`。

### 长度

```bash
a[100]=foo
echo ${#a[@]} # number of array elements
# 1，未把前面元素初始化，所以不计入数组长度
echo ${#a[100]} # length of element 100
# 3，第 100 个元素的长度，即 foo 的长度
for i in "${!foo[@]}"; do echo $i; done
# ${!array[*]} 或 ${!array[@]} 遍历时得到的是数组下标
```

### 数组末尾添加元素

`+=` 赋值操作即可，`foo+=(d e f)`。

### 数组排序

使用管道排序：

```bash
#!/bin/bash
a=(f e d c b a)
a_sorted=($(for i in "${a[@]}"; do echo $i; done | sort))
```

### 删除数组

使用 `unset array` 删除数组，而 `unset 'array[index]'` 删除单个元素，注意要**加引号**防止展开操作，且下标是**从 0 开始**数的。

### 关联数组（通常认知的 Map）

关联数组使用字符串而不是整数作为数组索引。 这种功能给出了一种有趣的新方法来管理数据，其实就是一个 `Map`。

```bash
declare -A colors
colors["red"]="#ff0000"
colors["green"]="#00ff00"
colors["blue"]="#0000ff"
```

不同于整数索引的数组，仅仅引用它们就能创建数组，关联数组必须用带有 -A 选项的 declare 命令创建。访问关联数组元素的方式几乎与整数索引数组相同：`echo ${colors["blue"]}`。

## 高级操作

### 组命令和子 shell

`bash` 允许把命令组合在一起。可以通过两种方式完成；要么用一个 `group` 命令，要么用一个子 `shell`。 这里是每种方式的语法示例：

```bash
# group
{ command1; command2; [command3; ...] }
# 子 shell
(command1; command2; [command3;...])
```

组命令用花括号把它的命令包裹起来，而子 `shell` 用括号。值得注意的是，鉴于 `bash` 实现组命令的方式，**花括号与命令之间必须有一个空格，并且最后一个命令必须用一个分号或者一个换行符终止**。

> 一个组命令在当前 `shell` 中执行它的所有命令，而一个子 `shell`（顾名思义）在当前 `shell` 的一个 子副本中执行它的命令。这意味着运行环境被复制给了一个新的 `shell` 实例。当这个子 `shell` 退出时，环境副本会消失，所以在子 `shell` 环境（包括变量赋值）中的任何更改也会消失。因此，在大多数情况下，除非脚本要求一个子 `shell`，组命令比子 `shell` 更受欢迎。组命令运行很快并且占用的内存也少。

### 进程替换

问题：

```bash
echo "foo" | read
echo $REPLY
```

该 `REPLY` 变量的内容总是为空，是因为这个 `read` 命令在一个子 `shell` 中执行，所以当该子 `shell` 终止的时候，它的 `REPLY` 副本会被毁掉。因为**管道线中的命令总是在子 `shell` 中执行**，任何给变量赋值的命令都会遭遇这样的问题。幸运地是，`shell` 提供了一种奇异的展开方式，叫做进程替换，它可以用来解决这种麻烦。进程替换有两种表达方式：

一种适用于产生标准输出的进程：

```bash
< (list)
```

另一种适用于接受标准输入的进程：

```bash
> (list)
```

例如解决上述提出的问题：

```bash
read < <(echo "foo")
echo $REPLY
```

> 注意两个 `<` 之间是有空格，两个连在一起则是 [here documents](#变量和常量)。

### 陷阱（trap）

语法：

```bash
trap argument signal [signal...]
```

这里的 `argument` 是一个字符串，它被读取并被当作一个命令，`signal` 是一个信号的说明，它会触发执行所要解释的命令。这里的 `argument` 可以是简单的一条命令，也可以是一个**函数**。

### 异步执行

#### wait

`wait` 命令导致一个父脚本暂停运行，直到一个特定的进程（例如，子脚本）运行结束。语法为 `wait pid`，其中 `pid` 为特定进程的 id。
