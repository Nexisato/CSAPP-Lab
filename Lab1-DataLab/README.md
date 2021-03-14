---
title: CSAPP Lab (01)-Data Lab
date: 2021-03-05
categories:
	- 基础知识	
	- CSAPP

---

> 本文记录了CSAPP配套实验——Data Lab的详情WriteUp

- 运行环境：
  - Ubuntu 20.04 

---

### 0. Directions

- **Files**:

  Makefile	- Makes btest, fshow, and ishow
  README		- This file
  bits.c		- The file you will be modifying and handing in
  bits.h		- Header file
  btest.c		- The main btest program
  btest.h	- Used to build btest
  decl.c	- Used to build btest
  tests.c       - Used to build btest
  tests-header.c- Used to build btest
  dlc*		- Rule checking compiler binary (data lab compiler)	 
  driver.pl*	- Driver program that uses btest and dlc to autograde bits.c
  Driverhdrs.pm   - Header file for optional "Beat the Prof" contest
  fshow.c		- Utility for examining floating-point representations
  ishow.c		- Utility for examining integer representations

- **Modifying bits.c and checking it for compliance with dlc**

  **IMPORTANT**: Carefully read the instructions in the bits.c file before you start. These give the coding rules that you will need to follow if you want full credit.

  Use the dlc compiler (./dlc) to automatically check your version of bits.c for compliance with the coding guidelines:

         unix> ./dlc bits.c

  dlc returns silently if there are no problems with your code. Otherwise it prints messages that flag any problems.  Running dlc with the -e switch:

      	unix> ./dlc -e bits.c  

  causes dlc to print counts of the number of operators used by each function.

  Once you have a legal solution, you can test it for correctness using the ./btest program.

- **Testing with btest**

  The Makefile in this directory compiles your version of bits.c with additional code to create a program (or test harness) named btest.

  To compile and run the btest program, type:

      unix> make btest
      unix> ./btest [optional cmd line args]

  You will need to recompile btest each time you change your bits.c program. When moving from one platform to another, you will want to get rid of the old version of btest and generate a new one.  Use the commands:

      unix> make clean
      unix> make btest

  Btest tests your code for correctness by running millions of test cases on each function.  It tests wide swaths around well known corner cases such as Tmin and zero for integer puzzles, and zero, inf, and the boundary between denormalized and normalized numbers for floating point puzzles. When btest detects an error in one of your functions, it prints out the test that failed, the incorrect result, and the expected result, and then terminates the testing for that function.

  *Here are the command line options for btest*:

    unix> ./btest -h
    Usage: ./btest [-hg] [-r <n>] [-f <name> [-1|-2|-3 <val>]*] [-T <time limit>]
      -1 <val>  Specify first function argument
      -2 <val>  Specify second function argument
      -3 <val>  Specify third function argument
      -f <name> Test only the named function
      -g        Format output for autograding with no error messages
      -h        Print this message
      -r <n>    Give uniform weight of n for all problems
      -T <lim>  Set timeout limit to lim

  *Examples*:

    Test all functions for correctness and print out error messages:
    unix> ./btest

    Test all functions in a compact form with no error messages:
    unix> ./btest -g

    Test function foo for correctness:
    unix> ./btest -f foo

    Test function foo for correctness with specific arguments:
    unix> ./btest -f foo -1 27 -2 0xf

  Btest does not check your code for compliance with the coding guidelines.  Use dlc to do that.

- **Helper Programs**

  We have included the ishow and fshow programs to help you decipher integer and floating point representations respectively. Each takes a single decimal or hex number as an argument. To build them type:

      unix> make

  *Example usages*:

      unix> ./ishow 0x27
      Hex = 0x00000027,	Signed = 39,	Unsigned = 39
      
      unix> ./ishow 27
      Hex = 0x0000001b,	Signed = 27,	Unsigned = 27
      
      unix> ./fshow 0x15213243
      Floating point value 3.255334057e-26
      Bit Representation 0x15213243, sign = 0, exponent = 0x2a, fraction = 0x213243
      Normalized.  +1.2593463659 X 2^(-85)
      
      linux> ./fshow 15213243
      Floating point value 2.131829405e-38
      Bit Representation 0x00e822bb, sign = 0, exponent = 0x01, fraction = 0x6822bb
      Normalized.  +1.8135598898 X 2^(-126)

  ---

  ### 1. Writeup

  1. **bitXor**

     ```c
     /* 
      * bitXor - x^y using only ~ and & 
      *   Example: bitXor(4, 5) = 1
      *   Legal ops: ~ &
      *   Max ops: 14
      *   Rating: 1
      */
     int bitXor(int x, int y) {
       int com1 = x & y;
       int com2 = ~x & ~y; 
       int res = ~com1 & ~com2;
       return res;
     }
     ```

     布尔代数公式化简一下即可。

  2. **tmin**

     ```c
     /* 
      * tmin - return minimum two's complement integer 
      *   Legal ops: ! ~ & ^ | + << >>
      *   Max ops: 4
      *   Rating: 1
      */
     int tmin(void) {
       int res = 1;
       res <<= 31;
       return res;
      }
     ```

     补码表示法最小值，根据补码的定义，左移到最高位即可。

  3. **isTmax**

     ```C
     /*
      * isTmax - returns 1 if x is the maximum, two's complement number,
      *     and 0 otherwise 
      *   Legal ops: ! ~ & ^ | +
      *   Max ops: 10
      *   Rating: 1
      */
     int isTmax(int x) {
       int tmin = x + 1;//10000000
       x = x + tmin; //11111111
       x = ~x;//00000000
       tmin = !tmin;//0, if x==0xffffffff, tmin = 1
       x = x + tmin;//0 , if x == 0xffffffff, x = 1
       return !x;
     }
     ```

     tmax=0x7FFFFFFF

     tmin=0x10000000=tmax+1

     对于0xFFFFFFFF，tmin=0x00000000，注意到此时对tmin取**逻辑非！**得到结果为1；若x为tmax则对tmin进行**！**运算后得到的结果应当为0

  4. **allOddBits**

     ```C
     /* 
      * allOddBits - return 1 if all odd-numbered bits in word set to 1
      *   where bits are numbered from 0 (least significant) to 31 (most significant)
      *   Examples allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
      *   Legal ops: ! ~ & ^ | + << >>
      *   Max ops: 12
      *   Rating: 2
      */
     int allOddBits(int x) {
       int base = 0xAA;
       base += base << 8;
     base += base << 16;
       int tmp = x & base; //tmp == base if satisfied
     int res = !(tmp ^ base);
       return res;
     }
     ```

  5. **negate**

     ```c
     /* 
      * negate - return -x 
      *   Example: negate(1) = -1.
      *   Legal ops: ! ~ & ^ | + << >>
      *   Max ops: 5
      *   Rating: 2
      */
      int negate(int x) {
       /*Definition of two's complement*/
     int res = ~x + 1;
       return res;
     }
     ```

  6. **isAsciiDigit**

     ```C
     /* 
      * isAsciiDigit - return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
      *   Example: isAsciiDigit(0x35) = 1.
      *            isAsciiDigit(0x3a) = 0.
      *            isAsciiDigit(0x05) = 0.
      *   Legal ops: ! ~ & ^ | + << >>
      *   Max ops: 15
      *   Rating: 3
      */
     int isAsciiDigit(int x) {
       /*
       x - 0x30 >= 0 && 0x39 - x >= 0
       move right 31 bits to check if opr_res >= 0
       */
       int a = ~0x30 + 1;
       int l = x + a;
       x = ~x + 1;
       int r = 0x39 + x;
     int res = !(l >> 31) & !(r >> 31);
       return res;
     }
     ```

  7. **conditional**

     ```c
     /* 
      * conditional - same as x ? y : z 
      *   Example: conditional(2,4,5) = 4
      *   Legal ops: ! ~ & ^ | + << >>
      *   Max ops: 16
      *   Rating: 3
      */
     int conditional(int x, int y, int z) {
       /*
       C语言默认类型执行逻辑左移和算术右移
     */
       x = (!x << 31) >> 31;//produce 0x0 or 0xffffffff
       int res = (y & ~x) | (z & x);
       return res;
     }
     ```

  8. **isLessOrEqual**

     ```C
     /* 
      * isLessOrEqual - if x <= y  then return 1, else return 0 
      *   Example: isLessOrEqual(4,5) = 1.
      *   Legal ops: ! ~ & ^ | + << >>
      *   Max ops: 24
      *   Rating: 3
      */
     int isLessOrEqual(int x, int y) {
       /*
       1. judge the sign of two nums
       2. x_sign < 0 and y_sign > 0
       OR x_sign the same as y_sign and y-x >= 0 (which means (y-x)>>31 == 0)
       */
       int x_sign = x >> 31;
       int y_sign = y >> 31;
       int res = (x_sign & !y_sign) | (!(x_sign ^ y_sign) & !((y + ~x + 1) >> 31));
       return res;
     }
     ```

  9. **logicalNeg**

     ```c
     /* 
      * logicalNeg - implement the ! operator, using all of 
      *              the legal operators except !
      *   Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
      *   Legal ops: ~ & ^ | + << >>
      *   Max ops: 12
      *   Rating: 4 
      */
     int logicalNeg(int x) {
       /*
       neg: sign bit is not 0
       pos: plus tmax to overflow except 0 
       get res of 1 for not 0, and 0 for 0
       */
       int tmax = ~(1 << 31);
       int sign = (x >> 31) & 0x01;
       int res = sign | (((x + tmax) >> 31) & 0x01);
       res = res ^ 1;
       return res;
     }
     ```

  10. **howmanyBits**

      ```c
      /* howManyBits - return the minimum number of bits required to represent x in
       *             two's complement
       *  Examples: howManyBits(12) = 5
       *            howManyBits(298) = 10
       *            howManyBits(-5) = 4
       *            howManyBits(0)  = 1
       *            howManyBits(-1) = 1
       *            howManyBits(0x80000000) = 32
       *  Legal ops: ! ~ & ^ | + << >>
       *  Max ops: 90
       *  Rating: 4
       */
      int howManyBits(int x) {
        /*
        for positive num: the last bit of 1 plus the sign bit
        for negative num: the last bit of 0
        */
        int sign = x >> 31;
        x = (sign & ~x) | (~sign & x);//complement the bit
        int b16, b8, b4, b2, b1, b0;
        b16 = !(!(x >> 16)) << 4;//to check if the highest 16 bits include '1' bit, if including, then x move 16 bit to the R direction
        x = x >> b16;
        b8 = !(!(x >> 8)) << 3;
        x = x >> b8;
        b4 = !(!(x >> 4)) << 2;
        x = x >> b4;
        b2 = !(!(x >> 2)) << 1;
        x = x >> b2;
        b1 = !(!(x >> 1));
        x = x >> b1;
        b0 = x;
        int res = b16 + b8 + b4 + b2 + b1 + b0 + 1;
        return res;
      }
      ```

      太菜了orz...这题看的题解。

  11. **floatScale2**

      ```c
      /*  
       * floatScale2 - Return bit-level equivalent of expression 2*f for
       *   floating point argument f.
       *   Both the argument and result are passed as unsigned int's, but
       *   they are to be interpreted as the bit-level representation of
       *   single-precision floating point values.
       *   When argument is NaN, return argument
       *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
       *   Max ops: 30
       *   Rating: 4
       */
      unsigned floatScale2(unsigned uf) {
        int exp = (uf & 0x7f800000) >> 23;
        int sign = uf & (1 << 31);
        if (exp == 255) 
          return uf; // NaN
        if (exp == 0)
          return (uf << 1) | sign;//not regular
        exp += 1;
        if (exp == 255)
          return sign | 0x7f800000;//add to not regular
        return (uf & 0x807fffff) | (exp << 23);
      }
      ```

      

  12. **floatFloat2Int**(没太看懂)

      ```c
      /* 
       * floatFloat2Int - Return bit-level equivalent of expression (int) f
       *   for floating point argument f.
       *   Argument is passed as unsigned int, but
       *   it is to be interpreted as the bit-level representation of a
       *   single-precision floating point value.
       *   Anything out of range (including NaN and infinity) should return
       *   0x80000000u.
       *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
       *   Max ops: 30
       *   Rating: 4
       */
      int floatFloat2Int(unsigned uf) {
        unsigned exp = ((uf & 0x7f800000) >> 23) - 127;
        unsigned sign = uf >> 31;
        unsigned frac = uf & 0x007fffff | 0x00800000;
        if (!(uf & 0x7fffffff))
          return 0;
        if (exp > 31) 
          return 0x80000000u;
        if (exp < 0)
          return 0;
        if (exp > 23)
          frac <<= (exp - 23);
        else
          frac >>= (23 - exp);
        if (!((frac >> 31) ^ sign))
          return frac;
        else if (frac >> 31)
          return 0x80000000;
        else
          return ~frac + 1;
      }
      ```

      ~~此题看的题解，我太菜了orz~~

      我们将浮点数的符号、阶码、尾数分别拆开

      - 若阶码E部分大于31小于0，可以直接返回异常值
      - 对frac直接移位，再判断是否溢出。

  13. **floatPower2**

      ```c
      /* 
       * floatPower2 - Return bit-level equivalent of the expression 2.0^x
       *   (2.0 raised to the power x) for any 32-bit integer x.
       *
       *   The unsigned value that is returned should have the identical bit
       *   representation as the single-precision floating-point number 2.0^x.
       *   If the result is too small to be represented as a denorm, return
       *   0. If too large, return +INF.
       * 
       *   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while 
       *   Max ops: 30 
       *   Rating: 4
       */
      unsigned floatPower2(int x) {
        int INF = 0xff << 23;
        int exp = x + 127;
        if (exp <= 0)
          return 0;
        if (exp >= 255)
          return INF;
        return (exp << 23) & (0x7f800000);
      }
      ```

      浮点数的定义如下
      $$
      V = 2^E \times M
      $$
      对于非规格化数值来讲，设定`M=0`即可，因此我们只需要关注阶码E，加上Bias偏置值即可。

  
