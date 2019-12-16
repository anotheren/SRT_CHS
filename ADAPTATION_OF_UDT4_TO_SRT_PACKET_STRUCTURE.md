# Adaptation of UDT4 to SRT Packet Structure **从 UDT4 到 SRT 数据包结构的改进**

UDT is an ARQ (Automatic Repeat reQuest) protocol. It implements the third evolution of ARQ (Selective Repeat). The UDT version 4 (UDT4) implementation was proposed as “informational” to the IETF but remained a draft (draft-gg-udt-03).

UDT 是一种 ARQ（自动重复请求）协议。它实现了 ARQ（选择性重复）的第三个演进版本。UDT version 4（UDT4）的实现被 IETF 推荐为的国际性标准，但当前仍是草案（draft-gg-udt-03）。

UDT is aimed at the maximum use of capacity, so an application must ensure input buffers are always available when it is time to send data. When sending real-time video at a reasonable bit rate, the speed of packet generation is slow compared to reading a file. Buffer depletion causes some resets in the UDT transmission algorithm. Also, when congestion occurred the sender algorithm could block the UDT API, preferring to transmit the packets in the loss list, so that new video frames could not be processed. Real-time video cannot be suspended, so packets would be dropped by the application (since the transmission API was being blocked). Unlike UDT, SRT shares the available bandwidth between real-time and retransmitted packets, losing or dropping older packets instead of newer.

UDT 旨在最大程度地利用网络带宽，因此任何应用程序发送数据时，必须始终保持输入缓冲区可用。与读取文件相比，当以合理的比特率发送实时视频时，数据包生成的速度较慢。缓冲区耗尽会导致 UDT 传输算法中的某些复位。同样，当发生拥塞时，发送方的算法可能会阻塞 UDT 的 API，并优先发送丢失列表中的数据包，从而无法处理新的视频帧。实时视频无法暂停，因此应用程序将丢弃数据包（因为传输的 API 被阻塞）。与 UDT 不同，SRT 在实时数据包和重传数据包之间共享可用带宽，丢失或废弃旧的数据包而不是新到的数据。

The initial development of SRT involved a number of changes and additions to UDT version 4, primarily:

SRT 的初始开发涉及对 UDT version 4 的许多更改和补充，主要是：

* Statistics counters in bytes **统计计数器（以字节为单位）**
* Buffer sizes in milliseconds to control Latency (measuring buffers in time units is easier for configuring latency, and is independent of stream bitrate) **缓冲区大小（以毫秒为单位）以控制延迟（以时间为单位测量缓冲区更易于配置延迟，并且与流比特率无关）**
* Statistics in ACK messages (receive rate and estimated link capacity) **ACK 消息中的统计信息（接收速率和估计的链路容量）**
* Control packet timestamps (this is in the UDT draft but not in the UDT4 implementation) **控制数据包时间戳（在UDT草案中，但不在UDT4实现中）**
* Timestamp drift correction algorithm **时间戳漂移校正算法**
* Periodic NAK reports (UDT4 disabled this draft feature in favor of timeout retransmission of unACKed packets, which consumes too much bandwidth for real-time streaming applications) **定期的 NAK 报告（UDT4禁用了此草稿功能，而希望重新传输未确认的数据包，这对于实时流应用程序会占用过多的带宽）**
* Timestamp-Based Packet Delivery (configured latency) **基于时间戳的数据包传递（可配置的延迟）**
* SRT handshake based on UDT user-defined control packet for exchanging peer configuration and implementation information to ensure seamless upgrades while evolving the protocol, and to maintain backward compatibility **基于 UDT 用户定义控制包的 SRT 握手，用于交换对等配置和实现信息，以确保在演进协议的同时进行无缝升级，并保持向后兼容性**
* Maximum output rate based on configured value or measured input rate **基于配置值或测量输入速率的，最大输出速率**
* Encryption **加密**:
    * Keying material generation from a passphrase **从密码短语密钥材料生成**
    * Exchange of keying material and decryption status using a UDT user-defined control packet (sender knows if a receiver can properly decrypt stream) **使用 UDT 用户定义的控制包交换密钥资料和解密状态（发送者可以了解接收者是否可以正确解密流）**
    * AES-CTR encryption/decryption: reassign 2 bits in the data header‘s message number to specify the key (odd/even/none) used for encryption; similar to the DVB scrambling control field **AES-CTR 加密/解密：在数据包头的消息号中重新分配 2 位，以指定用于加密的密钥（奇/偶/无）；类似于 DVB 加扰控制字段**
    * Ensure smooth key regeneration after CTR exhaustion **确保 CTR 用尽后密钥恢复流畅**

Early development of SRT was conducted using hardware encoders and decoders (Makito X series from Haivision) on internal networks, where randomly distributed packet drops were simulated using the ​*netem​ utility [1]*.

SRT 的早期开发是通过内部网络上的硬件编码器和解码器（来自 Haivision 的 Makito X 系列）进行的，其中使用 *netem 实用程序[1]* 模拟了数据包的随机丢弃。

> *[1] It was later discovered that this model is not a good simulation as packets are dropped without any network congestion. Packet loss was simulated in both directions, which means some feedback control packets were also dropped. **后来发现该模型不是很好的仿真，因为数据包被丢弃而没有任何网络拥塞。双向模拟数据包丢失，这意味着一些反馈控制数据包也被丢弃。***

In the presence of moderate packet loss the decoder experienced buffer depletion (no packets to decode). Lost packets were not retransmitted in time. The fix for that was to trigger the unacknowledged packet retransmission (the Auto Repeat of ARQ) earlier. However, this caused a surge in bandwidth usage. Retransmission upon timeout is done when packets are not acknowledged quickly enough. In UDT4, this retransmission occurs only when the loss list is empty.

在存在中等丢包的情况下，解码器会经历缓冲区耗尽（没有要解码的包）。丢失的数据包没有及时重传。 解决此问题的方法是更早触发未确认的数据包重传（ARQ 的自动重复）。但是，这导致带宽使用激增。如果没有足够快地确认数据包，则会在重传时超时。在 UDT4 中，仅当丢失列表为空时才发生此重传。

Retransmitting all the packets for which an ACK was past due (sent more than RTT ms ago) resolved the test scenario since there was no congestion and the random packet drop would affect different packets. After many retransmissions, all packets would be delivered and the receiver‘s queue unblocked. This was at the cost of huge bandwidth usage since at the time there was no sending rate control.

重新传输所有 ACK 过期的数据包（发送时间超过 RTT 毫秒）可以解决测试设想，因为没有拥塞，随机数据包丢弃会影响其他的数据包。在多次重传之后，所有数据包都将被传送，并且接收者的队列将被解除阻塞。由于开始时没有发送速率控制，因此将以占用大量带宽作为代价。
