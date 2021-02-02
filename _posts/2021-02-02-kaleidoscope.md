---
layout: post
title: "镜里万花"
description: "探索 Android Runtime Hook 实现方式"
cover_url: /assets/covers/kaleidoscope.webp
cover_meta: illustration by [mocha](https://www.pixiv.net/artworks/84661248)
tags: 
  - Develop
  - Android
---

性能优化这一主题里，对资源分配相关函数进行监控是一种常用的性能问题检测方式。

本篇文字是近期探索部分 Android Runtime Hook 框架实现方式的记录，用于帮助理解相关技术的实现原理。

## Method in Runtime

Java 层中，每一个方法在 Android Runtime 中均对应着一个 C++ 对象，该类型为 `art::ArtMethod`，它在虚拟机实现方法执行的过程中至关重要。

`art::ArtMethod` 的声明如下：

``` c++
namespace art {
...
class ArtMethod final {
  ...
  protected:
    ...
    struct PtrSizedFields {
      ...
      void* entry_point_from_quick_compiled_code_;
    } ptr_sized_fields_;
}
...
}
```

其中，指针字段 `entry_point_from_quick_compiled_code_` 指向的正是方法的入口，因此一个 Hook 方案出现了：只需要将原函数该指针设置为目标函数该指针的值即可。这一方案称为**入口替换**。

### 全量替换

入口替换看似是一个简单易行的方法，但它存在这一个难点：如何找到指针 `entry_point_from_quick_compiled_code_` 在对象中的位置？`art::ArtMethod` 是一个对外隐藏的类型，外部无法得知字段地址相对于对象地址的偏移。

目前的解决方案大致为三种：

- 通过 AOSP 所开源的代码构建一个完全相同的类型，代表框架为 AndFix。这一方案最大的问题是不具备兼容性，当手机厂商自行对源码进行更改时，类型映射错误便会导致出错。

- 通过预测数据与字段相对位置推算偏移，代表框架为 SandHook。相比第一种将偏移写死的方案，该方案可以更好地实现碎片化环境下的兼容，但仍不能完全保证在各种情况下均获取到正确的偏移。

既然无法准确获取偏移，热修复框架 Sophix 便直接忽略对字段的偏移计算，而是直接替换整个 `art::ArtMethod` 对象，这便是**全量替换**。

> 实际方法执行过程中可能会依赖其它字段，因此，以 AndFix 为例，替换过程中并不会仅替换入口指针，而是会将所有字段都进行替换，这也是 AndFix 的优化版本 —— Sophix 采用全量替换的原因。

全量替换绕过了计算偏移的难点，但又产生了一个新的问题需要解决。

### 替换粒度

前文已经提到过，`art::ArtMethod` 是一个对外隐藏的类型，所以无法通过 `sizeof()` 来获取其对象大小。为了获取全量替换的替换粒度，必须另辟蹊径。

在 Android Runtime 中，初始化类时会以数组的方式分配所有方法 `art::ArtMethod` 对象的空间。

``` c++
void ClassLinker::LoadClass(Thread* self,
                            const DexFile& dex_file,
                            const dex::ClassDef& dex_class_def,
                            Handle<mirror::Class> klass) {
      ...
      klass->SetMethodsPtr(
          AllocArtMethodArray(self, allocator, accessor.NumMethods()),
          accessor.NumDirectMethods(),
          accessor.NumVirtualMethods());
      ...
}
```

这意味着，数组中相邻两个对象的地址差值，便是对象的大小。正因如此，我们可以构造一个特殊的类来获取这个差值。

``` java
public class ClassForCalculateOffset {
  public static final void start() {}
  public static final void end() {}
}
```

通过对两个方法对应 `art::ArtMethod` 对象的地址差值计算，便可以获取到当前设备 `art::ArtMethod` 对象的大小，且无需关注该类型的具体结构。

### 如何获取 ArtMethod 的地址

在 Android 11 前，获取方法对应的 `art::ArtMethod` 对象地址较为简单，由方法反射对象通过 JNI 调用所得的 `jmethodID` 便为 `art::ArtMethod` 对象的地址。

``` c++
art::ArtMethod *GetArtMethodFromReflectMethod(JNIEnv *env, jobject reflect_method) {
    jmethodID reflect_method_id = env->FromReflectedMethod(reflect_method);
    return reinterpret_cast<art::ArtMethod *>(reflect_method_id);
}
```

Android 11 无法通过 JNI 获取，但在 Java 层方法 `Method` 的父类 `Executable` 中仍保留一个字段记录着 `art::ArtMethod` 对象的地址。

``` c++
art::ArtMethod *GetArtMethodFromReflectMethodOnR(JNIEnv *env, jobject reflect_method) {
    jclass executable_class = GetJvmExecutableClass();
    jfieldID art_method_field_id = env->GetFieldID(executable_class, "artMethod", "J");
    jlong art_method_pointer = env->GetLongField(reflect_method, art_method_field_id);
    return reinterpret_cast<art::ArtMethod *>(art_method_pointer);
}
```

## 替换的局限性

当然事情远没有我们想象的那么简单 ( ＿ ＿)ノ｜。

无论是入口替换还是全量替换，本质上都是通过替换入口指针来实现 Hook，但并不一定所有方法的 `art::ArtMethod` 对象都存在着一个有效的入口。

### Invoke Virtual

public 方法，无论为类型自行声明还是实现父类，均为虚方法，需要在运行时确定实际被调用的方法对象。

当 Android Runtime 通过 `invoke-virtual` 对虚方法进行调用时，会检测调用者的类型，并以此从虚方法表中获取实际调用的方法。而对于本身不存在实现的虚方法，方法的实际调用并不会使用到其 `art::ArtMethod` 对象，所以对其进行入口替换并不能起到作用。

### Just-In-Time & Ahead-Of-Time

在 Android 5 之前，Android 采用的 Java 虚拟机为即时编译（Just-In-Time Compile，即 JIT）的 Dalvik，字节码在运行期间被翻译为机器码执行，使热点代码无需进行重复的解释操作，提升运行速度。

Android 5 采用 Android Runtime（ART）后，为进一步提升运行性能，也为了解决 JIT 运行期编译对资源的消耗，采用的是预先编译（Ahead-Of-Time Compile，即 AOT）。应用在安装时，dex 文件便被编译为可执行的 oat 文件，从而保证应用程序运行时的效率最大化。但 AOT 也存在不少问题，额外翻译的 oat 文件会占用磁盘空间不说，提前编译导致应用安装时间极大地增长，当手机重启时所有应用编译甚至会花费数十分钟。

那一天人们又回想起了盯着转圈的屏幕内心跑过数万头羊驼的恐惧。

为了解决这些会显著拉升用户血压的问题，Android 7 之后采用了 JIT + AOT 的混合编译模式，只有调用次数触发阈值的热点代码会采用 AOT 进行编译，且编译会在设备充电等空余时间进行，而非热点代码运行则采用 JIT 进行编译。两种编译方式相互结合，扬长避短，有效改善了用户的使用体验。

![Jit Workflow](/assets/posts/kaleidoscope/jit-workflow.webp)

<center style="font-size:14px;">Android Developer 官方文档展示的 JIT 工作流程图</center>

但 Android Runtime 对 JIT 的引入，极大地增加了虚拟机实现的复杂度。

如果尝试比较两个未编译方法的入口指针指向地址，会发现它们是相同的。当应用代码在 JIT 模式下运行时，`art::ArtMethod` 中的入口指针指向的是一个用于解释执行跳板，而非方法代码的入口，仅当方法触发热点阈值被编译时，方法的实际代码才会被生成，入口指针才能指向一个有效的位置。

除了入口无效之外，还有一个问题，JIT 编译导致方法的入口并不是固定的，倘若程序跑到一半热点编译被触发了，原有的替换便会失效。

### Sharpening

Android 8 后的 Android Runtime 采用了一个更激进的优化策略 —— Sharpening。

以位于 boot.oat 中的 Android 系统函数为例，文件在操作系统启动时便被加载到 Image Space 中，它们在内存中的位置是绝对的，所以 JIT 或 AOT 编译代码时便不再从方法的 `art::ArtMethod` 对象中获取入口，而是直接写死在编译完成的机器码中。

查找过程不存在了，替换也就不再有意义。

## 强制编译

为了避免 JIT 编译导致入口变动，在替换前必须通过对方法进行强制编译来固定方法入口。

得益于编译器库 libart-compiler.so 与虚拟机库 libart.so 的分离，我们可以采用 `dlsym()` 获取到 libart-compiler.so 暴露的函数符号，其中就有着我们所需要的方法编译函数。在 Android 11 之前，这个函数为 `jit_compile_method()`，Android 11 中，这个函数为 `art::jit::JitCompile::CompileMethod()`。

``` c++
void *jit_so_handle = DynamicLibraryOpen("libart-compiler.so", RTLD_NOW);

if (GetAndroidVersion() >= AndroidVersion::kR) {
    compile_method_ =
            dlsym(jit_so_handle, "_ZN3art3jit11JitCompiler13CompileMethodEPNS_6ThreadEPNS0_15JitMemoryRegionEPNS_9ArtMethodEbb");
} else {
    compile_method_ = dlsym(jit_so_handle, "jit_compile_method");
}
```

> Android 7 后 Google 对 `dlsym()` 函数调用进行了限制，需手动解析 ELF。目前 GitHub 已有相关的开源框架，可见 [Nougat_dlfunctions](https://github.com/avs333/Nougat_dlfunctions)。

## Inline

尽管虚拟机未必会从 `art::ArtMethod` 对象中获取代码入口，但入口指针所指向的那些机器码是肯定会被使用到的，因此出现了有别于替换入口的另一个方案，也是各个 Android Runtime Hook 框架所使用的方案：不替换入口，而替换入口指针指向的机器码。

这个方案被称为 **dynamic callee-side rewriting**，本文以框架 [Epic](https://github.com/tiann/epic) 源码中 ARM 64 部分进行说明，其它架构在此就不做讨论了。

在聊这一方案之前，先聊一聊关于 ARM 的一些内容。

### 浅谈 ARM

ARM 是一个精简指令集处理器架构，具有等长指令与大量寄存器的特点，这里主要说明以下几点：

- ARM 采用 Load/Store 架构，内存单元中的数据无法直接参与计算，需通过 Load/Store 指令将数据从内存单元中读取到寄存器中进行操作。

- pc（Program Counter）寄存器为程序计数器，用于记录程序的执行地址，指向当前正在执行指令的下一条指令。

- sp（Stack Pointer）寄存器用于记录当前栈的栈顶地址。

- cpsr（Current Program Status Register）寄存器用于记录当前程序的执行状态，各指令可通过设置 Condition Field 结合 cpsr 寄存器数据以判断是否需要执行。

- Thumb（Thumb 16）是 ARM 32 指令集下的一个子集，指令长度由 32 位缩为 16 位。Thumb 模式下的指令虽然功能性更少，但可以提供整体更佳的编码密度。

- Thumb-2（Thumb 32）是 Thumb 的扩展，以额外的 32 位指令让 Thumb 指令集的使用更广泛。

### 跳板

通过强制编译确定方法代码入口后，即可通过修改内存的方式，将入口指针所指向的数据替换为类似下面的一段跳板代码（原数据备份）。插入跳板代码的目的是为了改变程序的执行流程，转为执行准备完成的用于分发 Hook 逻辑的代码。跳板代码需要尽可能短，只有在小于原代码体积的情况下才可成功插入。

``` armasm
  ldr x9, _target ; 将 _target 中的数据写入 x9 寄存器
  br x9           ; 跳转到 x9 寄存器所指向的指令并继续执行程序
_target:
  .quad 0x0       ; 8 Byte 长度的空位，在插入跳板时会被替换为二段跳板的地址
```

> ARM 64 相比 ARM 32，pc 寄存器不能用作计算指令的源或目的地，也不可用作加载或存储指令，因此 Epic 在 ARM 32 下的跳板逻辑与 ARM 64 下会稍有不同。

上方的代码最终会被编译为一段长 16 Byte 的数据。

``` text
0x50, 0x00, 0x00, 0x58,                         ; ldr x9, _target
0x00, 0x02, 0x1F, 0xD6,                         ; br x9
0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00  ; _target
```

### 函数分发

成功执行跳板代码的程序会到达设置好的二段跳板，目的有二：对入口相同的代码进行逻辑分发，并在保持栈平衡的情况下打包方法参数。

JNI 方法、未完成加载的 static 方法（static 方法是懒加载的）等，它们可能存在着相同的入口，所以二段跳板中需要存在判断逻辑确定方法是否需要 Hook。

无论是一段跳板还是二段跳板，为了使虚拟机在栈回溯时不发生错误，跳板代码中都不可以对栈空间和 sp 进行操作，因此需要通过结构体来保存原调用参数等需要获取的数据。

``` armasm
  nop                             ; 空指令，等待一个指令周期

  ; 判断当前方法是否需要 Hook，用于对入口相同的代码进行逻辑分发
  ldr x9, _source_art_method      ; 将原方法 ArtMethod 对象地址写入 x9 寄存器
  cmp x0, x9                      ; 判断调用方法与 Hook 原方法是否相等
  bne _ignore_hook                ; 如果不相等，跳转到 _ignore_hook，执行原方法

  ldr x0, _target_art_method
  ldr x9, _data_struct

  ; 写入 sp 指针信息到 _data_struct 指向的结构体，用于在目标方法中获取原调用的所有参数
  mov x10, sp                     ; x10 = sp
  str x10, [x9, #0]               ; 将 x10 寄存器的值写入地址为 x9 + 0 的内存中

  ; 写入参数信息到 _data_struct 指向的结构体
  str x2, [x9, #8]
  str x3, [x9, #16]
  mov x3, x9                      ; 将结构体地址作为目标调用的参数

  ; 写入原方法 ArtMethod 对象地址到 _data_struct 指向的结构体
  ldr x2, _source_art_method
  str x2, [x9, #24]

  ; 将当前线程信息作为目标调用的参数，用于在目标方法中还原保存在 Thread Local 内存中的数据
  mov x2, x19

  ; 写入并跳转到 _target_entry
  ldr x9, _target_entry
  br x9

_target_art_method:
  .quad 0x0
_target_entry:
  .quad 0x0
_source_art_method:
  .quad 0x0
_data_struct:
  .quad 0x0
```

> Android Runtime 中对方法的调用存在一定规则，这里讲一下 ARM 64。
>
> 除了栈保存有调用的参数之外，x0 ~ x3 也记录着调用的前四个参数（这里的参数顺序不针对 Java 方法），而存有重复数据的 sp ~ sp + 24 这段空间会被虚拟机所使用，因此无法对其进行操作。
>
> 在虚拟机调用方法的过程中，x0 保存被调用方法（Callee Method）的 `art::ArtMethod` 对象地址，x19（又称 tr 寄存器）保存当前进程的 native peer。
>
> x1 在调用非 static 方法时保存调用对象的地址（即方法的 this），否则用于保存参数。

至此，Epic 成功实现了对方法执行逻辑的替换，程序将转到 Java 层的目标方法执行。

### Stop The World

为了避免虚拟机 JIT 与代码修改过程发生冲突导致崩溃，Hook 过程中需要暂停所有其它的线程。

Android Runtime 中存在着暂停/恢复所有线程的函数：`ThreadList.SuspendAll()`/`ThreadList.ResumeAll()`。虽然作为 `ThreadList` 的成员函数，我们无法直接调用，但好在源码中采用 RAII 对其进行了封装，即 `ScopedSuspendAll`，它的构造函数与析构函数实现了对上述两个函数的调用，我们可以通过 `dlsym()` 获取到它们。

更进一步地，我们也可以通过 RAII 对获取到的函数再次封装。

``` c++
void *art_so_handle = dlopen("libart.so", RTLD_NOW);
void *suspend_function_ =
        DynamicLibrarySymbol(art_so_handle, "_ZN3art16ScopedSuspendAllC1EPKcb");
void *resume_function_ =
        DynamicLibrarySymbol(art_so_handle, "_ZN3art16ScopedSuspendAllD1Ev");

ScopedSuspendAll::ScopedSuspendAll(const char *cause) {
    if (suspend_function_) {
        suspend_function_(this, cause);
    }
}

ScopedSuspendAll::~ScopedSuspendAll() {
    if (resume_function_) {
        resume_function_(this);
    }
}
```

## 参考框架

- [weishu - Epic](https://github.com/tiann/epic)

- [ganyao - SandHook](https://github.com/ganyao114/SandHook)

- [rk700 - YAHFA](https://github.com/PAGalaxyLab/YAHFA)

- [canyie - Pine](https://github.com/canyie/pine)

## 参考链接

- [weishu - 论 ART 上运行时 Method AOP 实现](http://weishu.me/2017/11/23/dexposed-on-art/)

- [weishu - ART 深度探索开篇：从 Method Hook 谈起](http://weishu.me/2017/03/20/dive-into-art-hello-world/)

- [万壑 - Android 热修复升级探索 —— 追寻极致的代码热替换](https://developer.aliyun.com/article/74598#)

- [Wißfeld, Marvin - ArtHook: Callee-side Method Hook Injection on the New Android Runtime ART](https://publications.cispa.saarland/143/)

- [ganyao - Android ART Hook 实现](https://blog.csdn.net/ganyao939543405/article/details/86661040)

- [canyie - ART 上的动态 Java 方法 Hook 框架](https://blog.canyie.top/2020/04/27/dynamic-hooking-framework-on-art/)

- [rk700 - 在 Android N上对 Java 方法做 Hook 遇到的坑](https://rk700.github.io/2017/06/30/hook-on-android-n/)

- [Android Developer - 实现 ART 即时 (JIT) 编译器](https://source.android.com/devices/tech/dalvik/jit-compiler)

- [arm - ARM Architecture Reference Manual](https://documentation-service.arm.com/static/5f8dacc8f86e16515cdb865a)