---
title: CSAPP Lab (02)-Bomb Lab
date: 2021-03-14
toc: true
categories:
- 基础知识	
- CSAPP

---

> 本文记录了CSAPP配套实验——Bomb Lab的详情WriteUp

- 运行环境：

  - Ubuntu 20.04

---

### 任务

拆除"bomb"：通过反汇编，拆除bomb，以达到拆弹的目的。

---

### 知识储备

- **GDB调试器的使用**

对于C/C++程序的调试，需要在编译前加上`-g`。

```shell
$g++ -g test.cpp -o test
```

接下来开启`gdb`调试器调试程序：

```shell
$gdb test
```

启用`gdb`进入交互模式后，可以用如下命令调试程序

- **运行**

| Instruction           | Function                                                     |
| :-------------------- | ------------------------------------------------------------ |
| r(un)                 | 运行程序，当遇到断点后，程序会在断点处停止运行，等待用户输入下一步的命令 |
| c(ontinue)            | 继续执行，到下一个断点处（或运行结束）                       |
| n(ext)                | 单步跟踪程序，当遇到函数调用时，也不进入此函数体             |
| s(tep)                | 单步调试如果有函数调用，则进入函数                           |
| until( lineNumber)    | 运行程序直到退出循环体，加行号可以直接运行到某行             |
| finish                | 运行至当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息。 |
| call function(params) | 调用程序中可用函数，并传递参数，如对于 int sum(int a, int b)，可使用call sum(1, 2) |
| q(uit)                | 退出gdb调试器                                                |
| b(reak )              | 设置断点                                                     |
| info r                | 查看寄存器                                                   |
| p                     | 输出变量                                                     |
| x                     | 查看内容                                                     |
| set args              | 设置函数运行参数                                             |

- **设置断点**

---

### WriteUp

​	题目一共给了6个bombs，打开`bomb.c`查看部分源码，其实关键在于这几段

![image-20210326172525304](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210326172525304.png)

程序读取`input`字符串，如果输入`正确的`字符串，那么就拆解成功一个炸弹，~~否则你就会像下图一样被blown up~~

![image-20210326173121180](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210326173121180.png)

先执行下面的指令得到程序的反汇编代码，然后我们进行愉快的拆解

```shell
$objdump -d bomb > bomb.asm
```

- #### phase1

  在bomb.asm中找到`phase_1()`的汇编代码段如下：

  ![image-20210326181958171](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210326181958171.png)

  这里关于寄存器的几点需要补充：

  - %rdi:  保存了函数入口的**第一个**参数
  - %rsi:  保存了函数入口的**第二个**参数
  - %rdx: 保存了函数入口的**第三个**参数
  - %rcx: 保存了函数入口的**第四个**参数
  - %rax：函数的**返回值**

  由此，要避免0x400ef2处触发explode_bomb，需要在上一行进行跳转，跳转条件**%eax值为0**（test是先进行与运算再设置PSW的，判断相等的条件为**ZF==0**），因此我们去strings_not_equal处的汇编代码查看

  ![image-20210326182825730](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210326182825730.png)

  输入参数为%rdi和%rsi，返回值为%rax，当两个输入存储的值不相等时，返回%eax值1，可以确定，**%si存储了待比较的字符串**，也就是bomb1的key，因此我们在该函数处设置断点，并以字符串的形式打印%si中的值

  ```shell
  $gdb bomb
  (gdb)break strings_not_equal
  (gdb)r
  (gdb) p (char*)$rsi
  $1 = 0x402400 "Border relations with Canada have never been better."
  (gdb) p (char*)$rdi
  $2 = 0x603780 <input_strings> "2333"
  ```

  因此，bomb1的key为：**Border relations with Canada have never been better.**

  ![image-20210326183755389](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210326183755389.png)

  bomb1拆除成功。

- #### phase2

  `phase_2()`的汇编代码如下

  ![image-20210326184232130](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210326184232130.png)

  这里不难看到`phase_2()`其实先利用`read_six_numbers`读入6个数，再逐个比较。数字比较部分的代码分析如下：

  注意：

  - %rsp生长方向为地址减小的方向，因此，对于存储在%rsp中的`arr`首地址，`lea 0x4(%rsp), %rbx`实际表示为`%rbx = arr[i + 1]`

  ```assembly
  0000000000400efc <phase_2>:
    400efc:	55                   	push   %rbp
    400efd:	53                   	push   %rbx
    400efe:	48 83 ec 28          	sub    $0x28,%rsp         //开辟栈空间
    400f02:	48 89 e6             	mov    %rsp,%rsi		//%rsp实际上保存了数组的首地址，
    400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers>
    400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)			//1. arr[0] = 1
    400f0e:	74 20                	je     400f30 <phase_2+0x34>	
    400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
    400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
    400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax		//4. %eax = arr[?-1]
    400f1a:	01 c0                	add    %eax,%eax			//5. %eax *= 2
    400f1c:	39 03                	cmp    %eax,(%rbx)			//6. if %eax == arr[?] 
    400f1e:	74 05                	je     400f25 <phase_2+0x29>//
    400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
    400f25:	48 83 c3 04          	add    $0x4,%rbx			//7. arr[?+1]
    400f29:	48 39 eb             	cmp    %rbp,%rbx			//8. arr[?] != arr[6],loop to 4
    400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
    400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
    400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx		//2. (%rbx) = arr[1]
    400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp		//3. (%rbp) = arr[6]，0x18是十进制的24
    400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
    400f3c:	48 83 c4 28          	add    $0x28,%rsp
    400f40:	5b                   	pop    %rbx
    400f41:	5d                   	pop    %rbp
    400f42:	c3                   	retq  
  ```

  根据对汇编代码的分析，可以得知代码的逻辑如下：

  ```c
  if (arr[0] != 1)
  	explode_bomb();
  for (int i = 1; i <= 6; ++i) {
  	int tmp = arr[i - 1] * 2;
  	if (tmp != arr[i])
  		explode_bomb();
  }
  ```

  因此，`phase_2()`的key为首项为1，公比为2的等比数列的前6项：**1 2 4 8 16 32**

- #### phase3

  先让我们看一下源码

  ![image-20210327213828651](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210327213828651.png)

  

- #### phase4

- #### phase5

- #### phase6