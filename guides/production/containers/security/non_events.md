# `Docker`非事件安全

该页面列出了`Docker`缓解的安全漏洞，因此在`Docker`容器中运行的进程永远不会受到`bug`的攻击-即使在修复该`bug`之前。假设容器在运行时未添加任何额外功能，或者未以身份运行`--privileged`。

下面的列表甚至还不完整。相反，它只是我们实际上已经注意到引起安全审查和公开披露的漏洞的少数错误的示例。极有可能，尚未报告的错误远远超过已报告的错误。幸运的是，由于`Docker`的默认方法是通过`apparmor`，`seccomp`和`dropping`功能来确保安全，因此它可以像已知漏洞一样缓解未知漏洞。

漏洞修复：

- CVE-2013至1956年， 1957年， 1958年， 1959年， 1979年， CVE-2014-4014， 5206， 5207， 7970， 7975， CVE-2015至2925年， 8543， CVE-2016-3134， 3135，等：引进非特权用户名称空间通过为此类用户提供对诸如`root`用户之类的先前仅系统调用的合法访问权限，极大地增加了非特权用户可用的攻击面`mount()`。由于引入了用户名称空间，所有这些`CVE`都是安全漏洞的示例。`Docker`可以使用用户名称空间来设置容器，但是随后不允许容器内部的进程通过默认的`seccomp`配置文件创建其自己的嵌套名称空间，从而使这些漏洞无法利用。
- CVE-2014-0181， CVE-2015-3339：这些是需要存在`setuid`二进制文件的错误。`Docker`通过`NO_NEW_PRIVS`进程标志和其他机制禁用容器内的`setuid`二进制文件。
- CVE-2014-4699：中的错误`ptrace()`可能允许特权升级。`Docker ptrace()` 使用`apparmor`，`seccomp`并通过`drop` 禁用容器内部`CAP_PTRACE`。那里有三倍的保护层！
- CVE-2014-9529：一系列精心设计的`keyctl()`调用可能导致内核`DoS` /内存损坏。`Docker keyctl()`使用`seccomp` 禁用容器内部。
- CVE-2015-3214， 4036：这些都是常见的虚拟化驱动程序中存在错误，可能允许来宾操作系统用户在主机操作系统上执行代码。利用它们需要访问来宾中的虚拟化设备。当不使用`Docker`运行时，`Docker`隐藏对这些设备的直接访问`--privileged`。有趣的是，这些情况似乎是容器比虚拟机“更安全”的情况，这违背了虚拟机比容器“更安全”的常识。
- CVE-2016-0728：精心制作的`keyctl()`调用导致的使用后使用可能导致特权提升。`Docker keyctl()`使用默认的`seccomp`配置文件禁用内部容器。
- CVE-2016-2383：eBPF中的一个错误-用于表达`seccomp`过滤器之类的特殊内核内`DSL-`允许任意读取内核内存。该`bpf()`系统呼叫被封锁使用（讽刺）的`Seccomp Docker`容器内部。
- CVE-2016-3134， 4997， 4998：在`setsockopt`的一个`bug`用`IPT_SO_SET_REPLACE`，`ARPT_SO_SET_REPLACE`以及 `ARPT_SO_SET_REPLACE`造成内存损坏/本地权限提升。这些参数被阻止`CAP_NET_ADMIN`，默认情况下`Docker`不允许。

错误未缓解：

- CVE-2015-3290， 5157：在内核的非屏蔽中断处理允许权限提升缺陷。可以在`Docker`容器中利用，因为`modify_ldt()`当前未使用`seccomp`阻止系统调用。
- CVE-2016-5195：在`Linux`内核的内存子系统处理私有只读内存映射的写时复制（`COW`）中断的方式中发现了一种竞争状况，这允许无特​​权的本地用户获得对只读磁盘的写访问权限只有记忆。也称为“脏牛”。 部分缓解：在某些操作系统上，`seccomp`筛选`ptrace`和`/proc/self/mem`只读事实相结合可缓解此漏洞。