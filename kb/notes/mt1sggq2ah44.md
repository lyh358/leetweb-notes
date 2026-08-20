# 十、嵌入式平台工具

# 一、git

## 01 git rebase 和 git merge 的区别

git rebase 和 git merge 都是 Git 版本控制系统中用于合并分支的操作，它们的主要区别在于合并方式不同。

1. **git merge**：将一个分支的修改内容合并到另一个分支中，并创建一个新的 commit 记录来记录这个合并操作。当当前分支需要与其他分支合并时，git merge 是最常用的合并方式。例如：

```
git checkout dev      // 切换到dev分支
git merge feature     // 将feature分支合并到dev分支
```

1. **git rebase**：将一个分支的修改内容以 “衍合”（rebase）的方式应用到另一个分支中，即将当前分支的修改内容重新基于目标分支进行一次提交。通过使用 git rebase，可以将分支的提交历史线变得更加清晰、直观。例如：

```
git checkout feature  // 切换到feature分支
git rebase dev        // 将dev分支的修改内容应用到feature分支中
```

总的来说，git merge 和 git rebase 都可以实现分支的合并操作，但是合并结果以及提交历史都有所不同。相对而言，git merge 的合并过程比较简单，但会产生额外的 merge commit；而 git rebase 可以更好地保持提交历史的清晰和整洁，但需要进行分支的重定位，同时也可能引发冲突问题。因此，在使用时需要根据实际情况进行选择，以便达到最佳的版本控制效果。

## 02 切换分支、创建分支

- `git checkout`
- `git branch`

## 03 git 分支冲突如何解决

### 考察场景

考察你是否理解 Git 多人协作时，分支合并可能产生冲突，如何定位和解决。

### 底层实现原理

1. 冲突产生原因：两条分支修改了同一文件的同一行，Git 无法自动合并。
2. Git 会标记冲突区域，用 `<<<<<<<`、`=======`、`>>>>>>>` 标记不同分支的内容。
3. 解决冲突一般步骤：手动修改冲突文件 → 保存 → `git add` → `git commit`。
4. 可以选择保留某一分支版本，也可以手动合并内容，保证逻辑正确。
5. 解决冲突后，Git 会记录为一次合并提交，确保历史完整。

### 代码说明

```
# 假设在 feature 分支，拉取 main 分支变更
git checkout feature
git pull origin main

# 出现冲突提示
Auto‑merging file.txt
CONFLICT (content): Merge conflict in file.txt
```

### 常见处理策略

1. 保留本地：`git checkout --ours file.txt`
2. 保留远端：`git checkout --theirs file.txt`
3. 手动合并：直接编辑冲突标记内容
4. 工具辅助：如 VSCode、GitKraken、Meld 等可视化解决冲突

### 延申知识点

- Rebase 与 Merge 时冲突的差异
- 如何用 `git mergetool` 解决冲突

# 三、ROS 机器人操作系统

## 01 ROS1 和 ROS2 的区别

两者最根本的区别，直接影响通信可靠性和扩展性。

- **ROS1 架构**：依赖中心节点（Master）实现全局信息管理，所有节点（Node）必须先向 Master 注册，才能发现其他节点并建立通信（如 Topic/Service）。
  - 缺陷：Master 是 “单点故障” —— 若 Master 崩溃，整个系统通信中断；无法支持无中心的分布式场景（如多机器人协作）。
- **ROS2 架构**：移除中心节点，采用分布式架构，基于 DDS（Data Distribution Service，数据分发服务）作为底层通信协议。
  - 节点（Node）通过 DDS 的 “发现机制” 自动发现其他节点，直接建立点对点通信，无需中心协调。
  - 优势：无单点故障，支持多机器人、跨网络（甚至跨设备）通信，稳定性和扩展性大幅提升。

# 四、CMake

## 01 cmake 如何生成动态共享库

```
# 设置项目名称
project(my_library)

# 设置最低要求的CMake版本
cmake_minimum_required(VERSION 3.0)

# 指定源文件目录
set(SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)

# 指定头文件目录
set(INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)

# 将头文件目录添加到编译器的搜索路径中
include_directories(${INCLUDE_DIR})

# 查找源文件
file(GLOB_RECURSE SRC_FILES ${SRC_DIR}/*.cpp)

# 创建动态共享库
add_library(my_library SHARED ${SRC_FILES})

# 设置动态共享库的输出路径
set_target_properties(my_library PROPERTIES LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/build)
```

> 追问 1 so 库可以在运行时被代码调用么
>
> `dlopen` 函数用于在运行时打开一个动态共享库。

## 02 cmake 版本低于 CMakeLists 中的 cmake_min_required 会出现什么问题？

cmake error, 提示需要更高的版本

## 03 cmake 可以进行宏编译，并且在代码里使用么

可以

## 04 cmake 如何搜索动态库或者静态库

- `find_library`

## 05 cmake 编译常用指令

```
cmake
make -j8
make install
```

## 06 常用指令

```
cmake_minimum_required(VERSION 2.5)
project(mymuduo)

# mymuduo最终编译成so动态库，设置动态库的路径，放在根目录的lib文件夹下面
set(LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)

# 设置调试信息 以及 启动C++11语言标准
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=c++11")

# 定义参与编译的源代码文件
aux_source_directory(. SRC_LIST)

# 编译生成动态库mymuduo
add_library(mymuduo SHARED ${SRC_LIST})
```

## 07 如何生成一个动态库

```
add_library(<library_name> SHARED <source_files>)
```

### CMake 常用命令大全

- `cmake_minimum_required` 最低版本号确定
- `project`       项目名称
- `add_subdirectory`  加载子目录下的 CMakeLists
- `set` 一些设置选项  设置路径等
- `include_directories` 头文件搜索路径
- `link_directories`  库文件搜索路径 local/include local/lib
- `add_executable`   指定生成可执行文件
- `target_link_libraries` 执行可执行文件所需要依赖的库

# 五、Shell 脚本编程

## 01 如何写一个自启动 shell 脚本

- **考察场景**

考察你是否理解 Linux 系统如何在启动时自动执行脚本，包括系统服务和用户登录两种方式。

- **底层实现原理**

1. Systemd 系统：通过创建一个 service 单元文件，在启动时自动执行脚本。Systemd 管理服务的依赖、启动顺序和状态。
2. 传统 init /rc.local：在 `/etc/rc.local` 或对应 runlevel 的脚本目录里添加命令，开机时由 init 调用。
3. 用户登录自动执行：可以在用户家目录下 `.bashrc`、`.profile` 或图形界面 `.xinitrc` 中调用脚本。
4. 核心思想是 把脚本注册到系统启动流程或用户登录流程中，由操作系统调用，而不是手动执行。

## 02 Linux 三剑客

- 文本搜索 **grep**
- 文本行处理 **sed**
- 文本列处理 **awk**

Linux 三剑客指的是 grep、sed 和 awk，它们是在 Linux 系统中广泛使用的文本处理工具，可以用于快速、高效地处理文本文件。

1. **grep**：用于在文本文件中查找指定的字符串或模式，并输出匹配行。它支持多种匹配模式、正则表达式和选项参数，可以实现快速的文本搜索。例如：
2. **sed**：用于对文本文件进行流式编辑，即按照行来处理文件中的内容，并可以对文件进行修改、替换等操作。它支持多种正则表达式和命令参数，可以实现复杂的文本处理需求。例如：

```
sed 's/hello/world/g' file.txt  // 将file.txt中所有的"hello"替换为"world"

sed 参数 命令 处理对象
sed -n 5p westos     ##显示第5行
sed -n 3,5p westos     ##显示第3行到第5行
sed -n "3p;5p" westos   ##显示第3行和第5行
sed -n 1,5p westos  ##显示1‑5行
sed -n '5,$p' westos  ##显示第5行到最后一行
sed -n '/^#/p' fstab  ##显示以#开头的行
```

1. **awk**：用于在文本文件中进行基于行和列的数据处理，它支持多种分隔符和函数、逻辑表达式等，可以实现高级的文本数据处理需求。例如：

三剑客通常都是通过命令行使用，可以针对不同的文本处理需求进行组合使用。它们在 Linux 系统中拥有很高的使用率和广泛的应用场景，是 Linux 下文本处理的重要工具。

## 03 常见 Linux 指令

- ls
- mv
- cp
- rm
- cat
- free
- uname
- ps
- top
- kill

## 04 查看指定进程占用的端口号

```
ps aux | grep python
```

## 05 打印某一列并按空格分隔

```
awk '{print $N}' 文件名 | tr '\n' ' '
```

- 查看内存资源 **top**
- 查看进程 **ps**
- 查找文本 **find**

## 06 查看线程堆栈信息

```
pstack <PID>
```

## 07 shell 脚本执行的原理

- **考察场景**

考察你是否理解 shell 脚本不是直接由 CPU 执行，而是由解释器逐行解析执行。

- **底层实现原理**

1. shell 脚本本质是一个文本文件，里面写的是 shell 命令，比如 `cd`、`echo`、`if`、`for`、管道、重定向等。
2. 执行脚本时，系统会根据脚本第一行 shebang，比如 `#!/bin/bash`，找到对应解释器来运行。
3. shell 解释器会逐行读取脚本，进行变量替换、命令解析、通配符展开、管道 / 重定向处理。
4. 如果遇到外部命令，比如 `ls`、`grep`，shell 通常会 fork 子进程，再通过 exec 加载对应程序执行。
5. 所以 shell 脚本的本质是：shell 进程解析脚本逻辑，并按需创建子进程执行外部命令。

- **代码说明**

```
#!/bin/bash

name="linux"
echo "hello $name"

ls /tmp | grep log
```

执行过程可以理解成：

1. 内核看到 `#!/bin/bash`
2. 启动 `/bin/bash` 作为解释器
3. bash 读取脚本内容
4. 解析变量 name
5. 执行 echo
6. 执行 ls 和 grep，并建立管道

外部命令执行大致是：

```
bash
-> fork() 创建子进程
-> 子进程 exec("ls")
-> 父进程等待或继续调度
```

如果直接执行：

```
./test.sh
```

需要脚本有执行权限：

```
chmod +x test.sh
```

如果这样执行：

```
bash test.sh
```

则不依赖脚本自身执行权限，因为是手动把脚本作为参数交给 bash。

- **延申知识点**

`source script.sh` 和 `./script.sh` 的区别

shell 中管道和重定向的底层实现原理

# 八、Makefile

## 01 Makefile 基础编译规则

- **考察场景**

考察你是否理解 Makefile 如何把源文件编译成可执行程序或库，以及基本规则的写法。

- **底层实现原理**

1. **目标（target）**：你想生成的文件，比如 `program` 或 `module.o`。
2. **依赖（dependencies）**：生成目标需要的文件，比如源文件 `.c`、头文件 `.h`。
3. **命令（recipe）**：生成目标的命令，比如 `gcc -c main.c -o main.o`。
4. 当依赖文件发生变化时，Make 会重新执行对应命令生成目标，从而实现 **增量编译**。
5. 可以使用 **模式规则（pattern rule）**、变量（如 `CC`，`CFLAGS`）和自动变量（如 `$@`，`$^`）来简化编写。