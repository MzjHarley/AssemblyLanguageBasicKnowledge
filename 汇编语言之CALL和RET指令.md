---
title: 汇编语言之CALL和RET指令
date: 2021-07-18 21:27:23
author: MZJ
---

# CALL和RET指令
## 引言
call和ret指令都是转移指令，它们都修改IP，或同时修改CS和IP。  
它们经常被共同用来实现子程序的设计，这一章，我们主要讲解call和ret指令的原理。
## ret和retf
### ret
ret指令用栈中的数据修改IP的内容从而实现近转移。  
CPU执行ret指令时，进行下面两步操作：  
（IP)=（（ss） * 16+（sp））  
（sp）=（sp）+2
### retf
retf指令用栈中的数据修改CS和IP的内容从而实现远转移。  
CPU执行retf指令时，进行下面四步操作：  
（IP)=（（ss） * 16+（sp））  
（sp）=（sp）+2  
（CS)=（（ss） * 16+（sp））  
（sp）=（sp）+2
### 总结
可以看出如果我们用汇编语法来解释ret和retf指令，则  
CPU执行ret指令时，相当于执行POP IP  
CPU执行retf指令时相当于执行POP IP;POP CS
## call指令
call指令经常跟ret指令配合使用，因此CPU执行call指令，进行两步操作：  
1.将当前的IP或CS和IP压入栈中  
2.转移（jmp）  
call指令不能实现短转移，除此之外，call指令实现转移的方法和jmp指令的原理相同。
## 依据位移进行转移的call指令
call 标号：将当前IP压栈，转到标号处执行
CPU执行此种格式call指令时，进行如下操作：  
1.（SP）=（SP）-2；（（SS） * 16+（SP））=（IP），相当于PUSH IP  
2.（IP）=（IP）+16位位移，相当于jmp near 标号  
16位位移=标号处的地址-call指令后第一个字节的地址  
16位位移范围为-32768 ~ 32767，用补码表示  
16位位移由编译程序在编译时算出
## 转移的目的地址在指令中的call指令
call far ptr 标号实现的是段间转移。  
CPU执行此种格式call指令时，进行如下操作：  
1.（SP）=（SP）-2；（（SS） * 16+（SP））=（CS），相当于PUSH CS  
2.（SP）=（SP）-2；（（SS） * 16+（SP））=（IP），相当于PUSH IP  
3.（CS）=标号所在段地址；（IP）=标号所在偏移地址 ，相当于jmp far ptr 标号
## 转移地址在寄存器中的call指令
call 16位寄存器  
功能：  
（SP）=（SP）-2；（（SS） * 16+（SP））=（IP），相当于PUSH IP  
（IP）=（16位寄存器），相当于jmp 16位寄存器
## 转移地址在内存中的call指令
转移地址在内存中的call指令有两种格式
### call word ptr 内存单元地址
汇编语法解释：  
PUSH IP  
jmp word ptr 内存单元地址
### call dword ptr 内存单元地址
汇编语法解释：  
PUSH CS  
PUSH IP  
jmp dword ptr 内存单元地址
## call和ret的配合使用
我们可以写一个具有特定功能的程序段，我们称之为子程序。在需要时用call指令转去执行。执行完子程序后，可以在子程序的后面用ret指令，用栈中的数据设置IP的值，从而转到call指令后面的指令继续执行。  
子程序框架如下：  
标号：  
指令  
ret  
具有子程序的源程序框架如下：
```assembly
assume cs:code
code segment
main:
      ···
      ···
      call sub1
      ···
      ···
      mov ax,4c00H
      int 21H
sub1:
     ···
     ···
     call sub2
     ···
     ···
     ret
sub2:
     ···
     ···
     ···
     ret
code ends
end main
```
## mul指令
mul指令是乘法指令。
### 使用mul时注意3点
1.两个相乘的数，要么都是8位，要么都是16位。  
2.如果是8位乘法，那么这两个相乘的数，一个放在默认AL中，另一个放在8位reg或者内存单元中，相乘的结果默认放在AX中。  
3.如果是16位乘法，那么这两个相乘的数，一个放在默认AX中，另一个放在16位reg或者内存单元中，相乘的结果高位默认放在DX中，低位默认放在AX中。  
使用格式如下：  
**mul reg  
mul 内存单元**  
内存单元可以以不同的寻址方式给出。
### mul举例
#### mul byte ptr ds:[0]
含义：  
（ax）=（al） * （（ds） * 16+0）

#### mul word ptr ds:[bx+si+8]
含义：  
（ax）=（ax） * （（ds） * 16+（bx）+（si）+8）结果的低16位  
（dx）=（ax） * （（ds） * 16+（bx）+（si）+8）结果的高16位

#### 计算100 * 10000
mov ax,100  
mov bx,10000  
mul bx

#### 计算100 * 10
mov al,10  
mov bl,100  
mul bl 
## 参数和结果传递的问题
子程序一般都要根据提供的参数处理一定的事物，处理后，将结果提供给调用者。  
参数和结果传递的问题，实际上是在探讨应该如何存储子程序需要的参数何产生的返回值。  
用寄存器来存储参数和结果是最常用的方法。对于存放参数的寄存器和存放结果的寄存器，调用者和子程序的读写恰恰相反。：  
调用者将参数送入参数寄存器，从结果寄存器中取得返回值。  
子程序从参数寄存器中取得参数，将返回值送入结果寄存器。
### 编程计算data段中的第一组数据的3次方，结果保存在后面一组dword单元中
数据段如下  
data segment  
dw 1,2,3,4,5,6,7,8  
dd 0,0,0,0,0,0,0,0  
data ends
#### 程序如下
##### 一
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/1.png)  
##### 二
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/2.png)
## 批量数据的传递
前面的例程中，子程序只有一个参数，它放在寄存器中；如果有两个参数，那么可以用两个寄存器来存放。如果传递的数据有3个、4个甚至更多直至N个，如何存放呢？  
寄存器的数量终究有限，我们不可能简单的用寄存器来存放多个需要传递的数据。对于返回值，也有同样的问题。  
在这种时候，我们将批量数据放在内存中，然后将它们所在的内存空间的首地址放在寄存器中，传递给需要它们的子程序。对于具有批量数据的返回结果，也可以用同样的方法。
### 编程将data段中的字符串转化为大写
数据段如下  
data segment  
db 'conversation'  
data ends  
程序如下：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/3.png)  
除了寄存器、内存传递参数外，还有一种通用的方法使用栈来传递参数。
## 寄存器冲突的问题
### 一、设计一个子程序
功能：将一个全是字母，以0结尾的字符串，转化为大写。  
程序要处理的字符串以0作为结尾符，这个字符串可以定义：db 'conversation',0
#### 程序如下
##### A方式 
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/4.png)
##### B方式
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/5.png)
### 二、设计一个子程序将data段中的字符全部转化为大写
数据段如下：  
data segment  
db 'word',0  
db 'unix',0  
db 'wind',0  
db 'good',0  
data ends
#### 程序如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/6.png)  
上面的程序存在问题，在于cx的使用，子程序cx值覆盖了外层循环cx的值，导致程序错误。  
解决这个问题的简捷方法是，在子程序开始前将子程序中用到的所有寄存器中的内容都保存起来，在子程序返回前再恢复。可以用栈来保存寄存器中的内容。  
故程序如下
```assembly
assume cs:code,ds:data,ss:stack
data segment
db 'word',0
db 'unix',0
db 'wind',0
db 'good',0
data ends'
stack segment
dw 8 dup(0)
stack ends
code segment
start:
mov ax,data
mov ds,ax
mov ax,stack
mov ss,ax
mov sp,16
mov bx,0

mov cx,4
s:
mov si,bx
call s1
add bx,5
loop s

mov ax,4c00H
int 21H

s1:
push cx          ;将外层循环的cx值压栈保存起来，防止其被子程序CX值覆盖
push si
s2:
mov ch,0
mov cl,[si]
jcxz ok
and byte ptr [si],11011111B
inc si
jmp short s2
ok:
pop si
pop cx
ret

code ends
end start
```
## 实验十：编写子程序
### 显示字符串
#### 子程序描述
名称： show_str
功能：在指定位置用指定颜色，显示一个用0结束的字符串。  
参数：（dh）=行号（取值范围0 ~ 24），（dl）=列号（取值范围0 ~ 79）  
（cl）=颜色，ds:di指向字符串的首地址  
返回：无  
应用举例：在屏幕的8行3列，用绿色显示data段中的字符串。  
data segment  
db 'welcome to masm!',0  
data ends
#### 程序如下
##### A方式
该方式并不按题意，而是用前面所学知识解决。  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/7.png)  
程序运行结果如下：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/8.png)
##### B方式
该方式是按题意解答
```assembly
assume cs:code,ds:data
data segment
db 'welcome to masm!',0
data ends
code segment
start:
mov ax,data
mov ds,ax
mov si,0
mov dh,8
mov dl,3
mov cl,2H

call show_str
mov ax,4c00H
int 21H

show_str:
mov al,0A0H
dec dh                ;相当于sub dh,1
mul dh
mov bx,ax
mov al,2
mul dl
sub ax,2
add bx,ax
;求得8行3列的内存单元的偏移地址
mov ax,0B800H
mov es,ax
;求得8行3列的内存单元的段地址
;es:[bx]可表示8行3列的内存单元的地址

mov di,0
mov al,cl

mov ch,0
s:
mov cl,[si]           ;[si]指向'welcome to masm!',0
jcxz ok
mov es:[bx+di],cl
mov es:[bx+di+1],al
inc si
add di,2
jmp short s
ok:ret

code ends
end start
```
程序运行结果如下：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/9.png)
### 解决除法溢出问题
我们在用div指令做除法时，可能会出现除法溢出这个错误。  
除法溢出是指除法结果的商太大，超过寄存器所能存储的最大范围。  
由于有这样的问题，我们在进行除法运算时，要注意除数和被除数的值。  
如1000000/10不能用div指令来计算，我们用下面的子程序divdw解决。
#### 子程序描述
名称：divdw
功能：进行不会产生溢出的除法运算，被除数dword型，除数word型，结果为dword型  
参数：  
(ax)=dword型数据低16位  
(dx)=dword型数据高16位  
(cx)=除数  
返回：  
(ax)=结果低16位  
(dx)=结果高16位  
(cx)=余数  
应用举例：计算1000000/10  
提示:  
给出一个公式**X/N=int(H/N) * 65536+[rem(H/N) * 65536+L]/N**  
X：被除数；N：除数；H：X高16位；L：X低16位；int（）：取商；rem（）：取余  
该公式将可能会产生溢出的除法运算：X/N，转换为多个不会产生溢出的除法运算。  
公式中等号右边的所有除法运算都可以用div指令来做，肯定不会导致除法溢出。
#### 程序如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/10.png)
### 数值显示
#### 子程序描述
名称：dtoc
功能：将word型数据变为十进制的字符串，字符串以0结尾。  
参数：(ax)=word型数据，ds:si指向字符串的首地址  
返回：无  
应用举例：编程，将数据12666以十进制的形式在屏幕8行3列，用绿色显示出来。在显示时我们调用子程序show_str。
#### 程序分析
首先明确目的：**将12666的ASCII码放入8行3列的显示缓冲区即可**  
我们不能直接将12666的ASCII码放入8行3列的显示缓冲区，我们必须通过数据段作为媒介去存储12666的ASCII码，再将数据段中的ASCII码放入8行3列的显示缓冲区。
#### 编程实现
##### A方式
这是我自己写的一种实现方式，没有按照题意。但同样是成功的。
```assembly
assume cs:code,ds:data

data segment
db 16 dup(0)
data ends

code segment 
start:
mov ax,data
mov ds,ax
mov ax,12666
mov si,1                                   ;[si]指向数据段


call dtoc                                   ;将12666的ASCII码放入数据段中

mov dh,8
mov dl,3
mov al,2
call show_str                           ;将数据段中的ASCII码放入8行3列的显示缓冲区

mov ax,4c00H
int 21H

dtoc:
mov dx,0
mov bx,10
div bx
add dx,30H
mov [si],dl
mov cx,ax
jcxz ok
inc si
jmp short dtoc                       
ok:ret
;执行完后data段有效内容为066621

show_str:
mov cl,al
mov al,0A0H
dec dh
mul dh
mov bx,ax

dec dl
mov al,dl
mov ch,2
mul ch
add bx,ax
mov ax,0B800H
mov es,ax

mov al,cl
s:
mov cl,[si]
mov es:[bx],cl
mov es:[bx+1],al
dec si
add bx,2
mov cx,si
jcxz yes
jmp short s
yes: ret

code ends
end start
```
运行结果：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/11.png)
##### B方式
按照题意的方式
```assembly
assume cs:code,ds:data
data segment
db 10 dup(0)
data ends

code segment
start:
mov ax,12666
mov bx,data
mov ds,bx
mov si,0
call dtoc

mov dh,8
mov dl,3
mov cl,2
call show_str
mov ax,4c00H
int 21H

dtoc:
mov dx,0
mov bx,10              ;这里用16位除法，防止除法溢出
div bx
add dx,30H
push dx
inc si
mov cx,ax
jcxz ok
jmp short dtoc 
ok: 
mov cx,si
mov di,0
s:
pop dx
mov [di],dl
inc di
loop s
ret

show_str:
mov al,160
dec dh
mul dh
mov bx,ax
mov al,2
dec dl
mul dl
add bx,ax
mov ax,0B800H
mov es,ax

mov al,cl
mov ch,0
mov si,0
s1:
mov cl,[si]
jcxz yes
mov es:[bx],cl
mov es:[bx+1],al
inc si
add bx,2
jmp short s1
yes: ret
 
code ends
end start
```
运行结果：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/12.png)
## 课程设计1
任务：将实验七中的Power idea公司的数据按照如图所示格式在屏幕上显示出来。  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8BCALL%E5%92%8CRET%E6%8C%87%E4%BB%A4/13.png)
### 程序分析
### 程序展示
;这里只写了一部分，来日方长。
```assembly
assume cs:code
;=======================================================
data segment
db '1975','1976','1977','1978','1979','1980','1981','1982','1983'
db '1984','1985','1986','1987','1988','1989','1990','1991','1992'
db '1993','1994','1995'
dd 16,22,382,1356,2390,8000,16000,24486,50056,97479,140417,197514
dd 345980,590827,803530,1183000,1843000,2759000,3753000,4649000,5937000
dw 3,7,9,13,28,38,130,220,476,778,1001,1442,2258,2793,4037,5635,8226
dw 11542,14430,15257,17800
data ends
table segment
db 21 dup ('year summ ne ?? ')       
table ends
string segment
db 10 dup(0)
string ends
stack segment
db 16 dup(0)
stack ends
;=======================================================
code segment
start:
mov ax,stack
mov ss,ax
mov sp,16
call put_data_into_table
mov ax,0B800H
mov es,ax
call show_year
call show_person
call show_average_income
mov ax,4c00H
int 21H
;==========================================
put_data_into_table:
mov ax,data
mov ds,ax
mov ax,table
mov es,ax
mov si,0
mov di,0
mov bx,0
mov cx,21
s1:
call put_year_into_table
call put_general_income_into_table
call put_person_into_table
call put_average_income_into_table
add si,4
add di,2
add bx,16
loop s1
ret
;========================
put_year_into_table:
mov al,[si+0]
mov es:[bx+0],al
mov al,[si+1]
mov es:[bx+1],al
mov al,[si+2]
mov es:[bx+2],al
mov al,[si+3]
mov es:[bx+3],al
ret
;========================
put_general_income_into_table:
mov ax,[84+si]
mov es:[bx+5],ax
mov ax,[86+si]
mov es:[bx+7],ax
ret
;========================
put_person_into_table:
mov ax,[di+168]
mov es:[bx+10],ax
ret
;========================
put_average_income_into_table:
mov ax,[si+84]
mov dx,[si+86]
div word ptr [di+168]
mov es:[bx+13],ax
ret
;========================
;==========================================
show_year:
mov ax,table
mov ds,ax
mov bx,640
mov si,0
mov cx,21
s2:
mov al,[si]
mov es:[bx],al
mov byte ptr es:[bx+1],2
mov al,[si+1]
mov es:[bx+2],al
mov byte ptr es:[bx+3],2
mov al,[si+2]
mov es:[bx+4],al
mov byte ptr es:[bx+5],2
mov al,[si+3]
mov es:[bx+6],al
mov byte ptr es:[bx+7],2
add bx,160
add si,16
loop s2
ret
;==========================================
show_person:
ret
;==========================================
show_average_income:
mov ax,table
mov ds,ax
mov si,0
mov bx,668
mov cx,21
s13:
mov ax
loop s13
ret
;==========================================
code ends
end start
```
# CALL和RET指令结束
