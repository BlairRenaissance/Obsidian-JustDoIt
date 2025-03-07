
> 关键词：CLI / shell / bash 

## CLI

命令行界面（英语：command-line interface，缩写：CLI）是**在图形用户界面得到普及之前使用最为广泛的用户界面**，它通常不支持鼠标，用户通过键盘输入指令，计算机接收到指令后，予以执行。 也有人称之为字符用户界面（CUI）。

用户在CLI中输入命令，Shell负责解释和执行这些命令，并将结果返回给用户。CLI和Shell的结合使得用户可以通过命令行来操作计算机系统。

## What is the shell

Unix / Linux shell 是指一种命令行界面，它允许用户输入命令来执行操作，是 Unix 和类 Unix 操作系统（如 Linux 和 macOS）的标准用户界面。Unix shell 可以执行各种命令，如文件操作、进程管理、网络通信等。Unix shell 还提供了一些编程特性，如变量、控制结构（如循环和条件语句）、函数等。

Unix shell 的种类有很多，最常见的有 Bourne shell（sh）、Bourne Again shell（bash）、Korn shell（ksh）、C shell（csh）、zsh 等。不同的 shell 可能在语法、功能和可用性上有所不同。例如，在 Unix 和 Linux 系统中，你可以通过在终端输入 `bash` 来启动 bash shell，然后你可以输入各种命令，如 `ls`，`cd`，`echo`，`sed` 等，来执行相应的操作。

**prompt** 是CLI的一个特性，它在等待用户输入命令时显示一些信息。这些信息通常包括当前用户的用户名、主机名、当前工作目录等，有时也可能包括其他信息，如当前的系统时间、加载情况等。例如，在Unix或Linux系统中，你可能会看到这样的prompt：`[username@hostname ~]$`。在这个例子中，`username`是当前用户的用户名，`hostname`是当前主机的名称，`~`表示当前用户的主目录，`$`是提示符，表示系统正在等待用户输入命令。


## Configuring the Shell

在Unix和Linux系统中，"rc"在配置文件名中的含义源于"run commands" (有时候也说是 "run configuration")。这是一种约定，**用于命名包含启动新shell时运行的命令的脚本文件。** 例如，`~/.zshrc`文件就是Zsh shell启动时会运行的脚本，`~/.bashrc`文件就是bash shell启动时会运行的脚本。这个文件通常包含一些设置环境变量、定义别名、设置shell选项等命令，用于定制用户的shell环境。这种命名约定并不仅限于shell，许多其他的程序（如vim、screen等）也使用以"rc"结尾的文件名来命名它们的配置文件。

Unix 和 Linux 系统的常见惯例是，有一个用于所有用户的 'global' configuration file，以及一个用户主目录中的 'user' configuration file，用户可以编辑该文件以自己个性化设置。The `zsh` shell uses a _~/.zshrc_ file for per-user configuration and _/etc/zsh/zshrc for global configuration.


## Using the shell

### 常用指令

1. `ls`：列出目录中的文件和子目录
    
2. `cd`：改变当前工作目录。
    
3. `pwd`：打印当前工作目录。
    
4. `cp`：复制文件或目录。
    
5. `mv`：移动或重命名文件或目录。
    
6. `rm`：删除文件或目录。
    
7. `cat`：查看文件内容。
    
8. `less`：分页查看文件内容。
    
9. `grep`：在文件中搜索文本。
    
10. `man`：查看命令的帮助手册。
    
11. `chmod`：改变文件或目录的权限。
    
12. `chown`：改变文件或目录的所有者。
    
13. `mkdir`：创建新目录。
    
14. `rmdir`：删除空目录。
    
15. `touch`：创建新文件。
    
16. `echo`：打印文本。
    
17. `history`：查看命令历史。
    
18. `exit`：退出当前 shell。
    
19. `sudo`：以超级用户权限运行命令。
    
20. `ssh`：远程登录到另一台计算机。
    
21. `scp`：在两台计算机之间复制文件。
    
22. `tar`：压缩或解压文件。
    
23. `find`：在文件系统中查找文件。
    
24. `ps`：查看当前进程。
    
25. `kill`：终止进程。
    
26. `top`：查看系统资源使用情况。
    
27. `df`：查看磁盘空间使用情况。
    
28. `du`：查看目录空间使用情况。
    
29. `ln`：创建链接。
    
30. `wget`：从网络下载文件。
    
31. `curl`：发送 HTTP 请求。
    
32. `ping`：测试网络连接。
    
33. `ifconfig`：查看或配置网络接口。
    
34. `netstat`：查看网络连接和统计信息。
    
35. `traceroute`：显示数据包到达目标所经过的路径。
    
36. `sed` ：流编辑器，一次处理一行内容。
    
37. `apropos`：查找与关键字相关的命令。
    
38. `alias`：创建命令的别名。
    
39. `unalias`：删除命令的别名。
    
40. `clear`：清除终端屏幕。


这些指令在 Unix/Linux 系统中非常常用，可以帮助用户完成各种任务。

### 常用符号

在命令行中，有许多特殊的符号和字符，它们有特定的含义。以下是一些常用的符号：

1. `|`  管道符号
	将左边的命令的输出作为右边命令的输入。
	`ls | grep ".txt"` 会列出当前目录下的所有文本文件。
    
2. `>`  重定向符号
	将命令的输出写入到文件中。
	`echo "Hello, World" > hello.txt` 会将 "Hello, World" 写入到 hello.txt 文件中。
    
3. `>>`  追加重定向符号
	将命令的输出追加到文件中。
	`echo "Hello, World" >> hello.txt` 会将 "Hello, World" 追加到 hello.txt 文件中。
    
4. `<`  输入重定向符号
	将文件的内容作为命令的输入。
	`wc -l < hello.txt` 会统计 hello.txt 文件中的行数。
    
5. `;`  分号
	允许在同一行中运行多个命令。
	`echo "Hello, World"; ls` 会先打印 "Hello, World"，然后列出当前目录下的所有文件。
    
6. `&`  后台运行符号
	允许在后台运行命令。
	`sleep 10 &` 会让 sleep 命令在后台运行，不会阻塞终端。
    
7. `&&`  逻辑与符号
	只有在前面的命令成功执行后，才会执行后面的命令。
	`ls && echo "Success"` 会先列出当前目录下的所有文件，然后打印 "Success"。
    
8. `||`  逻辑或符号
	只有在前面的命令失败后，才会执行后面的命令。
	`ls || echo "Failed"` 会先尝试列出当前目录下的所有文件，如果失败（例如，当前目录不存在），则打印 "Failed"。
    
9. `*`  通配符
	匹配任意0个或多个字符。
	`ls *.txt` 会列出当前目录下所有以 .txt 结尾的文件。
    
10. `?`  通配符
	匹配任意单个字符。
	`ls ?.txt` 会列出当前目录下所有以任意单个字符开头，以 .txt 结尾的文件。
    
11. `[]`  字符集
	匹配括号内的任意单个字符。
	`ls [abc].txt` 会列出当前目录下所有以 a、b 或 c 开头，以 .txt 结尾的文件。
    
12. `{}`  大括号
	用于批量处理。
	`touch file{1..10}.txt` 会创建 10 个名为 file1.txt 到 file10.txt 的文件。
    
13. `#`  注释符号
	用于在命令行中添加注释。
	`# This is a comment` 是一个注释。
    
14. `$`  变量符号
	用于引用变量的值。
	`echo $HOME` 会打印当前用户的主目录。
    
15. `()`  子shell
	用于创建一个新的 shell 来执行命令。
	`(cd /tmp; ls)` 会先切换到 /tmp 目录，然后列出该目录下的所有文件。
    
16. `''`  单引号
	用于禁止变量替换。
	`echo '$HOME'` 会打印 `$HOME`，而不是主目录的路径。
    
17. `""`  双引号
	用于允许变量替换。
	`echo "$HOME"` 会打印主目录的路径。
    
18. `\`  转义符号
	用于转义特殊字符。
	`echo "Hello\nWorld"` 会打印 "Hello" 和 "World" 在两行。
    
19. `~`  主目录符号
	表示当前用户的主目录。
	`cd ~` 会切换到当前用户的主目录。
    
20. `.`  当前目录符号
	表示当前目录。
	`cd .` 会保持在当前目录。
    
21. `..`  父目录符号
	表示当前目录的父目录。
	`cd ..` 会切换到当前目录的父目录。

### 详解

**sed**

`sed -i -e 's/foo/bar/' filename` 将所有的 "foo" 替换为 "bar"。
- `-i` 选项是 sed 的一个选项，它允许直接修改文件。默认情况下，sed 会将处理结果输出到标准输出，而不会修改原始文件。-i 选项会直接修改原始文件。
- `-e` 选项是 sed 的一个选项，它允许在命令行上指定多个编辑命令。每个编辑命令都需要使用 -e 选项来指定。`
- `'s/foo/bar/'` 是编辑命令，s 是 sed 的替换命令。


# 命令练习

## Game

Game Page : https://overthewire.org/wargames/bandit/bandit0.html
Ubuntu shell command : https://manpages.ubuntu.com/manpages/noble/en/man1/cp.1.html

## level 0

```
// ssh username@host -p(port)
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

```
$ cat readme
ZjLjTmM6FvvyRnrb2rfNWOZOTa6ip5If
```

## level 1

Level Goal : The password for the next level is stored in a file called **-** located in the home directory.

在Linux中含有特殊字符的文件名可能和shell的一些语法向冲突，比如“-”，shell就认定其之后的内容为参数。所以我们要通过./-表示文件来消除这种歧义。

```
$ cat ./-
263JGJPfgU6LtdEvgfWU1XP5yac29mFx
```

## level 2

Level Goal : The password for the next level is stored in a file called **spaces in this filename** located in the home directory.

文件名中含有空格可以用反斜杠+空格表示。

```
$ cat spaces\ in\ this\ filename
MNk8KNH3Usiio41PRUEoDFPqfxLPlSmx
```

## level 3

Level Goal : The password for the next level is stored in a hidden file in the **inhere** directory.

要列出当前目录中的所有文件，包括隐藏文件，可以使用 `ls` 命令加上 `-a` 选项。

```
$ ls -a
.  ..  ...Hiding-From-You

$ cat /...Hiding-From-You
2WmrDFRmJIq3IPxneAaMGhap0pFhF3NJ
```

## level 4

Level Goal : The password for the next level is stored in the only human-readable file in the **inhere** directory. Tip: if your terminal is messed up, try the “reset” command.

当使用 `file ./*` 命令时，它会检查当前目录下的所有文件和子目录，并显示每个文件或子目录的类型。

```
$ file ./*
./-file00: data 
./-file01: data
./-file02: data
./-file03: data
./-file04: data
./-file05: data
./-file06: data
./-file07: ASCII text
./-file08: data
./-file09: data

$ cat ./-file07
4oQYVPkxZOOEOO5pTW81FB8j8lxXGUQw
```

#### ==`file`==


## level 5

Level Goal : The password for the next level is stored in a file somewhere under the **inhere** directory and has all of the following properties:
- human-readable
- 1033 bytes in size
- not executable

```
$ find . -type f -size 1033c
./maybehere07/.file2

$ cat ./maybehere07/.file2
HWasnPhtq9AVKe0dmk45nxy20cvUa6EG
```

#### ==`find`==

1. **路径**：
    - `find [path]`：指定搜索的起始目录。如果省略，默认为当前目录 `.`。 `/` 表示从根目录开始查找，即在整个文件系统中查找。
    
1. **名称相关**：
    - `-name [pattern]`：按名称搜索文件，支持通配符（如 `*`、`?`）。
        `find . -name "*.txt"`
    - `-iname [pattern]`：按名称搜索文件，忽略大小写。
        `find . -iname "*.txt"`
    
3. **类型**：
    - `-type [type]`：按文件类型搜索。
        - `f`：普通文件
        - `d`：目录
        - `l`：符号链接
        - `c`：字符设备
        - `b`：块设备
        - `p`：命名管道（FIFO）
        - `s`：套接字
        `find . -type f`
    
4. **大小**：
    - `-size [n]`：按文件大小搜索。
        - `c`：字节
        - `k`：千字节
        - `M`：兆字节
        - `G`：吉字节
        `find . -size +1M`
    
5. **时间相关**：
    - `-mtime [n]`：按修改时间搜索，`n` 天前。
        - `+n`：大于 `n` 天
        - `-n`：小于 `n` 天
        - `n`：正好 `n` 天
        `find . -mtime -7`
    - `-atime [n]`：按访问时间搜索，`n` 天前。
        `find . -atime +30`
    - `-ctime [n]`：按状态改变时间搜索，`n` 天前。
        `find . -ctime 1`
    
6. **权限相关**：
    - `-perm [mode]`：按文件权限搜索。
        `find . -perm 644`
    
7. **用户和组**：
    - `-user [name]`：按文件属主搜索。
        `find . -user username`
    - `-group [name]`：按文件属组搜索。
        `find . -group groupname`
    
8. **执行操作**：
    - `-exec [command] {} \;`：对搜索结果执行命令，`{}` 代表当前文件。
        `find . -name "*.log" -exec rm {} \;`
    - `-ok [command] {} \;`：类似 `-exec`，但在执行前会提示用户确认。
        `find . -name "*.log" -ok rm {} \;`
    
9. **组合条件**：
    - `-and` 或 `-a`：逻辑与（默认）。
        `find . -name "*.txt" -and -size +1M`
    - `-or` 或 `-o`：逻辑或。
        `find . -name "*.txt" -or -name "*.log"`
    - `-not` 或 `!`：逻辑非。
        `find . -not -name "*.txt"`

## level 6

Level Goal : The password for the next level is stored **somewhere on the server** and has all of the following properties:
- owned by user bandit7
- owned by group bandit6
- 33 bytes in size

```
$ find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
/var/lib/dpkg/info/bandit7.password

$ cat /var/lib/dpkg/info/bandit7.password
morbNTDkSW6jIlUc0ymOdMaLnOlFVAaj
```

1. **`find /`**：
    - `find`：这是一个用于在目录树中查找文件和目录的命令。
    - `/`：表示从根目录开始查找，即在整个文件系统中查找。
    - `find .` ：表示从当前目录开始查找。
1. **`-user bandit7`**：
    - `-user bandit7`：查找文件所有者（用户）是 `bandit7` 的文件。
2. **`-group bandit6`**：
    - `-group bandit6`：查找文件所属组是 `bandit6` 的文件。
3. **`-size 33c`**：
    - `-size 33c`：查找文件大小为 33 字节的文件。`c` 表示字节（characters）。
4. **`2>/dev/null`**：
    - `2>`：重定向标准错误输出（文件描述符 2）。
    - `/dev/null`：这是一个特殊的文件，在 Unix 和类 Unix 操作系统中被称为“空设备”或“位桶”，所有写入它的数据都会被丢弃，读取它时总是返回 EOF（文件结束符）。将标准错误输出重定向到 `/dev/null` 可以忽略查找过程中产生的错误信息（例如权限不足导致的错误）。

### 重定向

> "重定向"（Redirection）是指将命令的输入或输出从默认位置（通常是终端或控制台）重定向到其他位置（如文件或其他命令）。重定向通常用于处理标准输入（stdin）、标准输出（stdout）和标准错误输出（stderr）。

#### 标准输入、输出和错误输出

- **标准输入（stdin）**：默认情况下，命令从键盘读取输入。文件描述符为 `0`。
- **标准输出（stdout）**：默认情况下，命令将输出发送到终端。文件描述符为 `1`。
- **标准错误输出（stderr）**：默认情况下，命令将错误信息发送到终端。文件描述符为 `2`。
#### 重定向符号

- `>`：将标准输出重定向到文件（覆盖文件内容）。
- `>>`：将标准输出追加到文件（不覆盖文件内容）。
- `<`：将文件内容作为标准输入。
- `2>`：将标准错误输出重定向到文件（覆盖文件内容）。
- `2>>`：将标准错误输出追加到文件（不覆盖文件内容）。
- `&>`：将标准输出和标准错误输出都重定向到文件（覆盖文件内容）。
- `&>>`：将标准输出和标准错误输出都追加到文件（不覆盖文件内容）。
#### 示例

1. **重定向标准输出**：
    `ls > output.txt`
    这将 `ls` 命令的输出重定向到 `output.txt` 文件。如果 `output.txt` 文件存在，它的内容将被覆盖。
    
2. **追加标准输出**：
    `ls >> output.txt`
    这将 `ls` 命令的输出追加到 `output.txt` 文件的末尾。
    
1. **重定向标准错误输出**：
    `ls non_existent_file 2> error.txt`
    这将 `ls` 命令的错误信息重定向到 `error.txt` 文件。
    
4. **重定向标准输出和标准错误输出**：
    `ls non_existent_file &> output_and_error.txt`
    这将 `ls` 命令的输出和错误信息都重定向到 `output_and_error.txt` 文件。
    
5. **将文件内容作为标准输入**：
    `sort < unsorted.txt`
    这将 `unsorted.txt` 文件的内容作为 `sort` 命令的输入。
    
6. **忽略错误信息**：
    `find / -name "file" 2>/dev/null`
    这将 `find` 命令的错误信息重定向到 `/dev/null`，从而忽略错误信息。

## level 7

Level Goal : The password for the next level is stored in the file **data.txt** next to the word **millionth**.

```
$ cat data.txt | grep "millionth"
millionth dfwvzFQi4mU0wfNbFOe9RoWskMLg7eEc
```

## level 8

Level Goal : The password for the next level is stored in the file **data.txt** and is the only line of text that occurs only once.

要在 Linux 中找到文件 `data.txt` 中唯一出现一次的那一行文本，可以组合使用：
1. 使用 `sort` 命令对文件内容进行排序（因为 `uniq` 只能识别相邻的重复行，而在原始文件中，重复的行并不相邻）。
2. 使用 `uniq -u` 命令过滤出只出现一次的行。

```
$ sort data.txt | uniq -u
4CKMh1JI91bUIZZPXDqGanal4xvAg0JM
```

#### ==`uniq`== 

过滤出只出现一次的行，或打印重复行，或打印行的重复次数。命令参数：
- **-u**     Print unique lines.
- **-d**     Print (one copy of) duplicated lines.
- **-c**     Prefix a repetition count and a tab to each output line.  Implies **-u** and **-d**.

## level 9

Level Goal : The password for the next level is stored in the file **data.txt** in one of the few human-readable strings, preceded by several ‘=’ characters.

```
$ strings data.txt | grep ==
}**==========** the
3JprD**==========** passwordi
~fDV3**==========** is
D9**==========** FGUW5ilLVJrxX9kMYMmlN4MgbpfMiqey
```

#### ==`strings`== 

find printable strings in files.

## level 10

Level Goal : The password for the next level is stored in the file **data.txt**, which contains base64 encoded data.

```
$ cat data.txt
VGhlIHBhc3N3b3JkIGlzIGR0UjE3M2ZaS2IwUlJzREZTR3NnMlJXbnBOVmozcVJyCg==

$ cat data.txt | base64 --decode
The password is dtR173fZKb0RRsDFSGsg2RWnpNVj3qRr
```

#### ==`base64`== 

使用base64指令对以base64编码的文本进行解码。命令参数：
- **-d**, **--decode**      decode data
- **-i**, **--ignore-garbage**     when decoding, ignore non-alphabet characters

## level 11

Level Goal : The password for the next level is stored in the file **data.txt**, where all lowercase (a-z) and uppercase (A-Z) letters have been rotated by 13 positions

```
$ cat data.txt
Gur cnffjbeq vf 7k16JArUVv5LxVuJfsSVdbbtaHGlw9D4

$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The password is 7x16WNeHIi5YkIhWsfFIqoognUTyj9Q4
```

#### ==`tr`==

translate characters :
1. **`tr`**: The translate or delete characters command.
2. **`'A-Za-z'`**: The set of characters to be translated (all uppercase and lowercase letters).
3. **`'N-ZA-Mn-za-m'`**: The set of characters to translate to (ROT13 transformation).

## level 12

Level Goal : The password for the next level is stored in the file **data.txt**, which is a hexdump of a file (文件的十六进制转储) that has been repeatedly compressed. For this level it may be useful to create a directory under /tmp in which you can work. Use mkdir with a hard to guess directory name. Or better, use the command `"mktemp -d"`. Then copy the datafile using cp, and rename it using `mv` (read the manpages!)

由于在原目录下无修改权限，因此需要创建一个临时目录，并把 data.txt 复制过去进行处理。

```
$ mktemp -d
/tmp/tmp.mEK7cefagz

$ cp data.txt /tmp/tmp.mEK7cefagz
$ cd /tmp/tmp.mEK7cefagz

/tmp/tmp.mEK7cefagz $ ls
data.txt
```

根据游戏指南，通过 xxd 命令将 hexdump 的 data.txt 文件转出成正常二进制文件。

```
$ xxd -r data.txt data2
$ ls
data2  data.txt
```

已知文件被压缩多次。那么需要先用 `file` 命令查看文件类型，为 gzip 类型压缩文件。发现文件没有正确后缀名，因此先使用 `mv` 命令修改为对应后缀名，再使用 `gzip` 命令解压。`-d` 参数为decompress。

```
$ file data2
data2: gzip compressed data, was "data2.bin", last modified: Thu Sep 19 07:08:15 2024, max compression, from Unix, original size modulo 2^32 574

$ mv data2 data2.gz
$ ls
data2.gz  data.txt

$ gzip -d data2.gz
$ ls
data2  data.txt
```

使用 `bzip2` 命令解压。如果不确定一种压缩格式的后缀名，可以先压缩一下看看。

```
$ file data2
data2: bzip2 compressed data, block size = 900k
$ bzip2 data2
$ ls
data2.bz2  data.txt
$ bzip2 -d data2.bz2
$ mv data2 data2.bz2
$ bzip2 -d data2.bz2
$ ls
data2  data.txt
```

使用 `tar` 命令解压。tar 格式并不强制文件后缀名。`tar` 命令是一个非常强大的工具，用于创建和解压归档文件。

```
$ file data2
data2: POSIX tar archive (GNU)
$ tar -xvf data2
data5.bin
$ file data5.bin
data5.bin: POSIX tar archive (GNU)
$ tar -xv data5.bin // 重点噢
tar: Refusing to read archive contents from terminal (missing -f option?)
$ tar -xvf data5.bin
data6.bin
$ file data6.bin
data6.bin: bzip2 compressed data, block size = 900k
$ bzip2 -d data6.bin
bzip2: Can't guess original name for data6.bin -- using data6.bin.out
$ file data6.bin.out
data6.bin.out: POSIX tar archive (GNU)
$ tar -xvf data6.bin.out
data8.bin
```

```
$ file data8.bin
data8.bin: gzip compressed data, was "data9.bin", last modified: Thu Sep 19 07:08:15 2024, max compression, from Unix, original size modulo 2^32 49

$ gzip -d data8.bin
gzip: data8.bin: unknown suffix -- ignored

$ mv data8.bin data8.gz
$ gzip -d data8.gz
$ ls
data2  data5.bin  data6.bin.out  data8  data.txt

$ file data8
data8: ASCII text

$ cat data8
The password is FO5dwFsc0cbaIiH0h8J2eUks2vdTDwAn
```

#### ==`cp`== 

copy files and directories. 
Copy SOURCE to DEST, or multiple SOURCE(s) to DIRECTORY.

```
cp data.txt "$temp_dir"
```

#### ==`xxd`==

make a hex dump or do the reverse. 
creates a hex dump (十六进制转储) of a given file or standard input.  It can also convert a hex dump back to its original  binary  form.
- **-r , -revert**   Reverse  operation:  convert  (or  patch)  hex dump into binary.

#### ==`mv`==

move or rename files. 
Rename SOURCE to DEST, or move SOURCE(s) to DIRECTORY. 

#### ==`tar`==

GNU **tar** is an archiving program designed to store multiple files  in  a  single  file  (an **archive**),  and to manipulate such archives.
- **-v , --verbose**    显示详细信息。
- **-f , --file**    指定归档文件的名称。
- **-c , --create**    创建一个新的归档文件。
- **-x , --extract**    解压归档文件。
- **-t , --list**    列出归档文件的内容。
- **-z , --gzip**    使用 gzip 解压归档文件。
- **-j , --bzip2**    使用 bzip2 解压归档文件。
- **-J , --xz**    使用 xz 解压归档文件。

**常见用法**
（您猜怎么着？连招的顺序不能错）

- **-cvf**：创建归档文件
```
$ tar -cvf archive.tar file1 file2 
$ tar -czvf archive.tar.gz file1 file2 
$ tar -cjvf archive.tar.bz2 file1 file2 
$ tar -cJvf archive.tar.xz file1 file2
```
- **-xvf**：解压归档文件
```
$ tar -xvf archive.tar 
$ tar -xzvf archive.tar.gz 
$ tar -xjvf archive.tar.bz2 
$ tar -xJvf archive.tar.xz
```
- **-tvf**：列出归档文件内容
```
$ tar -tvf archive.tar 
$ tar -tzvf archive.tar.gz 
$ tar -tjvf archive.tar.bz2 
$ tar -tJvf archive.tar.xz
```
- **-rvf**：追加文件到归档文件
```
$ tar -rvf archive.tar newfile
```

#### ==`gzip`== ==`bzip2`== ==`xz`==

- **gzip、bzip2、xz 等压缩工具**：通常依赖文件名后缀来确定文件格式。如果文件名没有正确的后缀，需要手动更改文件名后缀。
- **tar 归档工具**：不依赖文件名后缀，通过文件内容来确定文件格式，可以直接处理没有正确后缀的文件。

## level 13

Level Goal : The password for the next level is stored in **/etc/bandit_pass/bandit14 and can only be read by user bandit14**. For this level, you don’t get the next password, but you get a private SSH key that can be used to log into the next level. **Note:** **localhost** is a hostname that refers to the machine you are working on.

#### ==`ssh`==

`ssh`（Secure Shell）是一个用于安全远程登录和其他安全网络服务的协议。`ssh` 命令有许多选项，可以用来配置和控制连接的行为。以下是一些常见的 `ssh` 命令选项：

1. **基本选项**：
    - `-l [login_name]`：指定登录用户名。
        `ssh -l username hostname`
    - `-p [port]`：指定远程主机的端口号（默认是 22）。
        `ssh -p 2222 username@hostname`
    
2. **身份验证**：
    - `-i [identity_file]`：指定私钥文件用于身份验证。
        `ssh -i ~/.ssh/id_rsa username@hostname`
    
3. **转发**：
    - `-L [local_port:remote_host:remote_port]`：本地端口转发，将本地端口流量转发到远程主机的指定端口。
        `ssh -L 8080:localhost:80 username@hostname`
    - `-R [remote_port:local_host:local_port]`：远程端口转发，将远程主机的端口流量转发到本地主机的指定端口。
        `ssh -R 8080:localhost:80 username@hostname`
    - `-D [local_port]`：动态端口转发，设置本地端口作为 SOCKS 代理。
        `ssh -D 1080 username@hostname`
    
4. **X11 转发**：
	`X11 转发是一种通过 SSH 隧道将 X11 图形界面应用程序的显示从远程服务器转发到本地计算机的技术。X11 是 X Window System 的简称，它是 Unix 和 Linux 系统上用于图形用户界面的基础技术。`
	
    - `-X`：启用 X11 转发。
        `ssh -X username@hostname`
    - `-Y`：启用受信任的 X11 转发。
        `ssh -Y username@hostname`
    
5. **配置文件**：
    - `-F [configfile]`：指定一个替代的用户配置文件。
        `ssh -F /path/to/configfile username@hostname`
    
6. **调试和详细输出**：
    - `-v`：启用详细模式，可以多次使用（`-vv` 或 `-vvv`）以增加详细级别。
        `ssh -v username@hostname`
    
7. **连接控制**：
    - `-C`：启用压缩。
        `ssh -C username@hostname`
    - `-N`：不执行远程命令，仅用于端口转发。
        `ssh -N -L 8080:localhost:80 username@hostname`
    - `-f`：在后台运行 ssh，会在请求密码或短语后立即返回。
        `ssh -f -N -L 8080:localhost:80 username@hostname`
        
8. **跳板主机**：
    - `-J [user@jump_host]`：通过跳板主机连接到目标主机。
        `ssh -J jump_user@jump_host target_user@target_host`
    
9. **限制带宽**：
    - `-l [limit]`：限制带宽，单位为 Kbps。
        `ssh -l 1000 username@hostname`
    
10. **指定加密算法**：
    - `-c [cipher]`：指定加密算法。
        `ssh -c aes256-ctr username@hostname`
    
11. **指定 MAC 算法**：
    - `-m [mac_spec]`：指定 MAC 算法。
        `ssh -m hmac-sha2-256 username@hostname`
    
12. **指定主机密钥算法**：
    - `-o HostKeyAlgorithms=[algorithms]`：指定主机密钥算法。
        `ssh -o HostKeyAlgorithms=ssh-rsa username@hostname`
    
13. **禁用密码认证**：
    - `-o PasswordAuthentication=no`：禁用密码认证，仅使用密钥认证。
        `ssh -o PasswordAuthentication=no username@hostname`
    
14. **连接超时**：
    - `-o ConnectTimeout=[seconds]`：设置连接超时时间。
        `ssh -o ConnectTimeout=10 username@hostname`
    
15. **使用代理命令**：
    - `-o ProxyCommand=[command]`：通过代理命令连接到目标主机。
        `ssh -o ProxyCommand="nc -X connect -x proxyhost:proxyport %h %p" username@hostname`
    
16. **使用配置文件**：
    - `~/.ssh/config`：可以在用户主目录下的 `.ssh/config` 文件中配置常用的 SSH 选项。然后可以简单地使用 `ssh myserver` 进行连接。
    
```
Host myserver
    HostName hostname
    User username
    Port 2222
    IdentityFile ~/.ssh/id_rsa
```

## level 14



## level 15


## level 16

