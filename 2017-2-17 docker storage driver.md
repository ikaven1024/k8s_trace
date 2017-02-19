# docker存储

docker对镜像和容器，采用层叠加和写时复制（CoW）技术。在不同的存储驱动下其实现方式不同，也具有不同的特性。没有单一的驱动适合所有的应用场景，要根据不同的场景选择合适的存储驱动，才能有效的提高Docker的性能。如何选择适合的存储驱动，要先了解存储驱动原理才能更好的判断。本文就此展开谈论。



TODO: delete it

**节省镜像空间**

**提高容器效率**

Containers that write a lot of data will consume more space than containers that do not. This is because most write operations consume new space in the container’s thin writable top layer. If your container needs to write a lot of data, you should consider using a data volume. （比如一个镜像中有一个大文件，许多容器对他进行写，那么会产生许多个副本，存储剧增，可以使用devicemapper）



A copy-up operation can incur a noticeable performance overhead. This overhead is different depending on which storage driver is in use. However, large files, lots of layers, and deep directory trees can make the impact more noticeable. Fortunately, the operation only occurs the first time any particular file is modified. Subsequent modifications to the same file do not cause a copy-up operation and can operate directly on the file’s existing copy already present in the container layer.



## AUFS

AUFS是docker最早采用，也是目前最成熟的存储驱动。AUFS能透明覆盖一或多个现有文件系统的层状文件系统，把多层合并成文件系统的单层表示。简单来说就是支持将不同目录挂载到同一个虚拟文件系统下的文件系统。这种文件系统可以一层一层地叠加修改文件。无论底下有多少层都是只读的，只有最上层的文件系统是可写的。当需要修改一个文件时，AUFS创建该文件的一个副本，使用CoW将文件从只读层复制到可写层进行修改，结果也保存在可写层。在Docker中，底下的只读层就是image，可写层就是container。结构如下图所示：

[](https://docs.docker.com/engine/userguide/storagedriver/images/aufs_layers.jpg)

不同容器可以共享相同的镜像层。有以下几个

- 容器启动快。因为不同容器可以共享相同的镜像层，所以容器启动过程中不必再重新启动已存在的镜像。
- 节省存储空间
- 节省内存

**读写**

逐层搜索文件，直到找到文件，有一定的操作时延。对文件进行修改时，需要将文件复制到最上的读写层，不过以后的对此文件的读写都直接在最上层上进行。另外应为操作对象时文件，即使是对大文件进行某小部分的修改，都需要将文件全部复制到最上面的读写层。而且多个容器都对文件修改时，都会在各自的读写层上复制一个独立的副本。在文件大，容器多的场景，会有不少存储空间损耗。

**删除**

直接在容器层标记文件删除，速度快。

**相容性**

不完全支持`rename(2)`。会有 `EXDEV` (“cross-device link not permitted”)错误，即使源路径和目标路径在同一AUFS层。

**使用场景**

适合于容器密集环境，比如PaaS环境。

## Overlay

Overlay类似于AUFS，相比有以下不同： 

- 设计更简单
- 自3.18已合入Linux内核主线 
- 更快


和AUFS的多层不同的是overlay只有两层：一个 upper 文件系统和一个 lower 文件系统，分别代表Docker的镜像层和容器层。当需要修改一个文件时，使用CoW将文件从只读的lower 复制到可写的upper进行修改，结果也保存在 lower 层。在Docker中，底下的只读层就是image，可写层就是container。结构如下图所示：

[](https://docs.docker.com/engine/userguide/storagedriver/images/overlay_constructs.jpg)

`overlay`只有两层，也就意味着并不能实现多层镜像。. Instead, each image layer is implemented as its own directory under `/var/lib/docker/overlay`. Hard links are then used as a space-efficient way to reference data shared with lower layers. 

**overlay2**

从Docker1.12开始支持`overlay2`。overlay2在Linux4.0后提供，性能比overlay更好，消耗更少的inode.



**性能**

因为层级少，消耗更少的搜索时间，所以比AUFS有更少的读写时延。

OverlayuFS支持页缓存共享。多容器访问同一文件能够共享页缓存，使得`overlay`/`overlay2`内存上更加高效。

 OverlayFS supports page cache sharing. This means multiple containers accessing the same file can share a single page cache entry (or entries). This makes the `overlay`/`overlay2` drivers efficient with memory and a good option for PaaS and other high density use cases.

`overlay`极快消耗inode，特别在镜像多、容器多的机器上，很容易造成inode耗尽。`overlay2`没有这问题。

**相容性**

- **open(2)**. OverlayFS only implements a subset of the POSIX standards. This can result in certain OverlayFS operations breaking POSIX standards. One such operation is the *copy-up* operation. Suppose that your application calls `fd1=open("foo", O_RDONLY)` and then `fd2=open("foo", O_RDWR)`. In this case, your application expects `fd1` and `fd2` to refer to the same file. However, due to a copy-up operation that occurs after the first calling to `open(2)`, the descriptors refer to different files.


- **rename(2)**. 调用`reanme(2)`目录，只允许源路径和目标路径在都在上层，否则返回 `EXDEV` (“cross-device link not permitted”)错误。





[](https://docs.docker.com/engine/userguide/storagedriver/images/overlay_constructs.jpg)

每个镜像层在宿主机上表现为一个目录`/var/lib/docker/overlay/<ID>`。目录下包含完整的文件系统。不同的镜像层共享同一个文件，通过硬链接方式实现。`overlay` driver only works with a single lower OverlayFS layer and hence requires hard links for implementation of multi-layered images





## Devicemapper

`devicemapper`驱动存储每个镜像和容器在各自的虚拟设备中，这些设备是按需分配的。Device Mapper技术操作以块为单位，而不是文件，这就意味着修改文件时只要复制其中的块，而不需要复制整个文件。

![](https://docs.docker.com/engine/userguide/storagedriver/images/base_device.jpg)

为了开箱即用，docker的devicemapper默认采用`loop-lvm`模式，而这种模式只能用在测试环境。在生产环境中需要使用成`deirect-lvm`

**性能**

写新文件时采用写时分配策略。以64k为单位分配空间并完成挂在到读写层上。若每次写到的数据很小，远远少于64k，会造成存储空间的浪费。

修改镜像中的文件以写时复制策略。以64k为单位复制数据块到读写层。对大文件只需要复制其中的数据块，十分高效。然而对大量的小数据修改，性能不比AUFS。

总结：适合大文件，不适合小文件



## Overlay

The `overlay` driver has known limitations with inode exhaustion and commit performance. The `overlay2` driver addresses this limitation, but is only compatible with Linux kernel 4.0 and later.



## Data volume

Data volumes are not controlled by the storage driver. Reads and writes to data volumes bypass the storage driver and operate at native host speeds.



## 选择合适的驱动



[](http://img1.tuicool.com/Ajuaiyq.png!web)

[](https://docs.docker.com/engine/userguide/storagedriver/images/driver-pros-cons.png)







