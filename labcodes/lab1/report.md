

### [练习1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中 每一条相关命令和命令参数的含义,以及说明命令导致的结果)

```
$(UCOREIMG): $(kernel) $(bootblock)
|   $(V)dd if=/dev/zero of=$@ count=10000
|   $(V)dd if=$(bootblock) of=$@ conv=notrunc
|   $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

这三句话用来生成ucore.img，分别生成bootblock和kernel

生成bootblock的相关代码：
| $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
|   @echo + ld $@
|   $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ \
|       -o $(call toobj,bootblock)
|   @$(OBJDUMP) -S $(call objfile,bootblock) > \
|       $(call asmfile,bootblock)
|   @$(OBJCOPY) -S -O binary $(call objfile,bootblock) \
|       $(call outfile,bootblock)
|   @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

生成kernel的相关代码：
| $(kernel): tools/kernel.ld
| $(kernel): $(KOBJS)
|   @echo + ld $@
|   $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
|   @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
|   @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
|       /^$$/d' > $(call symfile,kernel)
```

### [练习3] 分析bootloader进入保护模式的过程
主要程序在/boot/bootasm.S中

首先将各种段寄存器清零：
```
 .code16
        cli
        cld
        xorw %ax, %ax
        movw %ax, %ds
        movw %ax, %es
        movw %ax, %ss
```
然后开启A20，启用全部32条地址线：
```
  seta20.1:               # 等待8042键盘控制器不忙
      inb $0x64, %al      # 
      testb $0x2, %al     #
      jnz seta20.1        #

      movb $0xd1, %al     # 发送写8042输出端口的指令
      outb %al, $0x64     #

  seta20.1:               # 等待8042键盘控制器不忙
      inb $0x64, %al      # 
      testb $0x2, %al     #
      jnz seta20.1        #

      movb $0xdf, %al     # 打开A20
      outb %al, $0x60     # 
```
接着初始化GDT表，并通过CR0寄存器开启保护模式：
```
  lgdt gdtdesc
  movl %cr0, %eax
  orl $CR0_PE_ON, %eax
  movl %eax, %cr0     
```
接着通过长跳转，跳到kernel:
```
  ljmp $PROT_MODE_CSEG, $protcseg
  .code32
  protcseg:
```
接着设置段寄存器、建立堆栈，进入bootmain:
```
  movw $PROT_MODE_DSEG, %ax
  movw %ax, %ds
  movw %ax, %es
  movw %ax, %fs
  movw %ax, %gs
  movw %ax, %ss
  movl $0x0, %ebp
  movl $start, %esp
  call bootmain
```
### [练习4]分析bootloader加载ELF格式的OS的过程。
readsect和readseg函数用来从磁盘中读取数据。
读取的过程为：
- 读取ELF的头部
- 判断ELF头是否合法
- 找到ELF的ph描述表
- 将ELF文件载入内存
- 进入内核

###[练习6] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
- IDT的一个表项占8字节
- gd_ss代表中断处理代码入口的段选择址
- gd_off_31_16代表中断处理代码入口的段偏移
