#### LT和ET概念简述：

LT模式：
 当epoll_wait检测到监听文件描述符上有事件发生时通知应用程序，应用程序可以不理解处理该事件，下次调用epoll_wait时该事件还会被通告直到该事件被处理。
 LT是系统默认， 工作在这种方式下， 程序员不易出问题， 在接收数据时，只要socket输入缓存有数据，都能够获得EPOLLIN的持续通知， 同样在发送数据时， 只要发送缓存够用， 都会有持续不间断的EPOLLOUT通知。
 ET模式：
 当epoll_wait检测到事件发生告知应用程序后应用程序必须立即处理该事件，后续的epoll_wait将不会再向应用程序告知这一事件。
 而对于ET是另外一种触发方式， 比EPOLLLT要高效很多， 对程序员的要求也多些， 程序员必须小心使用，因为工作在此种方式下时， 在接收数据时， 如果有数据只会通知一次， 假如read时未读完数据，那么不会再有EPOLLIN的通知了， 直到下次有新的数据到达时为止； 当发送数据时， 如果发送缓存未满也只有一次EPOLLOUT的通知， 除非你把发送缓存塞满了， 才会有第二次EPOLLOUT通知的机会， 所以在此方式下read和write时都要处理好。

#### LT和ET的相同点：

都是通过epoll_wait从EPOLL等待队列读取激活事件。

#### LT和ET的区别：

水平触发LT和边缘触发ET的区别：只要句柄满足某种状态，水平触发就会发出通知；而只有当句柄状态改变时，边缘触发才会发出通知。也就是说LT模式下只要某个socket处于readable/writable状态，无论什么时候进行epoll_wait都会返回该socket；而ET模式下只有某个socket从unreadable变为readable或从unwritable变为writable时，epoll_wait才会返回该socket。在epoll的ET模式下，正确的读写方式为:读：只要可读，就一直读，直到返回0，或者 errno = EAGAIN;写:只要可写，就一直写，直到数据发送完，或者 errno = EAGAIN。
 LT模式：

1. 水平触发，效率会低于ET触发，尤其在大并发，大流量的情况下。但是LT对代码编写要求比较低，不容易出现问题。LT模式服务编写上的表现是：只要有数据没有被获取，内核就不断通知你，因此不用担心事件丢失的情况。LT(level triggered)是缺省的工作方式，并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表。LT模式的时候，epoll_wait 会把有事件的 file 再次加到 rdllist 列表中，以便下次epoll_wait可以再检查一遍。
2. LT 模式下，状态不会丢失，程序完全可以于 epoll 来驱动。
3. 对于长连接，大数据包应用，因为 LT模式只能设置当时感兴趣的事件(如果不写数据也设置POLLOUT的话，会导致cpu 100%) ，所以要频繁调用epoll_ctl，内核也要多次操作链表，所以效率会比ET模式低。
4. 相对于ET，LT可以减少系统调用次数。
    ET模式：
5. 边缘触发，效率非常高，在并发，大流量的情况下，会比LT少很多epoll的系统调用，因此效率高。但是对编程要求高，需要细致的处理每个请求，否则容易发生丢失事件的情况。ET(edge-triggered)是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个EWOULDBLOCK 错误）。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once),不过在TCP协议中，ET模式的加速效用仍需要更多的benchmark确认。
6. ET模式下，程序首先要自己驱动逻辑，如果遇到 EAGAIN错误的时候，就要依赖epoll_wait来驱动，这时epoll帮助程序从阻碍中脱离。
7. ET 边沿触发的 边沿是 AGAIN错误，属于下降沿触发。
8. ET 的驱动事件依靠 socket的 sk_sleep 等待队列唤醒，这只有在有新包到来才能发生，数据包导致POLLIN, ACK确认导致 sk_buffer destroy从而导致POLLOUT, 但这不是一对一得关系，是多对一（多个网络包产生一个POLLIN, POLLOUT事件）。

#### 为什么epoll默认是LT？

LT(level triggered)：LT是缺省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表。

#### ET常见错误：

recv到了合适的长度， 程序处理完毕后就epoll_wait
 这时程序可能长期阻塞，因为这时socket的 rev_buffer里还有数据，或对端close了连接，但这些信息都在上次通知了你，你没有处理完，就epoll_wait了

正确做法是：
 recv到了合适的长度， 程序处理； 再次recv, 若果是EAGAIN则epoll_wait。
 使用ET模式时， 程序自己驱动下，会发生socket被关闭的情况，这时要处理EPIPE信号和返回值。（如果不处理EPIPE那么会导致程序core掉)

总结：ET 使用准则，只有出现EAGAIN错误才调用epoll_wait。

#### 通常的误区

通常的误区是，level-trigger 模式在 epoll 池中存在大量 fd 时效率要显著低于 edge-trigger 模式。但从 kernel 代码来看，edge-trigger/level-trigger 模式的处理逻辑几乎完全相同，差别仅在于 level-trigger 模式在 event 发生时不会将其从 ready list 中移除，略为增大了 event 处理过程中 kernel space 中记录数据的大小。

 然而，edge-trigger 模式一定要配合 user app 中的 ready list 结构，以便收集已出现 event 的 fd，再通过 round-robin 方式挨个处理，以此避免通信数据量很大时出现忙于处理热点 fd 而导致非热点 fd 饿死的现象。统观 kernel 和 user space，由于 user app 中 ready list 的实现千奇百怪，不一定都经过仔细的推敲优化，因此 edge-trigger 的总内存开销往往还大于 level-trigger 的开销。

 一般号称 edge-trigger 模式的优势在于能够减少 epoll 相关系统调用，这话不假，但 user app 里可不是只有 epoll 相关系统调用吧？为了绕过饿死问题，edge-trigger 模式的 user app 要自行进行 read/write 循环处理，这其中增加的系统调用和减少的 epoll 系统调用加起来，有谁能说一定就能明显地快起来呢？

 实际上，epoll_wait 的效率是 O(ready fd num) 级别的，因此 edge-trigger 模式的真正优势在于减少了每次epoll_wait 可能需要返回的 fd 数量，在并发 event 数量极多的情况下能加快 epoll_wait 的处理速度，但别忘了这只是针对 epoll 体系自己而言的提升，与此同时 user app 需要增加复杂的逻辑、花费更多的 cpu/mem 与其配合工作，总体性能收益究竟如何？只有实际测量才知道，无法一概而论。不过，为了降低处理逻辑复杂度，常用的事件处理库大部分都选择了 level-trigger 模式。

 epoll 的 edge-trigger 和 level-trigger 模式处理逻辑差异极小，性能测试结果表明常规应用场景 中二者性能差异可以忽略；使用 edge-trigger 的 user app 比使用 level-trigger 的逻辑复杂，出错概率更高；edge-trigger 和 level-trigger 的性能差异主要在于 epoll_wait 系统调用的处理速度，是否是 user app 的性能瓶颈需要视应用场景而定，不可一概而论。

#### 事件放入EPOLL等待队列条件

对于读事件：

1. 从不可读变为可读；
2. 有新数据到来；
3. 有老数据，并且通过epoll_ctl设置EPOLL_CTL_MOD(ET模式)；
4. 数据未读完(LT模式)。
    对于写事件：
5. 从不可写变为可写；
6. 有新的可写数据出现；
7. 数据可写，并且通过epoll_ctl设置EPOLL_CTL_MOD(ET模式)；
8. 数据可写(LT模式)。

#### LT模式ET模式存在的问题

LT模式问题：
 如果可读或可写事件未处理，会频繁激活未处理事件。
 解决方法：
 不想处理读写事件时， 从EPOLL中移除句柄。需要处理时再加入EPOLL。
 ET模式问题：
 如果可读或可写没有全部处理，会有老数据残留。要等待新数据到来。
 解决方法：

1. 循环读取或写入数据，一直到EAGAIN或EWOULDBLOCK；
2. 读取或者写入数据后，通过epoll_ctl设置EPOLL_CTL_MOD，激活未处理事件。

#### accept 要考虑 的两个问题



 **(1) 阻塞模式 accept 存在的问题**

考虑这种情况：TCP连接被客户端夭折，即在服务器调用accept之前，客户端主动发送RST终止连接，导致刚刚建立的连接从就绪队列中移出，如果套接口被设置成阻塞模式，服务器就会一直阻塞在accept调用上，直到其他某个客户建立一个新的连接为止。但是在此期间，服务器单纯地阻塞在accept调用上，就绪队列中的其他描述符都得不到处理。 解决办法是把监听套接口设置为非阻塞，当客户在服务器调用accept之前中止某个连接时，accept调用可以立即返回-1，这时源自Berkeley的实现会在内核中处理该事件，并不会将该事件通知给epool，而其他实现把errno设置为ECONNABORTED或者EPROTO错误，我们应该忽略这两个错误。

 **(2)ET模式下accept存在的问题**

 考虑这种情况：多个连接同时到达，服务器的TCP就绪队列瞬间积累多个就绪连接，由于是边缘触发模式，epoll只会通知一次，accept只处理一个连接，导致TCP就绪队列中剩下的连接都得不到处理。 解决办法是用while循环抱住accept调用，处理完TCP就绪队列中的所有连接后再退出循环。如何知道是否处理完就绪队列中的所有连接呢？accept返回-1并且errno设置为EAGAIN就表示所有连接都处理完。
 综合以上两种情况，服务器应该使用非阻塞地accept