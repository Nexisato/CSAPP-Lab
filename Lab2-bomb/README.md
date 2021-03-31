---
title: CSAPP Lab (02)-Bomb Lab
date: 2021-03-26
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

  ```assembly
  0000000000400f43 <phase_3>:
    400f43:	48 83 ec 18          	sub    $0x18,%rsp
    400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
    400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx	
    400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
    400f56:	b8 00 00 00 00       	mov    $0x0,%eax
    400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt>	//sscanf返回读取到的item个数
    400f60:	83 f8 01             	cmp    $0x1,%eax
    400f63:	7f 05                	jg     400f6a <phase_3+0x27>
    400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb>
    400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)		
    400f6f:	77 3c                	ja     400fad <phase_3+0x6a>	//num1>7,bomb()
    400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax			//eax = num1
    400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)		//(0x402470[8*num1])处的值
    400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax				
    400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
    400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
    400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
    400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
    400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
    400f91:	b8 85 01 00 00       	mov    $0x185,%eax
    400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
    400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
    400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
    400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
    400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
    400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
    400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
    400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
    400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
    400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
    400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
    400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
    400fc2:	74 05                	je     400fc9 <phase_3+0x86>
    400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
    400fc9:	48 83 c4 18          	add    $0x18,%rsp
    400fcd:	c3                   	retq   
  ```

  源码中可以看到调用了`sscanf`库函数，该函数表示返回读入参数的个数，我们设置断点查看

  ```bash
  (gdb)break *0x400f60
  ---
  phase_3():
  INPUT:2 3 3 3
  ---
  (gdb)info r eax
  eax            0x2                 2
  ```

  因此phase_3()只读取两个参数，查看0x8(%rsp)和0xc(%rsp)

  ```asm
  Breakpoint 1, 0x0000000000400f60 in phase_3 ()
  (gdb) x $sp
  0x7fffffffdf00:	0x00402210
  (gdb) x $rsp
  0x7fffffffdf00:	0x00402210
  (gdb) x $rsp+0x8
  0x7fffffffdf08:	0x00000002
  (gdb) x $rsp+0xc
  0x7fffffffdf0c:	0x00000003
  ```

  分别放在`0x8(%rsp)`和`0xc(%sp)`（输入的参数入栈，最后一个在栈底`0xc`，高地址；第一个在栈顶，低地址`0x8`），我们假设两处输入的参数分别为`num1`和`num2`，且0<=`num1`<=7（**ja只判定无符号数**），num1被存储在%eax中，根据

  ```asm
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)		//(0x402470[8*num1])处的值
  ```

  可知，根据输入不同的num1，需要跳转到不同的地址处，我们从0x402470开始连续打印8个地址（x64机器每个地址8个字节）

  ```asm
  (gdb) x/8xg 0x402470
  0x402470:       0x0000000000400f7c      0x0000000000400fb9
  0x402480:       0x0000000000400f83      0x0000000000400f8a
  0x402490:       0x0000000000400f91      0x0000000000400f98
  0x4024a0:       0x0000000000400f9f      0x0000000000400fa6
  ```

  所以根据num1=[0..7]，分别前往源码中不同的地址处对%eax赋值操作，跳转处标记在源码中

  ```assembly
    400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax				//num1=0,eax=0xcf=207
    400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
    400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax				//num1=2,eax=0x2c3=707
    400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
    400f8a:	b8 00 01 00 00       	mov    $0x100,%eax				//num1=3,eax=0x100=256
    400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
    400f91:	b8 85 01 00 00       	mov    $0x185,%eax				//num1=4,eax=0x185=389
    400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
    400f98:	b8 ce 00 00 00       	mov    $0xce,%eax				//num1=5,eax=0xce=206
    400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
    400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax				//num1=6,eax=0x2aa=682
    400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
    400fa6:	b8 47 01 00 00       	mov    $0x147,%eax				//num1=7,eax=0x147=327
    400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
    400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
    400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
    400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
    400fb9:	b8 37 01 00 00       	mov    $0x137,%eax			//num1=1,eax=0x137=311
    400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax		//num2 == eax? (defused()) : bomb
    400fc2:	74 05                	je     400fc9 <phase_3+0x86>
    400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
  ```

  因此，phase_3()的key：num1 num2，对应以下8组中任意一组即可：

  | num1 | num2 |
  | ---- | ---- |
  | 0    | 207  |
  | 1    | 311  |
  | 2    | 707  |
  | 3    | 256  |
  | 4    | 389  |
  | 5    | 206  |
  | 6    | 682  |
  | 7    | 327  |

- #### phase4

  ```assembly
  000000000040100c <phase_4>:
    40100c:	48 83 ec 18          	sub    $0x18,%rsp
    401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx	%rcx=num2
    401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
    40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
    40101f:	b8 00 00 00 00       	mov    $0x0,%eax
    401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
    401029:	83 f8 02             	cmp    $0x2,%eax
    40102c:	75 07                	jne    401035 <phase_4+0x29>	
    40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)
    401033:	76 05                	jbe    40103a <phase_4+0x2e>	//num1 <= 0xe(14)
    401035:	e8 00 04 00 00       	callq  40143a <explode_bomb>
    40103a:	ba 0e 00 00 00       	mov    $0xe,%edx			//edx = 14
    40103f:	be 00 00 00 00       	mov    $0x0,%esi			//esi = 0
    401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi		//edi = num1
    401048:	e8 81 ff ff ff       	callq  400fce <func4>
    40104d:	85 c0                	test   %eax,%eax
    40104f:	75 07                	jne    401058 <phase_4+0x4c>
    401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
    401056:	74 05                	je     40105d <phase_4+0x51>
    401058:	e8 dd 03 00 00       	callq  40143a <explode_bomb>
    40105d:	48 83 c4 18          	add    $0x18,%rsp
    401061:	c3                   	retq  
  ```

  依然是读入两个参数num1(0x8(%rsp))和num2(0xc(%rsp))，且由跳转函数可以得到0<=`num1`<=14，经过了三个寄存器的赋值以后，函数跳转到`fun4`，不过在这之前我们可以看到，在后续的汇编代码中，需要满足`%eax = 0`和`num2=0`两个条件。以下是func4的汇编代码：

  ```asm
  0000000000400fce <func4>://已知的参数:%edx=14,%esi=0,%edi=num1
    400fce:	48 83 ec 08          	sub    $0x8,%rsp
    400fd2:	89 d0                	mov    %edx,%eax	//%eax=14
    400fd4:	29 f0                	sub    %esi,%eax	//%eax -= (%esi)
    400fd6:	89 c1                	mov    %eax,%ecx	//%ecx=%eax
    400fd8:	c1 e9 1f             	shr    $0x1f,%ecx	//取%ecx的符号位 
    400fdb:	01 c8                	add    %ecx,%eax	//%eax += %ecx
    400fdd:	d1 f8                	sar    %eax
    400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx
    400fe2:	39 f9                	cmp    %edi,%ecx	//num1 >= %ecx
    400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
    400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx	%edx=%rcx-1
    400fe9:	e8 e0 ff ff ff       	callq  400fce <func4>
    400fee:	01 c0                	add    %eax,%eax		%eax*=2
    400ff0:	eb 15                	jmp    401007 <func4+0x39>//done
    400ff2:	b8 00 00 00 00       	mov    $0x0,%eax	//%eax=0
    400ff7:	39 f9                	cmp    %edi,%ecx	//num1<=%ecx，done
    400ff9:	7d 0c                	jge    401007 <func4+0x39>
    400ffb:	8d 71 01             	lea    0x1(%rcx),%esi	//%esi=%rcx+1
    400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
    401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
    401007:	48 83 c4 08          	add    $0x8,%rsp
    40100b:	c3                   	retq 
  ```

  > 注：类似于  cmp    %edi,%ecx	jle    400ff2 <func4+0x24>；跳转语句的主语是dst，如左边这句表示，compare %edi : %ecx，若 %ecx <=(lower or equal) %edi，则jump到0x400ff2处

  分析两次跳转条件，可以得到`%ecx=num1`，即可返回`%eax=0`，汇编代码里面用了比较多的递归，为了方便计算我们画出算法流程图后写个测试函数看看num1应该是多少：

  ```c
  #include <stdio.h>
  
  int fun4(int dx, int si, int di) {
     int ax = dx - si;
     int cx = (unsigned)ax >> 31;
     ax = (ax + cx) >> 1;
     cx = ax + si;
     if (di >= cx) {
        ax = 0;
        if (di <= cx) {
           return ax;
        }  
        else {
           si = cx + 1;
           ax = fun4(dx, si, di);
           return 2*ax + 1;
        }
     }
     else {
        dx = cx - 1;
        ax = fun4(dx, si, di);
        return 2 * ax;
     }
  }
  int main() {
     for (int num1 = 0; num1 <= 14; ++num1)
        if (fun4(14, 0, num1) == 0)
           printf("%d\n", num1);
  
     return 0;
  }
  ----运行结果----
  0
  1
  3
  7
  ```

  所以**phase_4()**的key为如下4组中的任意一组即可：

  **0 0**

  **1 0**

  **3 0**

  **7 0**

- #### phase5

  反汇编代码如下：

  ```assembly
  0000000000401062 <phase_5>:
    401062:	53                   	push   %rbx
    401063:	48 83 ec 20          	sub    $0x20,%rsp
    401067:	48 89 fb             	mov    %rdi,%rbx
    40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
    401071:	00 00 
    401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
    401078:	31 c0                	xor    %eax,%eax
    40107a:	e8 9c 02 00 00       	callq  40131b <string_length>
    40107f:	83 f8 06             	cmp    $0x6,%eax
    401082:	74 4e                	je     4010d2 <phase_5+0x70>
    401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb>
    401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
    40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
    40108f:	88 0c 24             	mov    %cl,(%rsp)
    401092:	48 8b 14 24          	mov    (%rsp),%rdx
    401096:	83 e2 0f             	and    $0xf,%edx
    401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
    4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
    4010a4:	48 83 c0 01          	add    $0x1,%rax
    4010a8:	48 83 f8 06          	cmp    $0x6,%rax
    4010ac:	75 dd                	jne    40108b <phase_5+0x29>
    4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
    4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
    4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
    4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal>
    4010c2:	85 c0                	test   %eax,%eax
    4010c4:	74 13                	je     4010d9 <phase_5+0x77>
    4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb>
    4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
    4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
    4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
    4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
    4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
    4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
    4010e5:	00 00 
    4010e7:	74 05                	je     4010ee <phase_5+0x8c>
    4010e9:	e8 42 fa ff ff       	callq  400b30 <__stack_chk_fail@plt>
    4010ee:	48 83 c4 20          	add    $0x20,%rsp
    4010f2:	5b                   	pop    %rbx
    4010f3:	c3                   	retq 
  ```

  从0x40107f处可以看到，phase5需要我们输入长度为`6`的字符串，我们先输入长度为6的字符串，并在0x40108b处设置断点，可以看到

  ```asm
  (gdb) p (char*)$rbx
  $2 = 0x6038c0 <input_strings+320> "abcdef"
  ```

  %rbx存储了我们输入字符串的首地址，可以看到0x4010ac处之前是在sp中存储这6位字符，然后直到0x4010bd处比较字符串。这样我们其实可以直接查询%rsi存储的地址处的字符串

  ```asm
  (gdb) p (char*)$esi
  $5 = 0x40245e "flyers"
  ```

  这就结束了？**不！**这不只是简单比较一下字符串就行，此前对输入的字符串进行了转换映射。我们将如下的汇编代码用高级语言改写：

  ```asm
    40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
    40108f:	88 0c 24             	mov    %cl,(%rsp)
    401092:	48 8b 14 24          	mov    (%rsp),%rdx
    401096:	83 e2 0f             	and    $0xf,%edx
    401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
    4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
    4010a4:	48 83 c0 01          	add    $0x1,%rax
    4010a8:	48 83 f8 06          	cmp    $0x6,%rax
    4010ac:	75 dd                	jne    40108b <phase_5+0x29>
    ------
    %rbx：我们输入的字符，设定为 inputs[i]，经过与0x4024b0的字符运算后存储在0x10(%rsp)处
    0x4024b0：事先存储的字符，设定为arr[i]
     int sp[6];
     for (int i = 0; i < 6; ++i) 
        sp[i] = arr[inputs[i] & 0xf];
    ------
  ```

  对于任意一个数num，有`num&0xf<=0xf`，因此我们可以查看0x4024b0处开始的16个字符

  ![image-20210329113622421](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210329113622421.png)

  可以得到

  ```c
  int arr[16] = {'m', 'a', 'd', 'u', 'i', 'e', 'r', 's', 'n', 'f', 'o',' t', 'v', 'b', 'y', 'l'};
  ```

  最终0x10(%rsp)处的字符（也即sp[]）被映射到%rdi处，%rsi处的字符为

  ![image-20210329113943236](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210329113943236.png)

  所以只需要找到合适的`inputs`，并作映射`sp[i]=arr[inputs[i]&0xf]`，使得`string(sp)=="flyers" `，写个测试函数找一找：

  ```c++
      string arr = "maduiersnfotvbyl";
      string si = "flyers";
      for (int i = 0; i < 6; ++i) {
          cout << si[i] << ": ";
          for (int k = 0; k < 128; k++)
              if (arr[int(k & 0xf)] == si[i])
                  cout << (char)k;
          cout << endl;
      }
  ----
  f: )9IYiy
  l: /?O_o
  y: .>N^n~
  e: %5EUeu
  r: &6FVfv
  s: '7GWgw
  ```

  以上**6行字符的任意排列组合**均可拆除phase5()

  其实这里有个彩蛋，`Ctrl+C`可以直接拆弹，这是我没想到的2333

  ![image-20210329130013620](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210329130013620.png)

- #### phase6

  激动人心的时刻，终于来到最后一个phase了，让我们继续看一下源码

  ```asm
  00000000004010f4 <phase_6>:
    4010f4:	41 56                	push   %r14
    4010f6:	41 55                	push   %r13
    4010f8:	41 54                	push   %r12
    4010fa:	55                   	push   %rbp
    4010fb:	53                   	push   %rbx
    4010fc:	48 83 ec 50          	sub    $0x50,%rsp
    401100:	49 89 e5             	mov    %rsp,%r13
    401103:	48 89 e6             	mov    %rsp,%rsi
    401106:	e8 51 03 00 00       	callq  40145c <read_six_numbers>
    40110b:	49 89 e6             	mov    %rsp,%r14
    40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
    401114:	4c 89 ed             	mov    %r13,%rbp
    401117:	41 8b 45 00          	mov    0x0(%r13),%eax
    40111b:	83 e8 01             	sub    $0x1,%eax
    40111e:	83 f8 05             	cmp    $0x5,%eax
    401121:	76 05                	jbe    401128 <phase_6+0x34>
    401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
    401128:	41 83 c4 01          	add    $0x1,%r12d
    40112c:	41 83 fc 06          	cmp    $0x6,%r12d
    401130:	74 21                	je     401153 <phase_6+0x5f>
    401132:	44 89 e3             	mov    %r12d,%ebx
    401135:	48 63 c3             	movslq %ebx,%rax
    401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
    40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
    40113e:	75 05                	jne    401145 <phase_6+0x51>
    401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
    401145:	83 c3 01             	add    $0x1,%ebx
    401148:	83 fb 05             	cmp    $0x5,%ebx
    40114b:	7e e8                	jle    401135 <phase_6+0x41>
    40114d:	49 83 c5 04          	add    $0x4,%r13
    401151:	eb c1                	jmp    401114 <phase_6+0x20>
    401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
    401158:	4c 89 f0             	mov    %r14,%rax
    40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
    401160:	89 ca                	mov    %ecx,%edx
    401162:	2b 10                	sub    (%rax),%edx
    401164:	89 10                	mov    %edx,(%rax)
    401166:	48 83 c0 04          	add    $0x4,%rax
    40116a:	48 39 f0             	cmp    %rsi,%rax
    40116d:	75 f1                	jne    401160 <phase_6+0x6c>
    40116f:	be 00 00 00 00       	mov    $0x0,%esi
    401174:	eb 21                	jmp    401197 <phase_6+0xa3>
    401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
    40117a:	83 c0 01             	add    $0x1,%eax
    40117d:	39 c8                	cmp    %ecx,%eax
    40117f:	75 f5                	jne    401176 <phase_6+0x82>
    401181:	eb 05                	jmp    401188 <phase_6+0x94>
    401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
    401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
    40118d:	48 83 c6 04          	add    $0x4,%rsi
    401191:	48 83 fe 18          	cmp    $0x18,%rsi
    401195:	74 14                	je     4011ab <phase_6+0xb7>
    401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
    40119a:	83 f9 01             	cmp    $0x1,%ecx
    40119d:	7e e4                	jle    401183 <phase_6+0x8f>
    40119f:	b8 01 00 00 00       	mov    $0x1,%eax
    4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
    4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
    4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
    4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
    4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
    4011ba:	48 89 d9             	mov    %rbx,%rcx
    4011bd:	48 8b 10             	mov    (%rax),%rdx
    4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
    4011c4:	48 83 c0 08          	add    $0x8,%rax
    4011c8:	48 39 f0             	cmp    %rsi,%rax
    4011cb:	74 05                	je     4011d2 <phase_6+0xde>
    4011cd:	48 89 d1             	mov    %rdx,%rcx
    4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
    4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
    4011d9:	00 
    4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
    4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
    4011e3:	8b 00                	mov    (%rax),%eax
    4011e5:	39 03                	cmp    %eax,(%rbx)
    4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
    4011e9:	e8 4c 02 00 00       	callq  40143a <explode_bomb>
    4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
    4011f2:	83 ed 01             	sub    $0x1,%ebp
    4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
    4011f7:	48 83 c4 50          	add    $0x50,%rsp
    4011fb:	5b                   	pop    %rbx
    4011fc:	5d                   	pop    %rbp
    4011fd:	41 5c                	pop    %r12
    4011ff:	41 5d                	pop    %r13
    401201:	41 5e                	pop    %r14
    401203:	c3                   	retq   
  
  ```

  比以往的长了很多，不过从输入处我们还是可以看到，此phase依然是读入6个数，我们逐段分析一下

  ```asm
    40110b:	49 89 e6             	mov    %rsp,%r14
    40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
    401114:	4c 89 ed             	mov    %r13,%rbp
    401117:	41 8b 45 00          	mov    0x0(%r13),%eax
    40111b:	83 e8 01             	sub    $0x1,%eax
    40111e:	83 f8 05             	cmp    $0x5,%eax
    401121:	76 05                	jbe    401128 <phase_6+0x34>
  ```

​        可以得到以下信息

- 输入的6个数字保存在%rsp、%rbp、%r13、%r14中，且存储的均为数组的首地址，我们假设该数组为**arr[]**

- arr[0]-1<=5

  继续往下看到0x401153

  ```asm
    401114:	4c 89 ed             	mov    %r13,%rbp
    
    401117:	41 8b 45 00          	mov    0x0(%r13),%eax
    40111b:	83 e8 01             	sub    $0x1,%eax
    40111e:	83 f8 05             	cmp    $0x5,%eax
    401121:	76 05                	jbe    401128 <phase_6+0x34>
    401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
    
    401128:	41 83 c4 01          	add    $0x1,%r12d
    40112c:	41 83 fc 06          	cmp    $0x6,%r12d
    401130:	74 21                	je     401153 <phase_6+0x5f>
    401132:	44 89 e3             	mov    %r12d,%ebx
    
    401135:	48 63 c3             	movslq %ebx,%rax
    401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
    40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
    40113e:	75 05                	jne    401145 <phase_6+0x51>
    401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
    
    401145:	83 c3 01             	add    $0x1,%ebx
    401148:	83 fb 05             	cmp    $0x5,%ebx
    40114b:	7e e8                	jle    401135 <phase_6+0x41>
    40114d:	49 83 c5 04          	add    $0x4,%r13
    401151:	eb c1                	jmp    401114 <phase_6+0x20>
    
    401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
    ------
    大意如下：
    for (int i = 0; i < 6; ++i) {
    	if (arr[i] > 6)
    		bomb();
    	for (int j = i + 1; j < 6; ++j)
    		if (arr[j] == arr[i])
    			bomb();
    }
    ------
  ```

  (这一段挺麻烦的orz..整个phase6我看了整整一天)，总之上一段程序可以得到如下结论

  - **输入的6个数字互不相等且不超过6**

  继续往下分析：

  ```assembly
    401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi	//si=arr.end()
    401158:	4c 89 f0             	mov    %r14,%rax
    40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
    401160:	89 ca                	mov    %ecx,%edx
    401162:	2b 10                	sub    (%rax),%edx
    401164:	89 10                	mov    %edx,(%rax)	//arr[i] = 7 - arr[i]
    401166:	48 83 c0 04          	add    $0x4,%rax
    40116a:	48 39 f0             	cmp    %rsi,%rax
    40116d:	75 f1                	jne    401160 <phase_6+0x6c>
    ------
    for (int i = 0; i < 6; ++i)
    	arr[i] = 7 - arr[i];
    ------
  ```

  继续往下看

  ```assembly
    40116f:	be 00 00 00 00       	mov    $0x0,%esi
    401174:	eb 21                	jmp    401197 <phase_6+0xa3>
    
    401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
    40117a:	83 c0 01             	add    $0x1,%eax
    40117d:	39 c8                	cmp    %ecx,%eax
    40117f:	75 f5                	jne    401176 <phase_6+0x82>
    401181:	eb 05                	jmp    401188 <phase_6+0x94>
    
    401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
    401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
    40118d:	48 83 c6 04          	add    $0x4,%rsi
    401191:	48 83 fe 18          	cmp    $0x18,%rsi
    401195:	74 14                	je     4011ab <phase_6+0xb7>
    
    401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
    40119a:	83 f9 01             	cmp    $0x1,%ecx
    40119d:	7e e4                	jle    401183 <phase_6+0x8f>
    40119f:	b8 01 00 00 00       	mov    $0x1,%eax
    4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
    4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
    
    4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
    ------ 新建数组，narr，首地址为0x20(%rsp)
    for (int i = 0; i < 6; ++i) {
    	int count = arr[i];
    	int address = 0x6032d0;
    	for (int j = 1; j < count; ++j)
    		address = *(address) + 0x8;
    	*(%rsp + 2*%rsi + 0x20) = address;
    	
    }
    ------
  ```

  从而新数组`narr`的首地址为%rsp+0x20，结束地址(narr.end())为%rsp+0x50（一个数据为一个地址，占据8个字节），输出一下0x6032d0处存储的地址值

  ![image-20210331111938939](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210331111938939.png)

继续往下分析汇编代码(终于快结束了orz..)

```assembly
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx
  
  4011bd:	48 8b 10             	mov    (%rax),%rdx
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  ------
  for (int i = 1; i < 6; ++i) {
  	*(narr[i - 1] + 0x8) = narr[i];
  }
  ------
```

最后的代码：

```assembly
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  4011d9:	00 
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
  
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
  4011e3:	8b 00                	mov    (%rax),%eax
  4011e5:	39 03                	cmp    %eax,(%rbx)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	callq  40143a <explode_bomb>
  
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
  4011f2:	83 ed 01             	sub    $0x1,%ebp
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
  ------
  for (int i = 1; i < 6; ++i) {
  	if (*narr[i - 1] < *narr[i])
  		bomb();
  }
  ------
```

继续回顾一下0x6032d0处的取值

![image-20210331114704652](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210331114704652.png)

总结前面的汇编代码我们可以得出：

- narr[1..6]存储{0x6032d0, 0x6032e0, 0x6032f0, 0x0x603300, 0x603310, 0x603320}这几个地址
- 这几个地址处存储的整数值，满足逆序排序的关系，即*(narr[i - 1]) >= *(narr[i])
- narr[i]由7-arr[i]映射而来

满足*(narr[i - 1] >= *(narr[i]))的数组顺序为

```
3			4			5			6			1			2
0x6032f0	0x603300	0x603310	0x603320	0x6032d0	0x6032e0
924			691			477			443			332			168
```

由于第一行的对应下标为7-arr[i]得来的（为什么不是0开头呢？因为变换前后都要满足0<=arr[i]<=6，所以准确来说arr[i]>=1），因此arr[i]，也就是phase6的**key**为

**4 3 2 1 6 5**

![image-20210331122034081](https://msigl62m-1258130641.cos.ap-shanghai.myqcloud.com/typora_image_2021_03_26/image-20210331122034081.png)

至此，bomblab就全部完成了。当然还有一个彩蛋关，如果各位时间充裕的话可以尝试一下，这里就挖个坑（可能不会填）。这个lab确实花费了我蛮长的准备时间，以及中间做的时候翻了好多别人的博客（有能力的话建议独立解决），从gdb调试器的用法到简单汇编代码的拆分和阅读，确实是一个不错的练习机会。

### 参考

> https://zhuanlan.zhihu.com/p/57977157
>
> https://zhuanlan.zhihu.com/p/104130161
>
> https://github.com/luy-0/CS-APP-LABs/blob/master/Lab-Answer_HB/Lab2-BombLab/L2-Bomb-Note.md


