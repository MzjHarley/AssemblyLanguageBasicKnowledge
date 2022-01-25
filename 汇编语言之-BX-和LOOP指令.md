---
title: '汇编语言之[BX]和LOOP指令'
date: 2021-07-14 16:05:02
author: MZJ
---
# [BX]和LOOP指令
## 前言
### [bx]和内存单元的描述
MOV AX,[0]  将一个内存单元的内容送入AX，该内存单元的内容为2个字节，存放一个字，偏移地址为0，段地址在ds中。  
MOV AL,[0]  将一个内存单元的内容送入AL，该内存单元的内容为1个字节，存放一个字节，偏移地址为0，段地址在ds中。  
**要完整描述一个内存单元，需要两种信息：内存单元的地址、内存单元的长度(可以由具体指令中的其他操作对象(比如寄存器）指出)。**  
MOV AX,[bx]  将一个内存单元的内容送入AX，该内存单元的内容为2个字节，存放一个字，偏移地址为bx的内容，段地址在ds中。  
MOV AL,[bx]  将一个内存单元的内容送入AL，该内存单元的内容为1个字节，存放一个字节，偏移地址为bx的内容，段地址在ds中。
### loop
英文”循环“的含义，这个指令和循环有关。
### 描述性符号"()"
为了描述上的简洁，我们将使用一个描述性的非符号"()"来表示一个寄存器或内存单元的地址。  
（X）表示X的内容
#### （X）应用
1.ax内容为0010H，可以这样描述：（ax）=0010H  
2.2000：1000出的内容为0010H，可以这样描述：（21000H）=0010H  
3.对于MOV AX,[2]，可以这样描述：(AX)=( (DS) * 16 +2)  
4.对于MOV [2],AX，可以这样描述：( (DS) * 16 +2)=(AX)  
5.对于ADD AX,2，可以这样描述：(AX)=(AX)+2  
6.对于ADD AX,BX，可以这样描述：(AX)=(AX)+(BX)  
7.对于 PUSH AX，可以这样描述：(SP)=(SP)-2；((SS) * 16+(SP))=(AX)  
8.对于 POP AX，可以这样描述：(AX)=((SS) * 16+(SP))；(SP)=(SP)+2
### 约定符号idata表示常量
## [BX]
MOV AX,[BX]  
功能：BX存放的数据作为一个偏移地址EA，段地址SA默认在DS中，将SA:EA处的数据送入AX中即(AX)=( (DS) * 16 +（BX）)
## LOOP指令
LOOP指令的格式为：loop  标号。
### CPU执行loop指令时进行两步操作
1.（CX）=（CX）-1  
2.判断CX的值，不为0则转至标号处执行程序，如果为0则跳出循环，向下执行。  
注意：  
CX影响loop指令的执行结果，CX存放循环次数。
### 接下来写一个程序助你理解
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/1.png)  
在该程序中，标号代表一个地址，s就是一个标号，他实际上标识了一个地址，这个地址处有一条指令add ax,ax  
loop s:    
CPU执行loop  s时进行两步操作  
1.（CX）=（CX）-1  
2.判断CX的值，不为0则转至标号s所标识的地址处执行程序，如果为0则执行下一条指令（MOV AX,4C00H）。
### CX和LOOP相配合实现循环的三个要点
1.在CX中存放循环次数  
2.loop指令中标号所标识的地址要在前面  
3.要循环执行的程序段，要写在标号和loop指令的中间
### CX和LOOP相配合实现循环程序框架
```
mov cx,循环次数
s：
循环执行的程序段
loop s
```
## 在Debug中跟踪用LOOP指令实现的循环程序
计算ffff:0006单元中的数 * 3，结果存储在dx中  
1.运算后的结果是否会超出dx的存储范围？ 不会  
2.用循环累加实现乘法，使用哪个寄存器累加？将ffff:6单元内容赋给ax，用dx累加  
3.ffff:6单元是一个字节单元，ax是一个16位的寄存器，数据长度不一样如何赋值？（ah）=0；（al）=（ffff6H）  
程序如下：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/2.png)  
在汇编源程序中，数据不能以字母开头  
用debug对loop2.exe进行跟踪。  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/3.png)  
循环程序段从cs:0012开始，cs:0012前面的指令，我们不想再一步步跟踪，希望能一次执行完，然后从cs:0012处开始跟踪。  
g 0012表示执行程序到当前代码段的0012处。  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/4.png)  
如果循环次数过多，我们只跟踪了两次循环过程即可确认循环程序段在逻辑上是正确的，我们不想再继续跟踪循环过程了，怎么样才能让程序将剩余的循环一次全部执行完呢？  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/5.png)
## Debug和汇编编译器Masm对指令的不同处理
### 在debug编程中编程实现
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/6.png)
### 用masm编程
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/7.png)  
但在masm编译器中解释为  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/8.png)  
编译器将[idata]解释为idata，这样是错误的。  
如果我们非要在masm编程中正确给al、bl、cl、dl赋值，则可以将偏移地址送入bx中，用[bx]的方式访问内存单元,如访问2000：0内存单元：mov ax,2000；mov ds,ax；mov bx，0；mov  al，[bx]  
这样比较麻烦，我们同样可以在[idata]前显式给出段地址所在的段寄存器比如访问2000：0内存单元：mov ax,2000；mov ds,ax； mov al，ds:[0]
## loop和[bx]的联合使用
计算ffff:0 ~ ffff:b单元中数据和，结果存储在dx中。  
1.运算后的结果是否会超出dx的存储范围？ 不会  
2.累加时，我们有两种方法：  
2.1（dx）=（dx）+内存中的8位数据，该方法问题两个运算对象类型不匹配  
2.2 （dl）=（dl）+内存中的8位数据，该方法问题结果可能越界  
如何解决两个看似矛盾的问题？用一个16位的寄存器作中介。将内存中的8位数据赋给一个16位寄存器，在将ax中的数据加到dx中，从而使两个运算对象的类型匹配并且结果不会超界。
### 程序如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/9.png)  
inc bx相当于bx++  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/10.png)
## 段前缀
出现在访问内存单元的指令中，用于显式的指明内存单元的段地址的”ds:“   "es:" "ss:"  "cs:",在汇编语言中被称为段前缀，如mov ax，cs:[0]  
## 一段安全的空间
1.我们可以直接向一段内存中写入内容；  
2.这段内存空间不应存放系统或其他程序的代码或数据，否则写入操作很可能引发错误；  
3.DOS方式下，一般情况，0:200 ~ 0:2ff空间中没有系统或其他程序的代码或数据；  
4.以后我们需要直接向一段内存中写入数据时，就使用0:200~0:2ff这段空间
## 段前缀的使用
将内存单元ffff:0 ~ ffff:b中的数据复制到0:200 ~ 0:20b中。  
## 程序如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/11.png)  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8B-BX-%E5%92%8CLOOP%E6%8C%87%E4%BB%A4/12.png)
# [BX]和LOOP指令结束
