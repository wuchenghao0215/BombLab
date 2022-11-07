# BombLab

## phase_1 （考察判断）

输入

```text
(gdb) disas phase_1
```

可以看到phase_1函数的汇编代码

```text
   0x000000000000243d <+4>:     lea    0x1d04(%rip),%rsi        # 0x4148
   0x0000000000002444 <+11>:    callq  0x29b1 <strings_not_equal>
   0x0000000000002449 <+16>:    test   %eax,%eax
   0x000000000000244b <+18>:    jne    0x2452 <phase_1+25>
   0x000000000000244d <+20>:    add    $0x8,%rsp
   0x0000000000002451 <+24>:    retq   
   0x0000000000002452 <+25>:    callq  0x2c8d <explode_bomb>
```

```strings_not_equal```函数，看名字可以推测是判断字符串是否相等，若不相等返回1，导致爆炸。所以要找到希望的字符串。输入

```text
(gdb) x /s 0x4148
```

会得到

```text
0x4148:     "An endless Light awaits you in a perfect world of silent song..."
```

于是phase_1只要手动输入```An endless Light awaits you in a perfect world of silent song...```即可通过

## phase_2 （考察循环）

查看phase_2的汇编代码，注意到```read_six_numbers```函数，查看它的汇编

```text
   0x0000000000002d6a <+29>:     lea    0x1884(%rip),%rsi        # 0x45f5
   0x0000000000002d71 <+36>:     mov    $0x0,%eax
   0x0000000000002d76 <+41>:     callq  0x2150 <__isoc99_sscanf@plt>
   0x0000000000002d7b <+46>:     add    $0x10,%rsp
   0x0000000000002d7f <+50>:     cmp    $0x5,%eax
   0x0000000000002d82 <+53>:     jle    0x2d89 <read_six_numbers+60>
```

而```0x45f5```的内容为

```text
0x45f5:     "%d %d %d %d %d %d"
```

它用sscanf依次读入6个int，%eax寄存器的值（经过gdb查看寄存器的值发现到这一步总是6，猜测是读入的int的个数）与5比较，若小于等于5则触发炸弹

再回到phase_2主函数，经历以下步骤

1. 先将(%rsp)的值（即输入的第一个数）与0xc比较，若不相等则触发炸弹，因此第一个数应该输入12
2. 把%rsp存到%rbx，把%rsp加上0x14存到%rbp（一个int占4个字节，0x14是第6个数）
3. 把0x3c87(%rip)（即一个数字0x7）存到%eax
4. 0x7乘上第一个数（0xc）存到%eax（0x54，即84），判断与0x4(%rbx)（4刚好是一个int的偏移量，因此是第二个数）是否相等，如果不相等触发爆炸。于是第二个数应该是84
5. 在第4步相等的情况下，跳转到```<phase_2+53>```，重复执行，到这里看出来是一个循环
6. 从第二个数（即```a[1]```）开始，每个数是前一个数的7倍
7. 所以phase_2应该依次输入```12 84 588 4116 28812 201684```

```phase_2```写成C程序大概是

```C
void phase_2(int *a) {
   if (a[0] != 12) {
      explode_bomb();
   }
   for (int i = 1; i < 6; ++i) {
      if (a[i] != 7 * a[i - 1]) {
         explode_bomb();
      }
   }
}
```

## phase_3 （考察switch）

用同样的方法，先观察汇编。注意到和phase_2一样用到了```sscanf```函数，由下一行的cmp函数与0x2比较可知，这个phase应该输入3个数字。要找到准确的输入格式，输入：

```text
(gdb) x /s $rsi
```

得到

```text
0x5555555581b6:   "%d %c %d"
```

证实了之前的猜测，但要注意第二个输入应当是char类型。char存在0xf(%rsp)，第一个int存在0x10(%rsp)，第二个int存在0x14(%rsp)

输入的char与0x20做了一次异或，暂时不知道有什么用处

在下面有一组```movslq   (%rdx,%rax,4),%rax```然后```jmpq    *%rax```的指令非常引人注意，看起来是switch语句。

1. 如果第一个int输入0，跳转到第一个分支
2. 第二个int应该等于0x22f，即559
3. 然后知道输入的字符 xor 0x20 应该等于0x69，即应该输入```I```

综上，依次输入```0 I 559```即可过掉这个phase

## phase_4 （考察递归）

1. 输入两个int，第一个应当小于等于```0xe```，即小于等于14。
2. 把```(第一个输入, 0, 14)```作为参数，调用func4函数
3. func返回值应该是7，第二个输入减去11应该是7
4. 所以这个phase应该输入```14 18```

func4函数的C语言表示如下

```C
func4(int x, int y, int z) {
   int a = z - y;
   if (a < 0) {
      a = (a + 1) / 2;
   } else {
      a = a / 2;
   }
   a += y;
   if (a > x) {
      return 2 * func4(x, y, a - 1);
   } else if (x < a) {
      return 2 * func4(x, a + 1, z) + 1;
   } else {
      return 0;
   }
}
```

## phase_5 （还是考察循环）

1. 有一个string_length的函数，显然是返回输入字符串的长度
2. 应该输入长度为6的字符串
3. 每个字符与0xf做and运算，只取最低4位
4. 在一个长度为16的数组中找到对应整数，相加为50
5. 数组表示为```[2, 10, 6, 1, 12, 16, 9, 3, 4, 7, 14, 5, 11, 8, 15, 13]```
6. 由于字符```'0'```与0xf求and刚好是0，所以只要输入```111136```即可