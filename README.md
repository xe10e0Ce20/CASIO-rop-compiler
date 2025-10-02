# ROP_Compiler
一款给nx-u16以及nx-u8系列芯片的rop链编译工具，支持gadgets,自定义函数，以及地址标签
这里提供了一些基本的配置文件：verc.ggt, common.spf, common.cpf，以供参考，欢迎大家开发自己的配置文件
verf的gadgets还没整理，希望大佬们能提供
程序可能会有bug，欢迎提出issue，或者在贴吧上留言
如果反响比较好的话会考虑开发VS Code插件
本程序使用一套自己的语法系统，以下是语法介绍。

## 语法说明
### 导入配置
在文件开头增加
```
import <文件名>
```
如：
```
import verc.ggt
```
以导入配置文件，程序会读取配置文件中的gadgets，简单函数和高阶函数

### 函数和gadgets定义
使用
```
def ...
```
来定义函数，语法和配置文件相同，只不过头部加一个def

### 程序块
使用@block.xxx: 开始一个程序块，其中xxx是任意变量名，如：
```
@block.main:
```
别忘了用@blockend 结束程序块
例：
```
@block.main:
    程序......
@blockend
```

### 字节码
字节码没有前缀，直接输入即可，自动忽略大小写及空格
如:
```
0123 abcd
```
可以用x作为占位符，如
```
a821x1xx
```

#### 字节码运算

- 加减运算
字节码可以进行加减运算，直接在中间输入+-运算符即可，最好隔有空格
运算结果的位数为参与运算的字节码中位数最大者
如果为负数则返回补码
如：
```
a821 + 01 - a800  //返回0022
```

- 尖括号的使用
可以使用尖括号来包裹整个算式或一段字节码，使其被优先运算或合并为整体
如：
```
a821 - <02 - 01>  //其中02 - 01为整体被运算，返回01，最终返回a820
```
```
<02 04>  //返回0204，不过按理说02 04应该也会返回0204，但为了明晰边界，最好包裹整个字节码
```

- 方括号的使用
使用方括号包裹一段字节码，让其两两进行大小端转换
如：
```
[02 04]  //返回0402
```
```
[ 01 23 45 67 ]  //返回67452301
```

### gadgets
gadgets统一前缀为$，如
```
$er0=
```
gadgets本质上等效于一段字节码，如$er0= 等效于a821x1xx
gadgets命名较为宽泛，支持除了空格和左括号以外的任意字符，不过为了避免冲突，建议少用一些可能会冲突的字符

### 简单函数
简单函数统一前缀为*，如
```
*print (xxxx, xx)
```

### 高阶函数
高阶函数统一前缀为!，如
```
!loop (xxxx, xxxx){
    ...    //这里是循环体
}
```

### 偏移量声明
使用@offset= xxxx 来声明偏移量，如
```
@offset= 0xd710
```
在调用一些依赖地址标签的简单函数或高阶函数时，务必填入程序运行处域的真实起始地址，否则可能会无法正常运行
偏移量可以在一个程序块内随时重新声明

### 占位符声明
使用@x= x （其中x是任意十六进制数字）来声明一个占位符，如:
```
@x=0
```
这时，字节码中的x会被替换为0
同样地，占位符也可以在一个程序块内随时重新声明

### 地址标签（重要！！）
使用@adr.xxx 来标记一个地址位置，然后用#xxx来调用这个双字节地址（大端，会加上偏移量）
如果用##xxx，则返回没有偏移量的地址。
这么讲可能有点不好懂。举个例子：
```
@block.main:
   @x=0
   @offset= d710
   
   @adr.start
   00 00 00 00
   @adr.end
   @offset= e9e0
   @adr.end_2
   #start  //返回d710
   ##start //返回0000
   #end    //返回d714
   ##end   //返回0004
   #end_2  //返回e9e4
   ##end_2   //返回0004
@blockend
```
```
@block.launcher:
   @x=0
   @offset= d180
   
   #end    //返回d714,地址标签可以在程序块之间互通
```

### 注释
每一行的//后面就是注释

### 覆写
使用@overwrite (地址, 字节码)来在指定地址处覆写字节码，字节长度不限，除非超出边界
地址和字节码都支持表达式
@overwrite必须要在程序块内部

## 如何开始
新建一个文件，先写好导入的配置（这里我提供了几个默认的配置，verf的还没整理，希望有大佬帮忙整理），函数和gadgets定义
然后写好@block.xxx:
之后写入代码
最后用@blockend来结束一个程序块。

写好后，保存
进入ropc.exe（或ropc.py）所在的目录，将配置文件放在ropc同目录下，然后直接运行ropc.exe，
然后按照程序指示操作

当然，你也可以在命令行内运行：
```
./ropc.exe 文件路径(可以用相对路径也可以用绝对路径)
```
然后就能编译出.bin文件，每一个程序块会对应一个.bin文件
如果要让它编译出16进制字符串文本，则运行：
```
./ropc.exe 文件路径 -t
```

如果你不是windows系统或有别的原因想要直接运行.py文件，那就安装python，然后运行
```
pip install lark
```
```
python ropc.py
```
然后根据程序指示操作即可

## 配置文件语法
- gadgets
```
$name {value}
```

- 简单函数
```
*name (arg0(=defaultvalue?), arg1......){
    body
}
```

- 高阶函数
```
!name (arg0(=defaultvalue?), ......){%%BODY%%}{
   ......
}
```

在函数体中用
```
%_arg_%
```
来访问arg参数
如果要使用地址标签，那么用
```
@adr.&_xxx_&
```
```
#&_xxx_&
```
```
##&_xxx_&
```
来限制标签作用域，防止与外界标签冲突。

## gadgets，函数规范
- gadgets命名
如果其最后会pop寄存器，则在&后面加上对应的编码；如果该gadgets还伴随其它效果，则在前面加上问号。
比如，wait_key会伴随
```
mov sp, er14
pop xr4
pop qr8
```
则其命名为
```
$?waitkeyto[er8]&x4q8
```

- 函数传参
函数传参统一使用大端




-----

# ROP\_Compiler

A ROP chain compilation tool for **nx-u16 and nx-u8 series chips**. It supports **gadgets, custom functions, and address labels**.

Here are some basic configuration files provided for reference: **verc.ggt, common.spf, common.cpf**. Everyone is welcome to develop their own configuration files.

The gadgets for **verf** have not yet been organized. Contributions from experienced developers would be greatly appreciated.

The program may contain bugs. Please feel free to **raise an issue** or leave a message on the **Baidu Tieba** forum.

If the response is positive, the development of a **VS Code extension** will be considered.

This program uses its own syntax system. The syntax is introduced below.

## Syntax Guide

### Importing Configurations

Add the following at the beginning of the file:
`import <filename>`
e.g.:
`import verc.ggt`
This imports the configuration file. The program will read **gadgets**, **simple functions**, and **high-level functions** from it.

### Function and Gadget Definition

Use
`def ...`
to define a function. The syntax is the same as in the configuration file, just prepended with `def`.

### Program Block

Use `@block.xxx:` to start a program block, where `xxx` is any variable name, e.g.:
`@block.main:`
Don't forget to end the block with `@blockend`.
Example:

```
@block.main:
    // Program instructions...
@blockend
```

### Bytecode

Bytecode has no prefix and can be entered directly. Case and spaces are automatically ignored.
e.g.:
`0123 abcd`
You can use `x` as a placeholder, e.g.:
`a821x1xx`

#### Bytecode Operations

  - **Addition and Subtraction**
    Bytecodes can be added or subtracted by inputting the `+` or `-` operator between them. It is best to separate them with spaces.
    The bit length of the result will be the same as the longest bytecode involved in the operation.
    If the result is a negative number, its **two's complement** is returned.
    e.g.:
    `a821 + 01 - a800 // returns 0022`

  - **Using Angle Brackets (`< >`)**
    Angle brackets can be used to enclose an entire expression or a segment of bytecode to prioritize its calculation or to treat it as a single unit.
    e.g.:
    `a821 - <02 - 01> // 02 - 01 is calculated first, returning 01, final result is a820`
    `<02 04> // returns 0204. Although 02 04 should also return 0204, it is best to wrap the entire bytecode for explicit boundary definition.`

  - **Using Square Brackets (`[ ]`)**
    Square brackets are used to wrap a segment of bytecode, which then undergoes **byte-swapping** in pairs (endianness conversion).
    e.g.:
    `[02 04] // returns 0402`
    `[ 01 23 45 67 ] // returns 67452301`

### Gadgets

Gadgets are uniformly prefixed with **`$`**, e.g.:
`$er0=`
A gadget is essentially equivalent to a segment of bytecode, e.g., `$er0=` is equivalent to `a821x1xx`.
Gadget naming is quite flexible, supporting any character except spaces and left parentheses. However, to avoid conflicts, it is recommended to avoid characters that may potentially conflict with the syntax.

### Simple Functions

Simple functions are uniformly prefixed with **`*`**, e.g.:
`*print (xxxx, xx)`

### High-Level Functions

High-level functions are uniformly prefixed with **`!`**, e.g.:

```
!loop (xxxx, xxxx){
    // Loop body goes here
}
```

### Offset Declaration

Use `@offset= xxxx` to declare the offset, e.g.:
`@offset= 0xd710`
When calling simple or high-level functions that rely on address labels, you must provide the **true starting address** of the program's execution area, or the program may fail to run correctly.
The offset can be **re-declared at any time** within a program block.

### Placeholder Declaration

Use `@x= x` (where `x` is any hexadecimal digit) to declare a placeholder, e.g.:
`@x=0`
At this point, any `x` in the bytecode will be replaced by `0`.
Similarly, the placeholder can be **re-declared at any time** within a program block.

### Address Labels (Important\!\!)

Use **`@adr.xxx`** to mark an address location. Then use **`#xxx`** to reference this 2-byte address (big-endian, **offset included**).
If you use **`##xxx`**, the address is returned **without the offset**.

This might be a bit confusing. Here is an example:

```
@block.main:
    @x=0
    @offset= d710
    
    @adr.start
    00 00 00 00
    @adr.end
    @offset= e9e0
    @adr.end_2
    #start  // returns d710
    ##start // returns 0000
    #end    // returns d714
    ##end   // returns 0004
    #end_2  // returns e9e4
    ##end_2 // returns 0004
@blockend
@block.launcher:
    @x=0
    @offset= d180
    
    #end    // returns d714. Address labels are global and can be shared between program blocks.
```

### Comments

Everything after **`//`** on any line is a comment.

### Overwrite

Use `@overwrite (address, bytecode)` to overwrite bytecode at a specified address. Byte length is unrestricted unless it exceeds the boundary.
Both the address and bytecode support expressions.
`@overwrite` must be used **inside a program block**.

-----

## How to Get Started

1.  Create a new file. First, write the **import configurations** (I have provided a few default configurations here; the ones for verf haven't been organized yet, and I hope some experienced devs can help), function, and gadget definitions.
2.  Next, write the program block starting with **`@block.xxx:`**.
3.  Then, write your code.
4.  Finally, use **`@blockend`** to conclude the program block.

After writing and saving the file:

**If using the executable:**

1.  Navigate to the directory where `ropc.exe` (or `ropc.py`) is located. Place the configuration files in the same directory as `ropc`.
2.  Run `ropc.exe` directly.
3.  Follow the program instructions.

**Alternatively, via the command line:**

  * To compile to a **.bin** file (each program block corresponds to one .bin file):
    `./ropc.exe path/to/your/file (can be relative or absolute path)`
  * To compile to a **hexadecimal string text** file:
    `./ropc.exe path/to/your/file -t`

**If you are not on Windows or prefer to run the .py file directly:**

1.  Install Python.
2.  Run `pip install lark`.
3.  Run `python ropc.py`.
4.  Follow the program instructions.

-----

## Configuration File Syntax

  - **Gadgets**
    `$name {value}`

  - **Simple Functions**

    ```
    *name (arg0(=defaultvalue?), arg1......){
        body
    }
    ```

  - **High-Level Functions**

    ```
    !name (arg0(=defaultvalue?), ......){%%BODY%%}{
        ......
    }
    ```

In the function body, use
`%_arg_%`
to access the argument value.

To use address labels within a function's scope (to prevent conflicts with external labels), use:
`@adr.&_xxx_&`
`#&_xxx_&`
`##&_xxx_&`

## Gadgets and Function Specifications
- Gadget Naming  
  If a gadget will pop a register at the end, add the corresponding code after the `&` symbol; if the gadget has additional effects, add a question mark `?` at the beginning.  

  For example, the `wait_key` gadget is accompanied by:  
  ```
  mov sp, er14
  pop xr4
  pop qr8
  ```  
  Its name should be:  
  ```
  $?waitkeyto[er8]&x4q8
  ```

- Function Parameter Passing  
  All function parameters must be passed in big-endian order.