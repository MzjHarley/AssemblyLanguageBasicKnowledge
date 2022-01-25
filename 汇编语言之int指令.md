---
title: 汇编语言之int指令
date: 2021-08-20 17:59:10
author: MZJ
---
# int指令
## 引言
int格式：int n，n为中断类型码。它的过程是引发中断过程。
## int
CPU执行int N指令，相当于引发一个N号中断的中断过程，执行过程如下：
>取得中断类型码
>标志寄存器入栈，TF=0，IF=0
>CS、IP入栈
>(CS)=(N * 4+2)，(IP)=(N * 4)
>然后转去执行N号中断的中断处理程序

**可以在程序中使用int指令调用任何一个中断的中断处理程序。**  
一般情况下，系统将一些具有一定功能的子程序，以中断处理程序的方式提供给应用程序调用。  
以后我们将中断处理程序称为中断例程。
## 编写可供应用程序调用的中断例程
我们通过两个实例来讨论
### 实例一：编写、安装中断7CH的中断例程，实现求word型数据的平方
功能：求word型数据的平方  
参数：(ax)=要计算的数据  
返回值：dx、ax中存放结果的高16位和低16位  
应用举例：求2 * 3456^2
>程序框架：  
>assume cs:code  
>code segment  
>start:  
>>mov ax,3456  
>>int 7CH  
>>add ax,ax  
>>adc dx,dx  
>>mov ax,4C00H  
>>int 21H  
>>
>code ends  
>end start

我们主要完成以下三部分工作：  
1.编写实现求平方功能的程序  
2.安装程序，将其安装到0000:0200处  
3.设置中断向量表，在7CH表项处登记程序的入口地址
#### 安装中断例程如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/1.png)  
### 实例二：编写、安装中断7CH的中断例程，实现将一个全是字母以0结尾的字符串转换为大写
功能：将一个全是字母以0结尾的字符串转换为大写  
参数：ds:si指向字符串  
应用举例：将data段中的字符串转换为大写。
#### 安装中断例程如下
```assembly
assume cs:code
code segment
start:
;=========安装capital==========
mov ax,cs
mov ds,ax
mov si,offset capital
mov ax,0
mov es,ax
mov di,200H
mov cx,offset capitalend - offset capital
cld
rep movsb
;=======设置中断向量表========
mov ax,0
mov es,ax
mov word ptr es:[4*7cH],200H
mov word ptr es:[4*7cH+2],0
;========================
mov ax,4C00h
int 21H
;==========编写capital=======
capital:
            mov cl,[si]
            mov ch,0
            jcxz ok
            and byte ptr [si],11011111B
            inc si
            jmp short capital
ok:         iret
capitalend:nop
;========================
code ends
end start
```
## 对int、iret和栈的深入理解
用7CH中断例程完成loop指令的功能。  
loop s的执行需要啷个信息，循环次数和到s的位移，所以7CH中断例程要完成loop指令的功能也需要这两个信息作为参数。  
我们用cx存放循环次数，bx存放位移。
### 应用举例：在屏幕中间显示80个‘!’
>该程序框架  
>assume cs:code  
>code segment  
>start:  
>mov ax,0B800H  
>mov es,ax  
>mov di,12 * 160  
>mov bx,offset s - offset se  
>mov cx,80  
>s:  
>>mov byte ptr es:[di],'!'  
>>add di,2  
>>int 7CH 
>> 
>se:nop  
>mov ax,4c00H  
>int 21H  
>code ends  
>end start

在上面的程序框架中，用int 7CH调用7CH中断例程进行转移，用bx传递转移的位移。  
为了模拟loop指令，7CH中断例程中应该具备两个功能：dec cx；如果(cx)!=0，转到标号s处执行，否则向下执行。
#### 如何实现7CH中断例程到目的地址的转移？
转到标号s处，应该知道标号s的段地址和偏移地址，用它们来设置CS和IP。  
那么如何得到s的段地址和偏移地址以及如何用它们设置CS和IP？  
int 7CH引发中断过程后，进入7CH中断例程，在中断过程中，当前的标志寄存器、CS和IP都要压栈。  
此时压入的CS和IP中的内容，分别是调用程序的段地址(可以说是标号S的段地址)和int 7CH后一条指令的偏移地址(标号se的偏移地址)  
在中断例程中，可以通过栈获得s的段地址和se的偏移地址，用标号se的偏移地址加上bx中存放的转移位移就可以得到s偏移地址。  
可以利用iret指令将栈中的se偏移地址加上bx中的转移位移，则栈中的se偏移地址就变成了s的偏移地址。再使用iret指令用栈中的内容设置CS和IP，从而转移到标号s处。
#### 7CH中断例程
```assembly
lp:
    push bp
    mov bp,sp
    dec cx
    jcxz lpret
    add [bp+2],bx
lpret:
    pop bp
    iret
```
## BIOS和DOS中断例程
在系统板的ROM中存放着一套程序，称为BIOS。
>BIOS主要包含以下内容：  
>1.硬件系统检测和初始化程序  
>2.内部中断和外部中断的中断例程  
>3.用于对硬件设备进行I/O操作的中断例程  
>4.其他和硬件系统相关的中断例程

操作系统DOS也提供了中断例程。和硬件设备相关的DOS中断例程，一般都调用了BIOS中断例程。  
程序员在编程时，可以用int指令直接调用BIOS和DOS所提供的中断例程，来完成某些工作。
### BIOS和DOS中断例程的安装过程
>1.开机后，CPU一加电，初始化CS:IP为FFFF:0000，此处有一条跳转指令(只读，不可改变)，CPU执行该指令后转去**执行BIOS中的硬件系统检测和初始化程序**。  
>2.初始化程序将建立支持BIOS所支持的中断向量，即：**将BIOS提供的中断例程的入口地址登记在中断向量表中**。  
>3.硬件系统检测和初始化完成后，**调用int 19h进行操作系统(DOS)的引导**，此后将计算机交给操作系统控制。  
>4.DOS启动后，除完成其他工作，还将它所提供的中断例程装入内存并建立相应的中断向量。

## BIOS中断例程的应用
int 10h中断例程是BIOS提供的中断例程，其中包含了多个和屏幕输出相关的子程序。  
一般来说，一个供程序员调用的中断例程往往包含多个子程序，中断例程内部用传递进来的参数来决定执行哪个子程序。BIOS和DOS提供的中断例程，都**用ah来传递内部子程序的编号**。  
>举例：int 10h中断例程设置光标位置功能  
>mov ah,2      ；ah来传递内部子程序的编号，表示调用第10H号中断例程的2号子程序。功能为设置光标位置，提供行号、列号和页号作为参数。  
>mov bh,0      ；设置光标位置在第0页  
>mov dh,5      ；设置光标位置在第5行  
>mov dl,12     ；设置光标位置在第12列  
>int 10h  
>举例：int 10h中断例程在光标位置显示字符功能  
>mov ah,9 ；ah来传递内部子程序的编号，表示调用第10H号中断例程的9号子程序。功能为在光标位置显示字符，提供字符，属性，页号，个数作为参数。  
>mov al,'a'  ；al存放要显示的字符  
>mov bl,7 ；bl存放要显示字符的属性  
>mov bh,0 ；第0页  
>mov cx,3 ；字符重复个数  
>int 10h

### 课程练习：在屏幕5行12列显示3个红底高亮闪烁绿色的‘a’
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/2.png)  
运行如下：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/3.png)
## DOS中断例程应用
int 21H中断例程是DOS提供的中断例程，其中包含了DOS提供给程序员在编程时调用的子程序。  
我们一直使用的都是21H中断例程的4cH号子程序，它的功能为程序返回。
>mov ah,4cH ；调用21H号中断例程的4ch号子程序，提供返回值作为参数。  
>mov al,0 ；返回值为0  
>int 21H ；通常我们简写成mov ax,4c00H,int 21H  

举例：int 21H中断例程在光标位置显示字符串的功能  
ds:dx指向字符串，要显示的字符串需以“$”作为结束符  
mov ah,9 功能号9，表示在光标位置显示字符串  
int 21H
### 编程实现在屏幕5行12列显示字符串“welcome to masm!”
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/4.png)  
运行如下：  
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/5.png)
## 实验13
### 编写并安装int 7cH中断例程，功能为显示一个以0结尾的字符串，中断例程安装在0:200处
参数：(dh)=行号，(dl)=列号，(cl)=颜色，ds:si指向字符串首地址。
#### 调用程序
```assembly
assume cs:code
data segment
db 'welcome to masm!',0
data ends
code segment
start:mov dh,10
    mov dl,10
    mov cl,2
    mov ax,data
    mov ds,ax
    mov si,0
    int 7Ch
    mov ax,4c00H
    int 21H
code ends
end start
```
#### 7CH中断例程安装
```assembly
assume cs:code 
code segment
start:
;==================安装display_string========================
      mov ax,cs
      mov ds,ax
      mov si,offset display_string
      mov ax,0
      mov es,ax
      mov di,200H
      mov cx,offset display_string_end - offset display_string
      cld
      rep movsb
;==================登记display_string========================
      mov ax,0
      mov es,ax
      mov word ptr es:[4*7CH],200h
      mov word ptr es:[4*7CH+2],0
;===========================================================
      mov ax,4c00H
      int 21H
;==================编程display_string========================
display_string:
              mov cx,0
              mov ax,0B800H
              mov es,ax
              mov di,160*9+20
display:      mov cl,[si]
              jcxz ok
              mov es:[di],cl
              mov byte ptr es:[di+1],2
              add di,2
              inc si
              jmp short display
ok:           iret
display_string_end:nop
;==========================================================
code ends
end start
```
#### 结果如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/6.png)
### 编写并安装int 7cH中断例程，功能为完成loop指令功能，在屏幕中间显示80个"!"
参数：(cx)=循环次数，(bx)=位移
#### 调用程序
```assembly
assume cs:code
code segment
start:
      mov ax,0B800H
      mov es,ax
      mov di,160*12
      mov bx,offset s-offset se
      mov cx,80
s:    
      mov byte ptr es:[di],'!'
      add di,2
      int 7cH
se:   nop
      mov ax,4c00H
      int 21H
code ends
end start
```
#### 7CH中断例程安装
```assembly
assume cs:code 
code segment
start:
;==================安装display========================
      mov ax,cs
      mov ds,ax
      mov si,offset display
      mov ax,0
      mov es,ax
      mov di,200H
      mov cx,offset display_end - offset display
      cld
      rep movsb
;==================登记display========================
      mov ax,0
      mov es,ax
      mov word ptr es:[4*7CH],200h
      mov word ptr es:[4*7CH+2],0
;====================================================
      mov ax,4c00H
      int 21H
;==================编程display========================
display:
          push bp
          mov bp,sp
          dec cx
          jcxz ok
          add [bp+2],bx
ok:       pop bp
          iret
display_end:nop
;======================================================
code ends
end start
```
#### 结果如下
![contents](https://github.com/MzjHarley/AssemblyLanguageBasicKnowledge/blob/main/img/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E4%B9%8Bint%E6%8C%87%E4%BB%A4/7.png)
# int指令结束
