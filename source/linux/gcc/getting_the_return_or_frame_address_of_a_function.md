# [译]获取函数返回地址或者函数调用栈帧

翻译自: [Getting the Return or Frame Address of a Function](https://gcc.gnu.org/onlinedocs/gcc/Return-Address.html)

以下几个函数可用于获取函数的调用者信息.

## 内置函数: `void * __builtin_return_address (unsigned int level)`
这个函数返回当前函数或者调用它的一个函数(父函数, 爷爷函数等等...)的返回地址. 参数 `level` 是函数调用栈上对应的帧. 当其值 为 `0` 时返回当前函数的返回地址, 为 `1` 时返回调用当前函数的函数的返回地址, 后面的以此类推. 对于内联函数, 期望的行为是调用内联函数的函数的返回地址. 为了避免这种行为, 请在非内联函数中使用该函数.

参数 `level` 必须是整数常量(译注: 不能使用循环变量).

在一些机器上, 这个函数可能只能获取当前函数的返回地址, 无法获取父辈及其以上函数的返回地址; 对于这种情况, 或则已经到达函数调用栈的最顶层, 这个函数将返回 0 或者一个随机值.
因此, 应该使用 `__builtin_frame_address` 函数确定是否达到栈顶端.

另外, 该函数的返回值可能需要经过额外处理, 详情参阅 `__builtin_extract_return_addr`.

调用该函数时使用非 0 参数可能会有无法预期的现象, 例如导致调用者程序崩溃.
因此, 调用该函数被认为是不安全的, 只有在调试 `-Wframe-address` 选项时才是必须的(译注: 原文为 As a result, calls that are considered unsafe are diagnosed when the -Wframe-address option is in effect). 这种调用应该只存在调试环境中.


## 内置函数: `void * __builtin_extract_return_addr (void *addr)`
函数 `__builtin_return_address()` 返回的地址可能是经过编码的, 需要使用该函数得到真正的地址. 例如, 在 32 位 S/390 平台上, 最高位被屏蔽; 或者在 SPARC 平台上, 有一个偏移应用于将要被执行的下一条执行.

如果不需要修复, 该函数只是简单返回传递给它的参数.


## 内置函数: `void * __builtin_frob_return_address (void *addr)`
这个函数功能与上一个函数 `__builtin_extract_return_addr()` 相反.


## 内置函数: `void * __builtin_frame_address (unsigned int level)`
这个函数与 `__builtin_return_address()` 类似, 但是它返回的是函数栈帧的地址, 而不是函数的返回地址. 调用 `__builtin_frame_address` 时, 传递参数 0 表示获取当前函数栈帧地址, 传递参数 1 表示获取父函数的帧地址, 以此类推.

函数帧存储着局部变量和寄存器值. 函数帧地址是函数第一个被压入栈的地址. 然而,
这个确切的定义依赖处理器和函数调用约定. 如果处理器有一个专门用于寄存器的帧指针,
函数也有对应的帧指针, 则 `__builtin_frame_address()` 返回寄存器的帧指针.

在某些机器上, 该函数可能只能获取当前函数的帧指针, 而无法获取父函数及其以上函数的帧指针(与 `__builtin_return_address()` 类似);
在这种情况下, 或者, 已经到达函数栈的顶端, 如果在函数栈的第一帧被正确初始化的情况下(译注: 正常情况下),
该函数将返回 0.

调用该函数时使用非 0 参数可能会有无法预期的现象, 例如导致调用者程序崩溃.
因此, 调用该函数被认为是不安全的, 只有在调试 `-Wframe-address` 选项时才是必须的(译注: 原文为 As a result, calls that are considered unsafe are diagnosed when the -Wframe-address option is in effect). 这种调用应该只存在调试环境中.
