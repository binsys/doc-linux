GCC 中的编译器堆栈保护技术
================================================================================

以堆栈溢出为代表的缓冲区溢出已成为最为普遍的安全漏洞。由此引发的安全问题比比皆是。早在 1988 年，
美国康奈尔大学的计算机科学系研究生莫里斯 (Morris) 利用 UNIX fingered 程序的溢出漏洞，写了一段恶意
程序并传播到其他机器上，结果造成 6000 台 Internet 上的服务器瘫痪，占当时总数的 10%。各种操作系统
上出现的溢出漏洞也数不胜数。为了尽可能避免缓冲区溢出漏洞被攻击者利用，现今的编译器设计者已经开始
在编译器层面上对堆栈进行保护。现在已经有了好几种编译器堆栈保护的实现，其中最著名的是 StackGuard
和 Stack-smashing Protection (SSP，又名 ProPolice）。

编译器堆栈保护原理
--------------------------------------------------------------------------------

参考资料
--------------------------------------------------------------------------------

* http://www.ibm.com/developerworks/cn/linux/l-cn-gccstack/