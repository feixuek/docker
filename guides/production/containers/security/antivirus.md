# 防病毒软件和`Docker`

当防病毒软件扫描`Docker`使用的文件时，这些文件可能会以导致`Docker`命令挂起的方式被锁定。

减少这些问题的一种方法是将`Docker`数据目录（`/var/lib/docker`在`Linu`x，`%ProgramData%\dockerWindows Server`或`$HOME/Library/Containers/com.docker.docker/``Mac`上）添加到防病毒的排除列表中。但是，这需要权衡取舍，即不会检测到`Docker`镜像中的病毒或恶意软件，容器的可写层或卷。如果确实选择从后台病毒扫描中排除`Docker`的数据目录，则可能需要安排一个重复执行的任务，该任务将停止`Docker`，扫描数据目录，然后重新启动`Docker`。