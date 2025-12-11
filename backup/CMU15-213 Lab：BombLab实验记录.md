# 前言
本实验所需的预备知识：
* 汇编基础
* gdb的基本使用

本实验需要我们拆除每个phase的炸弹，每个炸弹都有一个密码，我们需要通过输入正确的密码来拆除炸弹，如果输入错误，炸弹就会爆炸。
具体地，我们需要通过阅读源码和反编译出的汇编代码来分析程序的逻辑同时使用调试工具**gdb**来找到正确的密码。
将实验所需的代码导入WSL中，我们发现bomb.c由于缺少头文件`phases.h`而报错，这是因为实验省略了每个phase的细节，不过我们实际上也无需运行代码bomb.c，因为实验给出了一个已经编译好的bomb二进制程序，所以我们第一步需要做的，是把这个二进制程序反编译成我们能看懂的汇编代码：
```bash
objdump -d bomb > bomb.asm
```
根据实验文档我们知道，炸弹的密码需要我们通过文件`psol.txt`以传入参数的形式给出，也就是说在进入bomb的gdb调试界面后我们需要设置参数：`set args psol.txt`。此外，我们为了获取每个阶段的拆弹密码，肯定需要分别对phase_1到phase_6六个函数设置断点进行调试，为避免每次进入gdb都进行重复的操作，我们进行如下配置：
```bash
touch .gdbinit
mkdir -p ~/.config/gdb
echo "set auto-load safe-path /" > ~/.config/gdb/gdbinit 
```
其中.gdbinit编辑如下内容
```bash
set args psol.txt

# 为各个phase函数设置断点，用以观察其执行过程
# 如果做完了某个phase，可以将其注释掉，这样就不会进入该phase了
b phase_1
b phase_2
b phase_3
b phase_4
b phase_5
b phase_6

r
```
这样以来，每次调试直接`gdb ./bomb`即可
注意，原始实验版本会通过`sendmsg`函数向服务器发送“爆炸”信息逻辑从而更新学生的得分，此时在调试时为了防止爆炸时扣分，我们需要在.gdbinit中建立保护机制跳过sendmsg的步骤，不过我现在的版本为简化版本，并未在bomb.asm发现sendmsg函数，它是不包含网络通信部分的，而仅在本地打印爆炸信息。
# 拆炸弹
## phase1
先在bomb.asm中找到phase_1的汇编代码如下：
```asm
0000000000400ee0 <phase_1>:
  400ee0: 48 83 ec 08           sub    $0x8,%rsp 
  400ee4: be 00 24 40 00        mov    $0x402400,%esi
  400ee9: e8 4a 04 00 00        call   401338 <strings_not_equal>
  400eee: 85 c0                 test   %eax,%eax
  400ef0: 74 05                 je     400ef7 <phase_1+0x17>
  400ef2: e8 43 05 00 00        call   40143a <explode_bomb>
  400ef7: 48 83 c4 08           add    $0x8,%rsp
  400efb: c3                    ret
```
第一步的目的是进行栈的16字节对齐，可以忽略

> 在x86-64的System V AMD64 ABI（Linux、macOS等系统常用）中，前6个整数/指针参数通过寄存器传递，具体地：
> 	第一个参数--%rdi
> 	第二个参数--%rsi
> 	第三个参数--%rdx
> 	第四个参数--%rcx
> 	第五个参数--%r8
> 	第六个参数--%r9
> 	第七个及以后的参数--按从右到左的顺序压入栈

第二步是对%esi赋值，注意寄存器%rsi是过程调用中第二个参数存放的位置，而下面恰好是对`strings_not_equal`的函数调用，我们再来看main函数中phase_1附近的汇编代码：
```asm
  400e32:    e8 67 06 00 00           call   40149e <read_line>
  400e37:    48 89 c7                 mov    %rax,%rdi
  400e3a:    e8 a1 00 00 00           call   400ee0 <phase_1>
  400e3f:    e8 80 07 00 00           call   4015c4 <phase_defused>
  400e44:    bf a8 23 40 00           mov    $0x4023a8,%edi
```
我们发现在调用phase_1之前，%rdi就被赋值为%rax，而函数的返回值就保存在%rax中，恰好上个语句就是对`read_line`函数的调用，那么就说明psol.txt的第一行字符串被传入了%rdi，结合400ee9调用的函数名`strings_not_equal`，我们怀疑这里就是传入了两个字符串如果相等直接跳到400ef7以避开触发炸弹的`exploded_bomb`函数，为了验证这一点，我们阅读函数`strings_not_equal`的汇编代码（已给出注释）：
```asm
0000000000401338 <strings_not_equal>:
  401338:    41 54                    push   %r12 # 寄存器保护
  40133a:    55                       push   %rbp # 寄存器保护
  40133b:    53                       push   %rbx # 寄存器保护
  40133c:    48 89 fb                 mov    %rdi,%rbx # 将第一个参数（字符串s1）保存在%rbx中
  40133f:    48 89 f5                 mov    %rsi,%rbp # 将第二个参数（字符串s2）保存在%rbp中
  401342:    e8 d4 ff ff ff           call   40131b <string_length> # 将%rip压入栈，并计算%rdi（s1）代表的字符串的长度，返回值保存在%rax中
  401347:    41 89 c4                 mov    %eax,%r12d # 将s1的长度保存在%r12d中
  40134a:    48 89 ef                 mov    %rbp,%rdi # 将s2保存在%rdi中
  40134d:    e8 c9 ff ff ff           call   40131b <string_length> # 将%rip压入栈，并计算s2的长度
  401352:    ba 01 00 00 00           mov    $0x1,%edx # 将1保存在%edx中
  401357:    41 39 c4                 cmp    %eax,%r12d 
  40135a:    75 3f                    jne    40139b <strings_not_equal+0x63> # 如果s1和s2的长度不同跳转到40139b，函数返回1
  40135c:    0f b6 03                 movzbl (%rbx),%eax # 取s1当前位置的字符c1放入%eax中
  40135f:    84 c0                    test   %al,%al 
  401361:    74 25                    je     401388 <strings_not_equal+0x50> # 如果c1为'\0'跳转到401388，函数返回0
  401363:    3a 45 00                 cmp    0x0(%rbp),%al 
  401366:    74 0a                    je     401372 <strings_not_equal+0x3a> # 如果s2当前位置的字符c2等于c1跳转到401372
  401368:    eb 25                    jmp    40138f <strings_not_equal+0x57>
  40136a:    3a 45 00                 cmp    0x0(%rbp),%al
  40136d:    0f 1f 00                 nopl   (%rax)
  401370:    75 24                    jne    401396 <strings_not_equal+0x5e> # 如果s2当前位置的字符c2不等于c1，函数返回1
  401372:    48 83 c3 01              add    $0x1,%rbx
  401376:    48 83 c5 01              add    $0x1,%rbp # s1和s2的当前字符分别后移一位
  40137a:    0f b6 03                 movzbl (%rbx),%eax 
  40137d:    84 c0                    test   %al,%al
  40137f:    75 e9                    jne    40136a <strings_not_equal+0x32> # 如果s1当前字符不为'\0'跳转到40136a
  401381:    ba 00 00 00 00           mov    $0x0,%edx
  401386:    eb 13                    jmp    40139b <strings_not_equal+0x63>
  401388:    ba 00 00 00 00           mov    $0x0,%edx
  40138d:    eb 0c                    jmp    40139b <strings_not_equal+0x63>
  40138f:    ba 01 00 00 00           mov    $0x1,%edx
  401394:    eb 05                    jmp    40139b <strings_not_equal+0x63>
  401396:    ba 01 00 00 00           mov    $0x1,%edx
  40139b:    89 d0                    mov    %edx,%eax
  40139d:    5b                       pop    %rbx
  40139e:    5d                       pop    %rbp
  40139f:    41 5c                    pop    %r12
  4013a1:    c3                       ret
```
发现这个函数本质上就是C语言中的`strcmp(s1, s2) != 0`逻辑
弄清了phase_1的逻辑，拆除这个炸弹的密码其实就是调用`strings_not_equal`之前时%rsi中存储的字符串了，我们可以使用gdb来轻松获取。

进入gdb后，我们输入`layout asm`和`layout regs`以便于调试，我第一次做的时候以为只用简单地使用gdb的s和l指令进入phase_1对应的C语言函数中，就可以看出%rsi对应的字符串被赋值的什么，但实际上这样做是不可行的，需要使用si命令替代s，原因如下：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145511.png)

具体过程可以见下图，发现第一个炸弹的密码就是`Border relations with Canada have never been better.`，我们将其写入psol.txt的第一行，注意写完之后要换行，因为strings_not_equal函数中的string_length还单独统计了换行符，完成之后我们进行测试，发现成功跳过了第一个炸弹，这样我们就可以进入下一个phase了：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145531.png)

## phase2
在进入第二个阶段之前，我们可以将.gdbinit中的`b phase_1`给注释掉。
找到phase_2的汇编代码，我们先看前五行：
```asm
0000000000400efc <phase_2>:
  400efc:    55                       push   %rbp
  400efd:    53                       push   %rbx
  400efe:    48 83 ec 28              sub    $0x28,%rsp
  400f02:    48 89 e6                 mov    %rsp,%rsi 
  400f05:    e8 52 05 00 00           call   40145c <read_six_numbers>
```
其中：第四行，%rsi被赋值为`phase_2`的栈顶
第五行，phase_2调用了一个非常关键的函数`read_six_numbers`，我们来分析此代码：
```asm
000000000040145c <read_six_numbers>:
  40145c:    48 83 ec 18              sub    $0x18,%rsp
  401460:    48 89 f2                 mov    %rsi,%rdx
  401463:    48 8d 4e 04              lea    0x4(%rsi),%rcx
  401467:    48 8d 46 14              lea    0x14(%rsi),%rax
  40146b:    48 89 44 24 08           mov    %rax,0x8(%rsp)
  401470:    48 8d 46 10              lea    0x10(%rsi),%rax
  401474:    48 89 04 24              mov    %rax,(%rsp)
  401478:    4c 8d 4e 0c              lea    0xc(%rsi),%r9
  40147c:    4c 8d 46 08              lea    0x8(%rsi),%r8
  401480:    be c3 25 40 00           mov    $0x4025c3,%esi
  401485:    b8 00 00 00 00           mov    $0x0,%eax
  40148a:    e8 61 f7 ff ff           call   400bf0 <__isoc99_sscanf@plt>
  40148f:    83 f8 05                 cmp    $0x5,%eax
  401492:    7f 05                    jg     401499 <read_six_numbers+0x3d>
  401494:    e8 a1 ff ff ff           call   40143a <explode_bomb>
  401499:    48 83 c4 18              add    $0x18,%rsp
  40149d:    c3                       ret
```
先不看前面几行，直接看到：`40148a:    e8 61 f7 ff ff           call   400bf0 <__isoc99_sscanf@plt>`，这里又掉用了一个函数叫做：`sscanf`，因此我们猜测这行以前的所有代码**作用就在于为sscanf**这个函数的参数作准备。sscanf的作用请自行上网搜素，其函数声明为：
```c
int sscanf(const char *str, const char *format, ...)
```
参数传递的第一个寄存器为%rdi，在整个过程中并没有被修改，因此与phase1一样，其中保存的就是psol.txt的第二行字符串（本小节以后称之为“**res_str**”）
参数传递的第二个寄存器为%rsi，这个是非常重要的，可以让我们知道sscanf从res_str读取了些什么，因此我们使用gdb来探究%rsi中保存的format：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145543.png)

可以发现sscanf从res_str中一共读取了6个整数，因此sscanf中除了前两个参数之外，还需要传入六个参数，而我们知道用于传递参数的寄存器只有6个，因此还需要有两个参数需要放在`read_six_numbers`的栈帧上进行传递，该函数的sscanf具体参数来源如下图所示：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145550.png)

总之，这个函数的效果就是从psol.txt读取六个整数放入phase_2过程栈的六个单元中。
接着，我们看phase_2汇编代码的其它部分，此处已经给出了详细的注释：
```asm
  400f0a:    83 3c 24 01              cmpl   $0x1,(%rsp) 
  400f0e:    74 20                    je     400f30 <phase_2+0x34> # 如果a == 1跳转mark1
  400f10:    e8 25 05 00 00           call   40143a <explode_bomb>
  400f15:    eb 19                    jmp    400f30 <phase_2+0x34>
  400f17:    8b 43 fc                 mov    -0x4(%rbx),%eax # mark2：%eax被赋值为a
  400f1a:    01 c0                    add    %eax,%eax # a乘以2
  400f1c:    39 03                    cmp    %eax,(%rbx) 
  400f1e:    74 05                    je     400f25 <phase_2+0x29> # 如果a*2==b，跳转到mark3
  400f20:    e8 15 05 00 00           call   40143a <explode_bomb> # 不满足上述，爆炸！
  400f25:    48 83 c3 04              add    $0x4,%rbx # %rbx更新为c的地址
  400f29:    48 39 eb                 cmp    %rbp,%rbx 
  400f2c:    75 e9                    jne    400f17 <phase_2+0x1b> # 如果%rbx的地址超过了f的地址，结束（全部比较完毕）
  400f2e:    eb 0c                    jmp    400f3c <phase_2+0x40>
  400f30:    48 8d 5c 24 04           lea    0x4(%rsp),%rbx # mark1：%rbx被赋值为b的地址
  400f35:    48 8d 6c 24 18           lea    0x18(%rsp),%rbp # %rbp被赋值为f的下一个地址
  400f3a:    eb db                    jmp    400f17 <phase_2+0x1b> # 跳转到mark2
  400f3c:    48 83 c4 28              add    $0x28,%rsp
  400f40:    5b                       pop    %rbx
  400f41:    5d                       pop    %rbp
  400f42:    c3                       ret
```
发现：**只有当满足:** `f = 2e = 4d = 8c = 16b = 32a = 32`时，炸弹不会爆炸，因此我们就得出了第二个炸弹的密码：`1 2 4 8 16 32`
## phase3
同样地，我们首先把.gdbinit的`b phase_2`注释掉

拿到phase_3的汇编代码：
```asm
0000000000400f43 <phase_3>:
  400f43:    48 83 ec 18              sub    $0x18,%rsp
  400f47:    48 8d 4c 24 0c           lea    0xc(%rsp),%rcx
  400f4c:    48 8d 54 24 08           lea    0x8(%rsp),%rdx
  400f51:    be cf 25 40 00           mov    $0x4025cf,%esi # "%d %d" 
  400f56:    b8 00 00 00 00           mov    $0x0,%eax
  400f5b:    e8 90 fc ff ff           call   400bf0 <__isoc99_sscanf@plt> # 分别从%rdi中读进地址为%rsp+0x8和%rsp+0xc的两个整数
  400f60:    83 f8 01                 cmp    $0x1,%eax
  400f63:    7f 05                    jg     400f6a <phase_3+0x27> 
  400f65:    e8 d0 04 00 00           call   40143a <explode_bomb> # 如果读取的整数数量<=1，爆炸！
  400f6a:    83 7c 24 08 07           cmpl   $0x7,0x8(%rsp)
  400f6f:    77 3c                    ja     400fad <phase_3+0x6a> # 如果第一个整数>7，爆炸！
  400f71:    8b 44 24 08              mov    0x8(%rsp),%eax # 将第一个整数赋值给%eax
  400f75:    ff 24 c5 70 24 40 00     jmp    *0x402470(,%rax,8) # 跳转到地址0x402470 + %eax*8所指向的内存的值的地址
  400f7c:    b8 cf 00 00 00           mov    $0xcf,%eax
  400f81:    eb 3b                    jmp    400fbe <phase_3+0x7b>
  400f83:    b8 c3 02 00 00           mov    $0x2c3,%eax
  400f88:    eb 34                    jmp    400fbe <phase_3+0x7b>
  400f8a:    b8 00 01 00 00           mov    $0x100,%eax
  400f8f:    eb 2d                    jmp    400fbe <phase_3+0x7b>
  400f91:    b8 85 01 00 00           mov    $0x185,%eax
  400f96:    eb 26                    jmp    400fbe <phase_3+0x7b>
  400f98:    b8 ce 00 00 00           mov    $0xce,%eax
  400f9d:    eb 1f                    jmp    400fbe <phase_3+0x7b>
  400f9f:    b8 aa 02 00 00           mov    $0x2aa,%eax
  400fa4:    eb 18                    jmp    400fbe <phase_3+0x7b>
  400fa6:    b8 47 01 00 00           mov    $0x147,%eax
  400fab:    eb 11                    jmp    400fbe <phase_3+0x7b>
  400fad:    e8 88 04 00 00           call   40143a <explode_bomb>
  400fb2:    b8 00 00 00 00           mov    $0x0,%eax
  400fb7:    eb 05                    jmp    400fbe <phase_3+0x7b>
  400fb9:    b8 37 01 00 00           mov    $0x137,%eax
  400fbe:    3b 44 24 0c              cmp    0xc(%rsp),%eax # 将第二个整数和0x137进行比较
  400fc2:    74 05                    je     400fc9 <phase_3+0x86> # 如果相等，结束
  400fc4:    e8 71 04 00 00           call   40143a <explode_bomb>
  400fc9:    48 83 c4 18              add    $0x18,%rsp
  400fcd:    c3                       ret
```
会发现实际上phase3就是一个简化版的phase2，sscanf读取了两个整数，我们称之为a和b，满足：
$`\begin{cases} 0 \leq a \leq 7 \\ \%rax = b \end{cases}`$
其中%rax的取值与a有关，我们在这里不妨取a为0，则在400f75处，程序会跳转到地址`0x402470`所指向的内存中的值的地址，所以我们可以先求出这个地址的值，如下所示：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145607.png)

发现此时正好跳转到下一行，此时%rax被赋值为十进制的207，因此答案为`0 207`

当然a取其它值也是可以的，这道题的答案不唯一。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145613.png)

## phase4
本阶段的炸弹涉及了两个函数，下面依次是两个函数的汇编代码以及注释
```asm
0000000000400fce <func4>: # 初始值%rdi = a, %rsi = 0, %rdx = 0xe
  400fce:    48 83 ec 08              sub    $0x8,%rsp
  400fd2:    89 d0                    mov    %edx,%eax # %rax = %rdx
  400fd4:    29 f0                    sub    %esi,%eax # %rax = %rdx - %rsi
  400fd6:    89 c1                    mov    %eax,%ecx # %rcx = %rax
  400fd8:    c1 e9 1f                 shr    $0x1f,%ecx # %rcx逻辑右移31位，%rcx = 0 or 1
  400fdb:    01 c8                    add    %ecx,%eax # %rax = %rax + %rcx
  400fdd:    d1 f8                    sar    %eax # %rax算术右移1位
  400fdf:    8d 0c 30                 lea    (%rax,%rsi,1),%ecx # %rcx = %rax + %rsi
  400fe2:    39 f9                    cmp    %edi,%ecx 
  400fe4:    7e 0c                    jle    400ff2 <func4+0x24> # 如果%rcx <= a，跳转到400ff2
  400fe6:    8d 51 ff                 lea    -0x1(%rcx),%edx # %rdx = %rcx - 1
  400fe9:    e8 e0 ff ff ff           call   400fce <func4> # 递归调用
  400fee:    01 c0                    add    %eax,%eax # %rax *= 2
  400ff0:    eb 15                    jmp    401007 <func4+0x39> # 跳转到结束
  400ff2:    b8 00 00 00 00           mov    $0x0,%eax # %rax赋值为0
  400ff7:    39 f9                    cmp    %edi,%ecx
  400ff9:    7d 0c                    jge    401007 <func4+0x39> # 如果%rcx >= a，结束
  400ffb:    8d 71 01                 lea    0x1(%rcx),%esi # %rsi = %rcx + 1
  400ffe:    e8 cb ff ff ff           call   400fce <func4> # 递归调用
  401003:    8d 44 00 01              lea    0x1(%rax,%rax,1),%eax # %rax = %rax*2 + 1
  401007:    48 83 c4 08              add    $0x8,%rsp
  40100b:    c3                       ret    

000000000040100c <phase_4>:
  40100c:    48 83 ec 18              sub    $0x18,%rsp
  401010:    48 8d 4c 24 0c           lea    0xc(%rsp),%rcx
  401015:    48 8d 54 24 08           lea    0x8(%rsp),%rdx # 0x8(%rsp)存放第一个整数a，0xc(%rsp)存放第二个整数b
  40101a:    be cf 25 40 00           mov    $0x4025cf,%esi
  40101f:    b8 00 00 00 00           mov    $0x0,%eax
  401024:    e8 c7 fb ff ff           call   400bf0 <__isoc99_sscanf@plt>
  401029:    83 f8 02                 cmp    $0x2,%eax
  40102c:    75 07                    jne    401035 <phase_4+0x29>
  40102e:    83 7c 24 08 0e           cmpl   $0xe,0x8(%rsp) # 必须满足a <= 0xe
  401033:    76 05                    jbe    40103a <phase_4+0x2e>
  401035:    e8 00 04 00 00           call   40143a <explode_bomb>
  40103a:    ba 0e 00 00 00           mov    $0xe,%edx # 后续传参的第三个参数赋值为0xe
  40103f:    be 00 00 00 00           mov    $0x0,%esi # 后续传参的第二个参数赋值为0
  401044:    8b 7c 24 08              mov    0x8(%rsp),%edi # 后续传参的第一个参数赋值为a
  401048:    e8 81 ff ff ff           call   400fce <func4>
  40104d:    85 c0                    test   %eax,%eax
  40104f:    75 07                    jne    401058 <phase_4+0x4c> # 如果返回值不为0，爆炸！
  401051:    83 7c 24 0c 00           cmpl   $0x0,0xc(%rsp)
  401056:    74 05                    je     40105d <phase_4+0x51> # 如果b == 0，结束
  401058:    e8 dd 03 00 00           call   40143a <explode_bomb>
  40105d:    48 83 c4 18              add    $0x18,%rsp
  401061:    c3                       ret
```
与phase3相同，依旧是在psol.txt中读取两个整数a和b，在phase_4函数中可以看出需要满足两个条件，第一，a是必须小于等于0xe的；第二，b必然为0，此外a要被传入到另一个函数func中，只有当func返回值为0炸弹才不爆炸。

因此问题转换为了a取什么值时，func4的返回值为0，直接分析func4的代码容易知道a应该等于7，因此最终答案为`7 0`
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145623.png)

## phase5
汇编代码+注释如下：
```asm
0000000000401062 <phase_5>:
  401062:    53                       push   %rbx
  401063:    48 83 ec 20              sub    $0x20,%rsp
  401067:    48 89 fb                 mov    %rdi,%rbx # 将输入字符串的地址存放到%rbx
  40106a:    64 48 8b 04 25 28 00     mov    %fs:0x28,%rax # 栈保护
  401071:    00 00 
  401073:    48 89 44 24 18           mov    %rax,0x18(%rsp) # 在栈中保存栈保护值
  401078:    31 c0                    xor    %eax,%eax # %eax = 0
  40107a:    e8 9c 02 00 00           call   40131b <string_length> # 计算输入字符串的长度
  40107f:    83 f8 06                 cmp    $0x6,%eax
  401082:    74 4e                    je     4010d2 <phase_5+0x70>
  401084:    e8 b1 03 00 00           call   40143a <explode_bomb> # 如果长度不为6，爆炸！
  401089:    eb 47                    jmp    4010d2 <phase_5+0x70>
  40108b:    0f b6 0c 03              movzbl (%rbx,%rax,1),%ecx # 取输入字符串的每个字符ch
  40108f:    88 0c 24                 mov    %cl,(%rsp) # 将ch存到栈顶
  401092:    48 8b 14 24              mov    (%rsp),%rdx # 将栈顶初存的内容存到%rdx
  401096:    83 e2 0f                 and    $0xf,%edx # ch & 0xf赋值给%rdx，%rdx范围0-15
  401099:    0f b6 92 b0 24 40 00     movzbl 0x4024b0(%rdx),%edx # 用%rdx作为下标，从0x4024b0处取一个字符，存入%rdx
  4010a0:    88 54 04 10              mov    %dl,0x10(%rsp,%rax,1) # 将取到的字符存到栈中，形成新的字符串
  4010a4:    48 83 c0 01              add    $0x1,%rax # 处理下一个字符
  4010a8:    48 83 f8 06              cmp    $0x6,%rax
  4010ac:    75 dd                    jne    40108b <phase_5+0x29> # 如果还没处理完6个字符，继续，总共就是处理6个字符
  4010ae:    c6 44 24 16 00           movb   $0x0,0x16(%rsp) # 新字符串结尾添加'\0'
  4010b3:    be 5e 24 40 00           mov    $0x40245e,%esi # 给%esi赋值为$0x40245e
  4010b8:    48 8d 7c 24 10           lea    0x10(%rsp),%rdi # 给%rdi赋值为新字符串的地址
  4010bd:    e8 76 02 00 00           call   401338 <strings_not_equal> # 保证两个地址保存的字符串一样
  4010c2:    85 c0                    test   %eax,%eax
  4010c4:    74 13                    je     4010d9 <phase_5+0x77> 
  4010c6:    e8 6f 03 00 00           call   40143a <explode_bomb>
  4010cb:    0f 1f 44 00 00           nopl   0x0(%rax,%rax,1)
  4010d0:    eb 07                    jmp    4010d9 <phase_5+0x77>
  4010d2:    b8 00 00 00 00           mov    $0x0,%eax # %eax = 0
  4010d7:    eb b2                    jmp    40108b <phase_5+0x29>
  4010d9:    48 8b 44 24 18           mov    0x18(%rsp),%rax # 取出栈保护值
  4010de:    64 48 33 04 25 28 00     xor    %fs:0x28,%rax # 栈保护
  4010e5:    00 00 
  4010e7:    74 05                    je     4010ee <phase_5+0x8c> # 栈保护成功
  4010e9:    e8 42 fa ff ff           call   400b30 <__stack_chk_fail@plt> # 栈保护失败，爆炸
  4010ee:    48 83 c4 20              add    $0x20,%rsp
  4010f2:    5b                       pop    %rbx
  4010f3:    c3                       ret
```
首先读取六个整数（参考phase2）a、b、c、d、e、f
依次将这六个整数与0xf进行与的掩码运算并作为一个字符数组s1（首地址为0x4024b0）的下标，从而得到六个不同的字符，并且满足这六个字符组合成的字符串与地址为0x40245e处的字符串s2相同
因此，接下来需要知道上述两个地址对应位置的字符串s1、s2分别为什么：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145633.png)

对于s2的第一个字符f，保证`s1[a&0xf] == 'f'`，其中下标取值一定为0-15，因此只需看s1的前16个字符即可，发现下标分别为9、15、14、5、6、7
对应的二进制分别为：
1001、1111、1110、0101、0110、0111
对照ascii表，发现可以填字符：IONEFG
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145639.png)

通过：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145646.png)

## phase6
这部分的汇编代码比较长，因此，我们进行分段进行讲解：
**第一部分**
```asm
00000000004010f4 <phase_6>:
  4010f4:    41 56                    push   %r14
  4010f6:    41 55                    push   %r13
  4010f8:    41 54                    push   %r12
  4010fa:    55                       push   %rbp
  4010fb:    53                       push   %rbx
  4010fc:    48 83 ec 50              sub    $0x50,%rsp
  401100:    49 89 e5                 mov    %rsp,%r13 # %r13存放六个整数的地址
  401103:    48 89 e6                 mov    %rsp,%rsi 
  401106:    e8 51 03 00 00           call   40145c <read_six_numbers>
  40110b:    49 89 e6                 mov    %rsp,%r14 # %r14存放六个整数的首地址
  40110e:    41 bc 00 00 00 00        mov    $0x0,%r12d # %r12 = 0
  401114:    4c 89 ed                 mov    %r13,%rbp # mark3：%rbp存放六个整数的首地址
  401117:    41 8b 45 00              mov    0x0(%r13),%eax # 取第一个整数a
  40111b:    83 e8 01                 sub    $0x1,%eax # %rax = a - 1
  40111e:    83 f8 05                 cmp    $0x5,%eax 
  401121:    76 05                    jbe    401128 <phase_6+0x34> # 保证%rax <= 5，因为是无符号，所以保证每个整数在1-6范围且各不相同
  401123:    e8 12 03 00 00           call   40143a <explode_bomb>
  401128:    41 83 c4 01              add    $0x1,%r12d # %r12 ++ 
  40112c:    41 83 fc 06              cmp    $0x6,%r12d 
  401130:    74 21                    je     401153 <phase_6+0x5f> # 如果处理完6个整数，跳转mark1
  401132:    44 89 e3                 mov    %r12d,%ebx # %rbx 存即将要处理的下个整数的索引
  401135:    48 63 c3                 movslq %ebx,%rax # mark2：%rax = %rbx
  401138:    8b 04 84                 mov    (%rsp,%rax,4),%eax # 取出即将要处理的下一个整数给%rax
  40113b:    39 45 00                 cmp    %eax,0x0(%rbp) # 将下一个整数和第一个整数比较
  40113e:    75 05                    jne    401145 <phase_6+0x51> # 如果相等，爆炸
  401140:    e8 f5 02 00 00           call   40143a <explode_bomb>
  401145:    83 c3 01                 add    $0x1,%ebx # 处理下一个整数
  401148:    83 fb 05                 cmp    $0x5,%ebx
  40114b:    7e e8                    jle    401135 <phase_6+0x41> # %rbx <= 5，跳转到mark2
  40114d:    49 83 c5 04              add    $0x4,%r13 # 处理下一个整数
  401151:    eb c1                    jmp    401114 <phase_6+0x20> # 跳转到mark3
```
前面的代码依旧是从psol.txt的第六行读取了六个整数a、b、c、d、e、f
其余代码是一个双重循环，有两个作用：
* 保证输入的六个整数均大于等于1且小于等于6
* 保证输入的六个整数互不相同
**第二部分**
```asm
  401153:    48 8d 74 24 18           lea    0x18(%rsp),%rsi # mark1：%rsi存放栈中六个整数的末尾地址
  401158:    4c 89 f0                 mov    %r14,%rax # %rax存放六个整数的首地址
  40115b:    b9 07 00 00 00           mov    $0x7,%ecx # %rcx = 7
  401160:    89 ca                    mov    %ecx,%edx # %rdx = 7
  401162:    2b 10                    sub    (%rax),%edx # %rdx = 7 - 当前整数
  401164:    89 10                    mov    %edx,(%rax) # 将%rdx存回当前整数的位置
  401166:    48 83 c0 04              add    $0x4,%rax # 处理下一个整数
  40116a:    48 39 f0                 cmp    %rsi,%rax # 比较是否处理完六个整数
  40116d:    75 f1                    jne    401160 <phase_6+0x6c> # 没有处理完，继续
```
这部分代码的作用是：把每个整数x，变化为`7 - x`
**第三部分**
```asm
  40116f:    be 00 00 00 00           mov    $0x0,%esi # %rsi = 0
  401174:    eb 21                    jmp    401197 <phase_6+0xa3> # 跳转到mark4
  401176:    48 8b 52 08              mov    0x8(%rdx),%rdx # %rdx = 当前节点的下一个节点地址
  40117a:    83 c0 01                 add    $0x1,%eax # %rax ++
  40117d:    39 c8                    cmp    %ecx,%eax 
  40117f:    75 f5                    jne    401176 <phase_6+0x82> # 直到%rax == %rcx（当前整数），结束
  401181:    eb 05                    jmp    401188 <phase_6+0x94>
  401183:    ba d0 32 60 00           mov    $0x6032d0,%edx # mark5：%rdx = 0x6032d0
  401188:    48 89 54 74 20           mov    %rdx,0x20(%rsp,%rsi,2) # 将%rdx存到栈中，形成链表节点
  40118d:    48 83 c6 04              add    $0x4,%rsi # 处理下一个整数
  401191:    48 83 fe 18              cmp    $0x18,%rsi # 比较是否处理完六个整数
  401195:    74 14                    je     4011ab <phase_6+0xb7> # 处理完，跳转到mark6
  401197:    8b 0c 34                 mov    (%rsp,%rsi,1),%ecx # mark4：%rcx = 当前处理的整数
  40119a:    83 f9 01                 cmp    $0x1,%ecx 
  40119d:    7e e4                    jle    401183 <phase_6+0x8f> # 如果当前整数<=1，跳转到mark5
  40119f:    b8 01 00 00 00           mov    $0x1,%eax # %rax = 1
  4011a4:    ba d0 32 60 00           mov    $0x6032d0,%edx # %rdx = 0x6032d0
  4011a9:    eb cb                    jmp    401176 <phase_6+0x82>
```
这部分代码的关键在于理解地址`0x6032d0`处是一个什么样的数据，由于我们不知道这里的数据有多大，我们可以用gdb在这里输出一个尽可能大的单元，比如使用`x/32xw`得到：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145656.png)

会发现每一行的第三列恰好都能对应到某一行node前的地址，因此我们推断出这是一个结点数量为6的链表，其中第七行为空结点NULL，第一列为结点的值val，第二列为结点的序号id，第三列为结点的next域。

知道这个之后，我们发现第三部分代码的作用是根据现在六个整数的值分别取出链表对应位置的节点地址放入过程的栈帧中（从%rsp+0x20这个位置往后放）
举一个例子，假设现在六个整数分别为3 2 1 6 5 4，对于第一个整数3，就是要取出第3个结点的地址0x6032f0放到%rsp+0x20这个位置，依次类推，最终栈的内存会变为：
（下图引用于文章：https://blog.csdn.net/m0_55763458/article/details/136542093）
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145710.png)

**第四部分**
```asm
  4011ab:    48 8b 5c 24 20           mov    0x20(%rsp),%rbx # mark6：%rbx存放链表头节点地址
  4011b0:    48 8d 44 24 28           lea    0x28(%rsp),%rax # %rax存放下一个结点地址
  4011b5:    48 8d 74 24 50           lea    0x50(%rsp),%rsi # %rsi存放栈中的链表节点最后一个地址
  4011ba:    48 89 d9                 mov    %rbx,%rcx # %rcx存放链表头节点地址%rbx
  4011bd:    48 8b 10                 mov    (%rax),%rdx # %rdx存放下一个链表节点地址
  4011c0:    48 89 51 08              mov    %rdx,0x8(%rcx) # 将当前节点的下一个节点地址存到当前节点的next域
  4011c4:    48 83 c0 08              add    $0x8,%rax # 处理下一个节点
  4011c8:    48 39 f0                 cmp    %rsi,%rax # 比较是否处理完所有节点
  4011cb:    74 05                    je     4011d2 <phase_6+0xde> # 如果处理完就跳转到mark7
  4011cd:    48 89 d1                 mov    %rdx,%rcx
  4011d0:    eb eb                    jmp    4011bd <phase_6+0xc9>
  4011d2:    48 c7 42 08 00 00 00     movq   $0x0,0x8(%rdx) # mark7：将最后一个节点的next域置为NULL
  4011d9:    00 
  4011da:    bd 05 00 00 00           mov    $0x5,%ebp # %rbp = 5
  4011df:    48 8b 43 08              mov    0x8(%rbx),%rax # 取栈中第一个链表结点的next域赋值给%rax
  4011e3:    8b 00                    mov    (%rax),%eax # %rax赋值给第一个链表节点的值域赋值给%eax
  4011e5:    39 03                    cmp    %eax,(%rbx) 
  4011e7:    7d 05                    jge    4011ee <phase_6+0xfa>  # 保证当前节点的值域大于等于下一个节点的值域，否则爆炸
  4011e9:    e8 4c 02 00 00           call   40143a <explode_bomb>
  4011ee:    48 8b 5b 08              mov    0x8(%rbx),%rbx # 处理下一个节点
  4011f2:    83 ed 01                 sub    $0x1,%ebp # %rbp --
  4011f5:    75 e8                    jne    4011df <phase_6+0xeb>
  4011f7:    48 83 c4 50              add    $0x50,%rsp
  4011fb:    5b                       pop    %rbx
  4011fc:    5d                       pop    %rbp
  4011fd:    41 5c                    pop    %r12
  4011ff:    41 5d                    pop    %r13
  401201:    41 5e                    pop    %r14
  401203:    c3                       ret
```
这部分就是保证从rsp+0x20到rsp+0x48对应的六个结点的值val是单调递减的，原始链表排序：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20251211145719.png)

按照数组顺序调整后的链表顺序：
924(3)->691(4)->477(5)->443(6)->332(1)->168(2)
故n各个元素为：3 4 5 6 1 2
但是别忘了，在上述代码中，对数组的每个元素x进行了运算：x = 7-x，所以原始数组的各个元素为：4 3 2 1 6 5，即为传入的参数
answer
4 3 2 1 6 5
最终我们就完成了所有实验。

