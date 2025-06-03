### windows

接口：SetUnhandledExceptionFilter

结合 `MiniDumpWriteDump` 函数在异常发生时生成内存转储文件（.dmp）

```c++
#include <DbgHelp.h>
#pragma comment(lib, "Dbghelp.lib")

HANDLE hFile = CreateFile("crash.dmp", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
MINIDUMP_EXCEPTION_INFORMATION exceptionInfo;
exceptionInfo.ThreadId = GetCurrentThreadId();
exceptionInfo.ExceptionPointers = ExceptionInfo;
exceptionInfo.ClientPointers = FALSE;
MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(), hFile, MiniDumpNormal, &exceptionInfo, NULL, NULL);
CloseHandle(hFile);
```



### mac

使用crashpad

1. 崩溃时上传dump数据
2. 维护一个有限长度的列表，将崩溃前的操作都记录下来，在崩溃时，通过        client_status.SetAnnotation(key, value);增加注释消息到dump文件中


Crashpad的异常捕获机制是其核心功能之一，它通过多层次、跨平台的设计实现了对各类异常的可靠捕获。以下是其工作原理的详细解析：


### **一、跨平台异常捕获架构**
Crashpad采用**客户端-服务端**架构，将异常捕获与处理分离，避免因崩溃处理逻辑导致二次崩溃：

1. **客户端库**（嵌入应用）：  
   - 注册低级别异常处理钩子。  
   - 与独立的服务端进程通信。  
   - 提供API供应用添加自定义信息。

2. **服务端进程**（Crashpad Handler）：  
   - 独立于主应用运行的进程。  
   - 接收客户端通知，安全收集崩溃数据。  
   - 生成崩溃转储文件并处理上传。


### **二、各平台异常捕获实现**
#### **1. Windows平台**
- **异常类型**：  
  捕获SEH（结构化异常处理）异常，如：  
  - 访问违规（EXCEPTION_ACCESS_VIOLATION）  
  - 除零错误（EXCEPTION_INT_DIVIDE_BY_ZERO）  
  - 栈溢出（EXCEPTION_STACK_OVERFLOW）  

- **实现机制**：  
  1. **向量化异常处理（VEH）**：  
     通过`AddVectoredExceptionHandler`注册全局异常处理回调，优先级高于SEH链。  
  2. **顶层异常过滤器**：  
     通过`SetUnhandledExceptionFilter`注册后备处理程序，处理未被VEH或SEH捕获的异常。  
  3. **异步异常通知**：  
     当异常发生时，客户端通过命名管道通知服务端进程。  
  4. **MiniDump生成**：  
     服务端使用`MiniDumpWriteDump`生成包含完整内存状态的.dmp文件。

#### **2. macOS平台**
- **异常类型**：  
  捕获Mach异常，如：  
  - EXC_BAD_ACCESS（内存访问错误）  
  - EXC_ARITHMETIC（算术错误）  
  - EXC_BREAKPOINT（断点/调试异常）  

- **实现机制**：  
  1. **Mach异常端口**：  
     通过`task_set_exception_ports`注册异常处理端口，拦截进程的所有异常。  
  2. **异步通知**：  
     异常发生时，客户端通过XPC（XPC Services）通知服务端。  
  3. **Core Dump生成**：  
     服务端收集线程状态、内存映射等信息，生成类似Unix core dump的文件。

#### **3. Linux平台**
- **异常类型**：  
  捕获POSIX信号，如：  
  - SIGSEGV（段错误）  
  - SIGFPE（浮点异常）  
  - SIGABRT（调用abort()）  

- **实现机制**：  
  1. **信号处理**：  
     通过`sigaction`注册信号处理函数，拦截致命信号。  
  2. **ptrace机制**：  
     使用`ptrace`系统调用附加到崩溃进程，获取详细状态信息。  
  3. **minidump生成**：  
     服务端生成类似Windows的minidump文件，包含线程堆栈和部分内存。

#### **4. Android平台**
- **异常类型**：  
  捕获Native层异常（如SIGSEGV）和Java层异常（通过JNI桥接）。  

- **实现机制**：  
  1. **Bionic C库集成**：  
     修改Bionic libc的信号处理逻辑，拦截崩溃信号。  
  2. **ART虚拟机钩子**：  
     注册Java异常处理回调，捕获未处理的RuntimeException。  
  3. **fd传递**：  
     通过Unix Domain Socket传递崩溃进程的文件描述符，用于内存分析。


### **三、异常处理流程**
1. **异常触发**：  
   应用程序执行过程中发生异常（如访问非法内存）。

2. **客户端捕获**：  
   平台特定的异常处理钩子被触发，执行Crashpad客户端回调。

3. **通知服务端**：  
   客户端通过进程间通信（IPC）通知Crashpad Handler服务端进程。  
   - Windows：命名管道（Named Pipe）  
   - macOS/Linux：Unix Domain Socket或XPC  

4. **服务端响应**：  
   服务端接收通知，通过`ptrace`（Linux）或`task_for_pid`（macOS）等机制获取崩溃进程的控制权。

5. **数据收集**：  
   服务端收集以下信息：  
   - 线程堆栈（通过 unwind 库）  
   - 内存快照（部分或全部）  
   - 寄存器状态  
   - 自定义注释（annotation）  
   - 环境变量和进程信息  

6. **转储生成**：  
   将收集的数据写入minidump文件，格式兼容Windows的.dmp文件，便于统一分析。

7. **上传处理**：  
   服务端根据配置决定是否上传崩溃报告，或存储在本地等待网络恢复。


### **四、关键技术细节**
1. **异步处理**：  
   崩溃数据收集在独立进程中进行，避免阻塞或干扰主应用。

2. **防二次崩溃**：  
   服务端进程与主应用隔离，即使处理逻辑出错也不会导致应用再次崩溃。

3. **符号化支持**：  
   崩溃转储文件包含模块信息，配合调试符号（PDB/DWARF）可还原可读堆栈。

4. **内存过滤**：  
   可配置过滤敏感内存区域（如包含密码的内存段），保护用户隐私。

5. **超时控制**：  
   对耗时操作设置超时，避免在崩溃时陷入死锁或长时间挂起。


### **五、与其他系统的对比**
| 特性                | Crashpad                     | Windows WER                 | Google Breakpad          |
|---------------------|------------------------------|-----------------------------|--------------------------|
| **跨平台支持**      | ✅ 全平台                     | ❌ 仅Windows                 | ✅ 多平台（但支持有限）   |
| **进程隔离**        | ✅ 客户端-服务端架构          | ✅                           | ❌ 单进程                |
| **异常类型覆盖**    | ✅ 全面覆盖（SEH/Mach/信号）  | ✅ Windows特定               | ✅ 基本覆盖               |
| **符号化能力**      | ✅ 内置支持                   | ✅ 需要Windows符号服务器     | ✅                       |
| **自定义数据**      | ✅ 灵活添加注释和附件         | ✅ 有限的自定义字段          | ✅                       |


### **六、常见问题与限制**
1. **权限问题**：  
   - Linux/macOS需要特殊权限才能捕获其他进程的详细信息。  
   - Android需要`android.permission.DUMP`权限。

2. **性能开销**：  
   注册异常处理会带来轻微的性能损耗，但Crashpad优化后影响极小。

3. **符号管理**：  
   需要妥善管理与发布版本对应的调试符号，否则无法解析堆栈。

4. **内核级崩溃**：  
   对于内核崩溃（如Linux OOPS），Crashpad无法直接捕获，需结合其他工具。


通过这种多层次、跨平台的异常捕获机制，Crashpad能够在各种复杂场景下可靠地收集崩溃数据，为开发者提供定位问题所需的关键信息。