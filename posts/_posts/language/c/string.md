---
title: C String
date: 2022-06-06
comment: true
---

复习 C 语言字符串及相关库函数 `gets` `fgets` `puts` `fputs` 等..

<!--more-->

**字符串 （character string）** 是一个或多个字符的序列。实际上，字符串就是一个字符数组，**但是在字符末尾插入了 \0 **用来表示字符串的结束。



所以，数组的容量一定是要比待存储的字符串字节数多 1，用来存储 \0，举个例子，我们想存储名字 `abc ` 就必须使用 `char name[4]` 来存储。

> 末尾的 \0 计算机会自动帮我们加上



## 使用

```c
// 接收控制台输入，parisel.c
#include <stdio.h>

#define PRAISE "good jobbbb"

int main(void) {
    char name[40];

    printf("Input you name ");
    scanf("%s", name);

    printf("%s, %s\n", name, PRAISE);
    return 0;
}
```

> 为什么 scanf 接收参数的时候不需要传递指针？
>

## 字符串和字符

```c
// 字符串 "x"
// 字符   'x'
```

这两个虽然看起来相似，但是底层的存储结构是不一样的，字符使用 char 类型来存储，字符串使用的是数组，而且还需要一个 `\0`，由两个字符组成。



## 常量

为什么使用符号常量更好？

- *常量名称可以表达更多的意思*
- *方便后续修改*



### 定义常量

`#define PI 3.14` 更通用的表示为 `#define NAME value`, C 中一般采用大写名称来表示常量。



代码中使用这样的常量，会在编译时替换掉（预处理阶段替换），即*编译时替换*（compile-time substitution）。



### const 限定符号

Go 中使用的方式 `const name = "xx"`

C 中要这样用 `const int Months = 12;`

> C 中 const 声明的是变量，不是常量。



## 库函数

### `puts`

与 `printf()` 类似，不同的是只显示字符串并在末尾加上换行符。

```c
#include <stdio.h>

#define MSG "i am a string"
#define MAXLENGTH 81

int main(void) {
    char words[MAXLENGTH] = "today i am very excited";
    const char * pt1 = "a prt string variable";

    puts(MSG);
    puts(words);
    puts(pt1);

    words[0] = 'l';
    puts(words);
    return 0;
}
```



> 小插曲：个人感觉还是对栈地址空间的不熟悉，详情见如下代码

```c
#include <stdio.h>

int main(void) {
    char side_a[] = "Side A";
    char dont[] = {'W', 'O', 'W', '!'};
    char side_b[] = "Side B";

    puts(dont);
    return 0;
}
```

**`puts` 函数在遇到空字符的时候停止输出**，向上述代码我们打印 dont 变量，因为没有字符串结束的标志，所以它会一直扫描下去。如果我这里说它打印的是 `WOW!SIDE A` 会不会感觉很奇怪？



我一开始想的是 `puts` 会一直往后扫，不应该输出 `WOW!Side B` 吗，`Side A` 不是往前扫了吗，如果你也这么认为说明我们陷入了同一个误区。 **输出结果取决于你编译器是怎样存储数据的** ，也就意味着如果你的栈地址分配是从高到低，那输出就是 A，反之是 B。



**字符串的三种声明方式** （在上述代码中都有体现）

- 字符串常量
- char 类型数组
- 指向 char 的指针



**数组形式和指针形式有何不同**？

初始化数组把静态存储区的字符串拷贝到数组中，初始化指针只把字符串的首地址拷贝给指针。

代码验证：

```c
#include <stdio.h>

#define MSG "i'm special"

int main(void) {
    char ar[] = MSG;
    const char * pt = MSG;
    printf("address of \"i'm special\": %p \n", "i'm special");
    printf("               address ar: %p\n", ar);
    printf("               address pt: %p\n", pt);
    printf("               address MSG: %p\n", MSG);
    printf("address of \"i'm special\": %p \n", "i'm special");
    return 0;
}

/*

Output:

address of "i'm special": 0x55779caeb000 
              address ar: 0x7ffcc126542c
              address pt: 0x55779caeb000
             address MSG: 0x55779caeb000
address of "i'm special": 0x55779caeb000 

*/
```

能够看到，MSG 的地址与第一个 `printf` 最后一个 `printf` 相同，应该是编译器优化的结果。另外，数组声明方式的地址与 MSG 不同，与前边提到的相符。



### `gets`

`gets` 存在一个致命问题是，无法判断字符数组能否容下输入的内容。

```c
#include <stdio.h>

#define STLEN 81

int main(void) {
    char words[STLEN];

    puts("Enter string.");
    // 如果输入内容超过 81 个
    // 
    gets(words);

    printf("your string twice:\n");
    printf("%s\n", words);

    puts(words);
    puts("Done");
    return 0;
}
```

如果说输入的内容超了，会发生 `segmentation fault` 。



### `fgets fputs`

`char *fgets(char *restrict s, int size, FILE *restrict stream);`

`int fputs(const char *restrict s, FILE *restrict stream);`



作为 `gets` 的替代品 `fgets` 解决了上述问题，它俩在读入字符时的主要区别如下：

- `fgets` 的第二个参数指定了读入字符长度，或读到换行符为止
- `fgets` 读到换行符会将其保存在字符串中，`gets` 则是丢弃换行符
- `fgets` 第三个参数指明要读入的文件，如果是控制台输入则为 `stdin`



`fputs` 与 `fgets`  配对使用

- 第二个参数是要写入的地方，如果是控制台，则为 `stdout`
- 与 `puts` 不同的是 `fputs` 不会在末尾加上换行符
