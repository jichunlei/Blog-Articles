# **一、工具**

![](http://img.xianzilei.cn/idea%E6%9F%A5%E7%9C%8B%E6%B1%87%E7%BC%96%E6%8C%87%E4%BB%A4-1.png)

**下载地址**

* 链接：https://pan.baidu.com/s/1ooF8IL6K_DgHM9-bCV3QxA 
* 提取码：1dda 

# **二、配置**

* **1）将上述两个文件拷贝到你按照的jdk中的jre/bin目录下**

  ![](http://img.xianzilei.cn/idea%E6%9F%A5%E7%9C%8B%E6%B1%87%E7%BC%96%E6%8C%87%E4%BB%A4-2.png)

* **2）配置idea的启动参数**

  ```shell
  # 其中ClassName.method配置你自己的类名和方法名，表示你要打印哪个类的哪个方法的汇编指令
  -server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*ClassName.method
  ```

  例如，我想打印main方法的字节码

  ```java
  public class MyTest {
  
      public static void main(String[] args){
          int i=0;
          i++;
          System.out.println(i);
      }
  }
  ```

  **在启动配置中配置如下vm参数，注意jre要选择你拷贝文件位置的jre**

  ```shell
  -server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*MyTest.main
  ```

  **如下图所示**

  ![](http://img.xianzilei.cn/idea%E6%9F%A5%E7%9C%8B%E6%B1%87%E7%BC%96%E6%8C%87%E4%BB%A4-3.png)

# **三、使用**

使用的时候**直接启动你要执行的方法**，以上述的main方法为例

启动main方法，查看控制台打印日志

```tex
CompilerOracle: compileonly *MyTest.main
Loaded disassembler from hsdis-amd64.dll
Decoding compiled method 0x0000000002e60890:
Code:
Argument 0 is unknown.RIP: 0x2e60a00 Code size: 0x000002a8
[Disassembling for mach='amd64']
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x00000000176d2b18} 'main' '([Ljava/lang/String;)V' in 'com/jicl/MyTest'
  # parm0:    rdx:rdx   = '[Ljava/lang/String;'
  #           [sp+0x40]  (sp of caller)
  0x0000000002e60a00: mov     dword ptr [rsp+0ffffffffffffa000h],eax
  0x0000000002e60a07: push    rbp
  0x0000000002e60a08: sub     rsp,30h
  0x0000000002e60a0c: mov     r8,1772c030h      ;   {metadata(method data for {method} {0x00000000176d2b18} 'main' '([Ljava/lang/String;)V' in 'com/jicl/MyTest')}
  0x0000000002e60a16: mov     esi,dword ptr [r8+0dch]
  0x0000000002e60a1d: add     esi,8h
  0x0000000002e60a20: mov     dword ptr [r8+0dch],esi
  0x0000000002e60a27: mov     r8,176d2b10h      ;   {metadata({method} {0x00000000176d2b18} 'main' '([Ljava/lang/String;)V' in 'com/jicl/MyTest')}
  0x0000000002e60a31: and     esi,0h
  0x0000000002e60a34: cmp     esi,0h
  0x0000000002e60a37: je      2e60b10h          ;*iconst_0
                                                ; - com.jicl.MyTest::main@0 (line 15)

  0x0000000002e60a3d: nop
  0x0000000002e60a40: jmp     2e60b84h          ;   {no_reloc}
  0x0000000002e60a45: add     byte ptr [rax],al
  0x0000000002e60a47: add     byte ptr [rax],al
  0x0000000002e60a49: add     byte ptr [rsi+0fh],ah
  0x0000000002e60a4c: Fatal error: Disassembling failed with error code: 15Decoding compiled method 0x0000000002e63a10:
Code:
Argument 0 is unknown.RIP: 0x2e63b40 Code size: 0x00000038
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x00000000176d2b18} 'main' '([Ljava/lang/String;)V' in 'com/jicl/MyTest'
  # parm0:    rdx:rdx   = '[Ljava/lang/String;'
  #           [sp+0x20]  (sp of caller)
  0x0000000002e63b40: mov     dword ptr [rsp+0ffffffffffffa000h],eax
  0x0000000002e63b47: push    rbp
  0x0000000002e63b48: sub     rsp,10h           ;*synchronization entry
                                                ; - com.jicl.MyTest::main@-1 (line 15)

  0x0000000002e63b4c: mov     edx,1bh
  0x0000000002e63b51: nop
  0x0000000002e63b53: call    2d957a0h          ; OopMap{off=24}
                                                ;*getstatic out
                                                ; - com.jicl.MyTest::main@5 (line 17)
                                                ;   {runtime_call}
  0x0000000002e63b58: int3                      ;*getstatic out
                                                ; - com.jicl.MyTest::main@5 (line 17)

  0x0000000002e63b59: hlt
  0x0000000002e63b5a: hlt
  0x0000000002e63b5b: hlt
  0x0000000002e63b5c: hlt
  0x0000000002e63b5d: hlt
  0x0000000002e63b5e: hlt
  0x0000000002e63b5f: hlt
[Exception Handler]
[Stub Code]
  0x0000000002e63b60: jmp     2dbf4a0h          ;   {no_reloc}
[Deopt Handler Code]
  0x0000000002e63b65: call    2e63b6ah
  0x0000000002e63b6a: sub     qword ptr [rsp],5h
  0x0000000002e63b6f: jmp     2d97600h          ;   {runtime_call}
  0x0000000002e63b74: hlt
  0x0000000002e63b75: hlt
  0x0000000002e63b76: hlt
  0x0000000002e63b77: hlt
1
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output

Process finished with exit code 0
```

**配置成功！**