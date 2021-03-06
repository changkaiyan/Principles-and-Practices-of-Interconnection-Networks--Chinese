# 第二章 一个简单的互联网络

在本章中，我们将研究简单互连网络的体系结构和设计，以提供全局视图。我们将研究最简单的网络：具有丢包流量控制（dropping flow control）功能的蝶形网络（butterfly network）。尽管最终的网络成本（cost）很高，但它强调了互连网络设计的许多关键方面。在后面的章节中，我们将学习如何产生更有效和实用的网络。

## 2.1 网络规格和约束

像所有工程设计问题一样，网络设计也从描述希望构建的规范(specifications)和一系列限制潜在解决方案范围的约束（constraints）开始。表2.1总结了本章示例网络的规范。实例网络的规范包括网络的大小（64个端口）和每个端口所需的带宽(bandwidth)。如表所示，峰值带宽和平均带宽相等，意味着输入以0.25 Gbyte/s的速率连续注入消息。随机流量中每个输入均以相同的概率发送到每个输出，并且期望消息大小为4到64字节。此外，服务质量和可靠性规格允许丢弃包。也就是说，并非每个数据包都需要成功传递到其目的地。丢弃数据包的能力可以简化流量控制实现。当然，一组实际的规范将需要更长和更多的规范。例如，QoS规范将指示可以丢弃什么部分的数据包以及在什么情况下。但是实例规范足以说明设计的许多要点。

参数|值
-|-
输入端口|64
输出端口|64
峰值带宽|0.25 Gbyte/s
平均带宽|0.25 Gbyte/s
消息延迟|4-64 bytes
流量模式|随机
服务质量|允许丢包
可靠性|允许丢包

表2.2说明了我们示例网络设计的约束条件，这些约束条件指定了每个包（packaging）级别的容量和成本。我们的网络由组装在电路板上的通过电线（cable）连接的芯片（chips）组成。约束指定信号的数量[1]可以跨每个级别的模块接口传递，以及每个模块的成本。对于电缆，约束条件还指定了在不减少电缆带宽的情况下可以穿越的最长距离。 [2]

[1] 信号（signals）不一定指的是引脚（pins）。例如，使用差分信号（differential signaling）非常普遍，每个信号需要两个引脚。
[2]由于带宽乘以距离平方(Bd^2)是给定类型电缆的常数，因此带宽必须降低四倍，至250 Mbits / s，以使电缆延伸两倍。



## 2.2 拓扑

为简单起见，我们的示例网络具有蝶形拓扑。从单个输入端口的角度来看，蝶形网络看起来像一棵树。（请参阅图2.1）树的每一级都包含交换节点（switching nodes），与终端节点（terminal nodes）不同，交换节点不发送或接收数据包，而仅传递数据包。同样，每个通道（channels）都是单向的，如箭头所示，从输入流向输出节点（图中从左到右）。但是，选择蝶形网络并不能完成拓扑设计的工作。我们还必须决定网络的增速（speedup），确定蝴蝶的基数，并确定拓扑如何映射到包级别。

网络的速度是网络的总输入带宽（input bandwidth）与网络的理想容量(capacity)之比。假设完美的路由和流量控制，网络在给定的流量模式（traffic pattern）下实现最大吞吐量，这就是容量的定义。增速为1的设计意味着输入的需求与网络传递流量的理想能力完全匹配。提供更高的增速会增加设计的余量，并允许实现中的非理想性。从某种意义上讲，速度类似于土木工程师对结构安全系数的概念。例如，安全系数为4的建筑物被设计为承受比其规格大4倍的应力。

对于蝴蝶网络，将网络的每个通道调整为具有与单个输入端口相同的带宽可以使增速提高到1。考虑随机流量（random traffic）下对任何特定通道的需求，即每个输入通过此通道发送的流量之和始终会产生等于输入端口带宽。在我们的简单网络中，这相当于设计带宽为0.25 Gbyte/s的通道。但是，基于在随后的设计选择中我们将选择简单性（simplicity）而不是效率（efficiency）这一事实，我们选择了8的网络增速。虽然这种增速非常大（布鲁克林大桥的设计安全系数仅为6！）。 我们将在本书中快速学习如何降低增速要求。

我们对增速的选择以及封装的约束决定了每个交换节点的输入和输出数量，称为蝶形基数（radix）。例如，图2.1中的蝶形设计的基数为2。每个交换节点都在单个芯片上实现，因此通道总数（输入和输出）乘以通道宽度在每个芯片上不能超过150个信号。要使增速达到8，我们需要8 × 0.25 = 2 Gbytes/s的网络通道带宽。这需要16个信号每个以1 Gbit / s的速度运行。允许2个额外的开销控制信号，所述信道是宽18个信号，并且我们可以适合只有150 / 18 ≈ 8在芯片上的通道。因此，我们选择了基数为4的蝶形网络，该蝶形每个交换节点具有4个输入通道和4个输出通道，总共8个通道。

为了使得每个输入端口连接到所有64个输出端口，我们的蝴蝶需要log_4^{64} = 3级或阶段(stages)的交换节点。因此，我们的网络将是基数为4的3级(radix-4, 3-stage)蝶形网络，或简称为4-ary 3-fly蝶形。该网络的完整拓扑如图2.2所示。尽管此图乍一看似乎令人生畏，但这是我们在图2.1中引入的较小蝴蝶的扩展。将输入1连接到64个输出中的每一个的通道再次形成一棵树（以粗体显示），但是现在树的度(degree)是4-我们网络的基数。

**设计拓扑的最后一步是封装。** 我们已经通过在每个芯片上放置一个交换节点来做出封装决定。通过选择满足每个芯片约束的网络基数，我们知道交换节点被封装在我们的设计约束之内。这些芯片必须安装在电路板上，为了最小化成本，我们希望在电路板上安装尽可能多的交换芯片。但是，我们必须限制不要超过进入或离开电路板的750个信号的最大引脚数。[3] 该约束由可通过板的一个边缘上路由连接器的最大信号数量所驱动-连接器密度（signals/ m）乘以连接器长度（m）。

交换节点在电路板之间的有效划分如图2.2所示。8个板中每个板的边界由虚线框表示。如图所示，通过将第一阶段的交换节点放置在4个电路板上（每个板包含4个芯片）来封装网络。接下来的2个阶段封装在4个板上，每个板上包含8个芯片，这些芯片以16端口蝶形网络连接。我们通过观察32个通道（每个通道包含18个信号）进入和离开每个电路板，来验证是否满足每个电路板的管脚约束。这使我们的总引脚分配为32 × 18 = 576个信号，这符合750个信号的约束。精明的读者会注意到，我们可以在第一级板上放置5个路由器芯片（总共40个通道或720个信号），但这并不高效，因为我们仍然需要4个第一阶段的板子。同样，我们不能在第二级板上放置10个路由器芯片，因为所需的46个通道（828个信号）将超出我们板上的引脚排列。

最后，板之间的连接是通过电缆进行的，如图2.3所示。图中的每条粗灰色线对应于一根电缆，该电缆承载从一个电路板到另一个电路板的四个18位通道。8个电路板与其中的16条电缆连接在一起：从第一阶段的每个板到第二阶段和第三阶段的每个板。将8个电路板布置在一个机箱中，这些电缆都在最大长度范围内。

退后一步，我们看到拓扑中的交换节点如何将任何输入连接到任何输出。交换的第一级在容纳其余级的4个电路板之间进行选择。交换的第二级选择4个芯片中的1个组成所选电路板上的第三级。最后，交换机的最后阶段选择所需的输出端口。当通过网络路由数据包时，我们利用了这种分而治之的结构。

[3] 实际系统中对于电路板上的信号密度有约束。


## 2.3 路由

我们的简单蝶形网络使用目标标记路由4，其中目标地址的位用于在网络的每个阶段选择输出端口。在我们的64节点网络中，目标地址是6位。交换器的每一级使用2个地址位来选择4个交换器输出中的1个，将数据包定向到其余节点的适当四分之一。例如，考虑将数据包从输入12路由到输出节点35 = 100011 2 。目的地的最高有效位（10）选择开关0的第三个输出端口。3，取包切换1 。11.然后，中间的dibit（00）选择开关1的第一个输出端口。11.从开关2 。在图8中，最低有效位（11）选择最后一个输出端口，将数据包传递到输出端口35。

请注意，开关输出选择的顺序完全独立于数据包的输入端口。例如，从节点51到输出35的路由遵循相同的选择顺序：第一级中的第三交换机端口，第二级中的第一端口以及第三级中的最后端口。因此，我们可以通过仅在每个数据包中存储目标地址来实现路由算法。

为了统一起见，我们所有的交换节点都在目标地址字段的最高有效位上运行。然后，在数据包离开节点之前，地址字段向左移动两位，丢弃刚刚使用的位，并将下一个二位暴露在最高有效位置。

例如，在节点35上，原始地址100011 2转移到001100 2 。这种约定使我们可以在网络的每个位置使用相同的交换节点，而无需进行特殊配置。它还有利于将网络扩展到更多的节点，而仅受地址字段大小的限制。

## 2.4 流量控制

我们网络的通道每个周期传输16位宽的物理数字或phits 。但是，我们已指定网络必须传送包含32到512位数据的整个数据包。因此，我们使用如图2.4所示的简单协议将phits组装为数据包。如图所示，每个数据包都包含一个头标，然后是零个或多个有效负载。标头phit表示新数据包的开始，还包含我们的路由算法使用的目标地址。有效负载phits保存数据包的实际数据，分为16位块。给定数据包的phits必须是连续的，且不得中断。但是，可以在数据包之间的通道上传输任意数量的空phits。为区分报头字和有效载荷字并表示数据包的结尾，我们在每个通道后附加一个2位类型字段。该字段将每个16位字描述为标头（H），有效载荷（P）或空（N）字。分组的长度可以是任意的，但总是由单个H字，零个或多个P字，依次为零个或多个N字组成。使用正则表达式表示法，链接上流动的零个或多个数据包可以描述为（HP ∗ N ∗ ）∗ 。
![](2_4.PNG)

既然我们已经将phits组装成数据包，那么我们就可以进行流控制的主要业务：为数据包分配资源。为简单起见，我们的蝶形网络使用下降流量控制。如果数据包到达交换机时正在使用该数据包所需的输出端口，则该数据包将被丢弃（丢弃）。流控制假定某些更高级别的端到端错误控制协议将最终重新发送丢弃的数据包。丢包是最糟糕的流控制方法之一，因为丢包率很高，而且浪费了最终丢包的信道带宽。正如我们将在第12章中看到的那样，有更好的流控制机制。对于我们目前的目的，滴流控制是理想的，因为从概念上和实现上它都非常简单。

## 2.5 路由器设计

蝶形网络中的每个交换节点都是一个路由器，能够在其输入端接收数据包，根据路由算法确定其目的地，然后将数据包转发到适当的输出。在一个非常简单的路由器中。单个路由器的框图如图2.5所示。路由器的数据路径包括四个18位输入寄存器，四个18位4：1多路复用器，四个移位器（用于移位标头phits的路由字段）和四个18位输出寄存器。数据路径由144位寄存器和大约650个门（等效于2个输入NAND）组成。

![](2_5.PNG)

Phits将在每个时钟周期到达输入寄存器，并路由到所有四个多路复用器。在每个多路复用器上，关联的分配器将检查每个phit的类型和每个head phit的下一跳字段，并相应地设置开关。接下来，来自所选输入的Phits将被路由到移位器。在分配器的控制下，移位器将所有头信息素左移两位，以丢弃当前路由字段并暴露下一个路由字段。有效载荷数量不变。

路由器的控制权完全位于与每个控制多路复用器和移位器的输出关联的四个分配器中。顾名思义，每个分配器都将一个输出端口分配给四个输入端口之一。分配器之一的示意图如图2.6所示。分配器由四个几乎相同的位片组成，每个位片分为三个部分：解码，仲裁和保持。在解码部分，对每个输入phit的高四位进行解码。每个解码器生成两个信号。如果输入i上的phit是head phit，并且route字段的高两位与输出端口号匹配，则信号请求i为true 。此信号表示输入phit要求使用输出端口来路由从此开始的数据包头撞 解码器还会生成信号有效载荷i ，如果输入i上的点是有效载荷点，则为true 。分配器的保持逻辑使用该信号在数据包的持续时间内保持信道（只要有效载荷位在所选输入端口上）。

分配器的第二阶段是四输入固定优先级[3]仲裁器。仲裁器接受四个请求信号并生成四个授权信号。如果输出端口是可用的（asindicatedbysignal果），thearbitergrantstheporttothefirst（最上面的）inputportmakingarequest.Assertinga授权signalcausesthecorresponding选择信号被断言，选择多路复用器的相应输入传递到输出。如果断言任何许可信号，则表明报头正在通过多路复用器，并且断言移位信号以使移位器移位路由字段。

分配器的最后阶段在数据包持续时间内保持输出端口到输入端口的分配。信号last i指示在最后一个周期选择了输入端口i 。如果端口在此周期中携带有效载荷，则该有效载荷是同一数据包的一部分，并且通过声明信号hold i来保持信道。这个信号使选择我被断言和果被清除，从而防止从仲裁器分配的端口到新的报头。

在本书中，我们通常会使用Verilog寄存器传输语言（RTL）模型来描述硬件。Verilog模型是模块的文本描述，描述其输入，输出和内部功能。图2.7给出了Verilog对图2.6分配器的描述。该模块的声明开始，注释后，与模块的声明，使该模块名称，ALLOC ，并宣布其八个输入和输出。接下来的五行根据尺寸和宽度声明这些输入和输出。接下来，声明内部连线和寄存器。真正的逻辑始于十个assign语句。这些描述了分配器的组合逻辑，并且完全对应于图2.6。最后，always语句定义了保持状态last [3：0]的触发器。

![](2_6.PNG)

![](2_7.PNG)

除了是描述特定硬件的便捷文本方式之外，Verilog还用作模拟输入语言和综合输入语言。因此，在以这种方式描述了我们的硬件之后，我们可以对其进行仿真以验证其是否正常运行，然后综合一个门级设计以在ASIC或FPGA上实现。供参考，图2.8中给出了整个路由器的Verilog描述。为了简洁起见，省略了对多路复用器和移位器模块的描述。
![](2_8.PNG)

## 2.6 性能分析

我们通过三种方法来判断互连网络：成本(cost)，延迟(latency)和吞吐量(throughput)。延迟和吞吐量都是性能指标：延迟是数据包穿越网络所花费的时间，吞吐量是网络可以从输入传输到输出的每秒位数。对于我们的示例网络通过丢弃流量控制，这些性能指标会受到数据包丢弃概率的严重影响。

我们的分析从网络的简单模型开始，以分析网络重新发送丢失的数据包的情况。（见图2.9。）首先，由于网络的输入和输出之间的对称性和随机流量模式，考虑来自网络的单个输入的数据包就足够了。如图2.9所示，数据包以$\lambda$ 的速率注入网络。不是将 $\lambda$ 表示为每秒的比特数，而是将其标准化为2Gbytes / s的信道带宽，因此$\lambda =1$对应于以信道允许的最大速率注入数据包。在数据包进入网络之前，它们将与通过网络重新发送的数据包合并。这两个比率 $p_0$ 的总和是注入到网络第一阶段的数据包的总速率。在第一阶段，可能会发生一些数据包冲突，并且一小部分数据包 $p_1$ 将通过而不会丢失。速率 $p_0 -p_1$ 的差表示已丢失的数据包。如果要重新发送丢弃的数据包，这些数据包将流回输入并重新注入。类似地，由于更多的冲突和丢包，第二级 $p 2$ 和第三级 $p_3$ 的输出速率将继续降低。

![](2_9.PNG)

图2.9：一个三阶段蝶形网络的简单分析模型，该网络具有丢包控制和丢包的重新注入。

这些输出速率可以通过从网络的输入处开始并向输出方向进行迭代来计算。我们的蝶形网络的单级由 $4 \times 4$ crossbar构成，对称地，开关的每个输入处的传入数据包速率相等。因此，对于网络的第 $i + 1$ 阶段，纵横开关每个端口的输入速率为 $p_i$ 。因为速率已被归一，它们也可以被解释为任何特定时钟周期。然后，数据包离开特定输出 $p_{i + 1}$ 的概率为1减去没有数据包想要该输出的概率。由于流量模式是随机的，因此每个输入将需要一个概率为 $p_i/4$ 的输出，因此，没有输入想要一个特定输出的概率就是

$$(1-\frac{p_i}{4})^4$$

因此，输出速率$p_{i+ 1}$ 在阶段$i + 1$是

$$p_{i+1}=1-(1-\frac{p_i}{4})^4$$

应用公式2.2 $n = 3$次，对于网络的每个阶段一次，并暂时忽略重发数据包（$p_0 =\lambda$ ），我们计算出输入占空比为 $\lambda = 0$ 。125（对应于加速8）时，三个开关级输出的占空比分别为0.119、0.114和0.109。也就是说，在网络输入端提供的链路容量为0.125的流量(offered traffic)中，网络的可接受流量(accepted traffic)或吞吐量(throughput)仅为0.109。剩余的由于网络冲突而丢弃了0.016（占数据包的12.6％）。一旦丢弃的数据包重新注入到网络中，就会发生有趣的动态变化。重新发送丢弃的数据包可提高对网络 $p_0$ 的有效输入速率。反过来，这会增加丢弃的数据包的数量，依此类推。如果此反馈环路稳定，使得网络能够支持所产生的输入速率（$p_0 \le 1$ ），则业务量注入网络将等于量喷射（$p_3 =\lambda $ ）。

图2.10绘制了示例网络中提供的流量和吞吐量之间的关系。将两个轴均归一化为网络的理想容量。我们看到，在非常低的负载下，几乎所有流量都通过网络，吞吐量等于所提供的流量。但是，随着所提供流量的增加，快速丢弃成为主要因素，如果不重新发送数据包，网络的吞吐量将大大低于所提供流量。最终吞吐量达到饱和，达到43.2％的渐近线。无论向网络提供多少流量，无论是否重新发送数据包，我们都无法实现大于通道容量43.2％的吞吐量。请注意，我们可以以低至2.5的加速比运行该网络，从而有效地将最大注入速率限制为0.4。但是，我们将看到，最初选择的加速比为8将在延迟方面有好处。同样，我们的网络实现的容量不足其一半的事实是在实践中几乎从未使用过丢弃流量控制的主要原因。在第12章中，我们将了解如何构建流控制机制，使我们能够在不超过通道容量90％的情况下运行网络。

![](2_10.PNG)

图2.10： 吞吐量是我们简单网络提供的流量（注入率）的函数。 显示了丢包和不重新注入（丢弃）时的吞吐量以及重新注入（重新注入）时的吞吐量。

在整个吞吐量讨论中，我们一直将流量和吞吐量表示为信道容量的一部分。我们遵循以下约定：

之所以选择这本书，是因为它比以每秒位数表示这些数字提供了更多的见识。例如，通过声明网络以24％的容量饱和而不是声明网络以480Mbytes / s的饱和度，我们对流量控制方法的相对性能有了更深入的了解。在下面的讨论中，我们以类似的方式标准化延迟。

数据包的等待时间由数据包成功穿越网络之前必须重新传输的次数决定。在网络中没有其他数据包的情况下，数据包的报头以6个时钟周期穿越网络。数据包通过三个阶段中每个阶段的两个触发器寄存器计时。为了清楚起见，我们将这6个周期的延迟称为1.0的相对延迟，并表示相对于该数字在更高负载下的网络延迟。在本节中，我们仅关注头延迟。数据包的总等待时间还包括一个序列化术语，该术语等于数据包的长度除以通道的带宽L / b ，它反映了数据包尾部追赶头部所需的时间。

随着负载的增加，一部分数据包$p_D$ 被丢弃，必须重新发送。

$$p_D=\frac{p_0-p_3}{p_0}$$

这些重发数据包中的相同部分再次被丢弃，必须再次重发，依此类推。我们通过对每种情况的延迟加总该情况的概率进行加权，来计算数据包的平均延迟。不切实际地，假设源立即发现数据包被丢弃并在6个周期后重新传输，则延迟为丢弃i次的数据包为i + 1。恰好丢弃i次的数据包的概率为 $P_D^i(1 -P_D )$。总结加权延迟，我们将平均延迟计算为

$$T=\sum_{i=0}^{\inf}(i+1)P_D^i(1-P_D)=\frac{1}{1-P_D}=\frac{p_0}{p_3}$$

该延迟在图2.11中作为提供的流量的函数进行绘制。在网络上没有负载的情况下，延迟始于统一（6个时钟周期）。随着吞吐量的增加，某些数据包将被丢弃，平均延迟时间将增加，吞吐率将增加约0.39。最终，网络以0.43的吞吐量达到饱和。此时，在任何延迟下都无法实现更大的吞吐量。

对于适度的负载，公式2.3提供了合理的延迟模型，但是随着网络趋于饱和，人们对可以立即重发数据包的假设提出了质疑。这是因为重新发送的数据包和新注入的数据包同时被引入网络的可能性增加。在实际的实现中，一个数据包将不得不等待另一个数据包继续前进，从而导致额外的排队等待时间。图2.11还显示了另一个包含排队时间模型的等待时间曲线，该曲线的形状在互连网络中更为典型，随着吞吐量接近饱和，等待时间逐渐增长到无穷大。两条曲线都与练习2.9中的模拟进行了比较。

注意公式2.3和图2.11给出数据包的平均延迟很重要。对于许多应用程序，我们不仅对平均延迟感兴趣，而且对延迟的概率分布也很感兴趣。特别是，我们可能会担心最坏情况下的延迟或延迟的变化（有时称为抖动）。例如，在视频播放系统中，在播放之前保存数据包所需的缓冲区大小是由抖动而不是平均延迟确定的。对于我们的示例网络，相对延迟为i的数据包被接收的概率为


![](https://latex.codecogs.com/gif.latex?P(T=i)=P_D^{(i-1)}(1-P_D))

![](2_11.PNG)

延迟的这种指数分布会导致无限的最大延迟，从而导致无限的抖动。更实际地，我们可以用给定分数（例如99％）的数据包实现的延迟范围来表示抖动。

上面讨论的性能指标都是针对统一随机流量的。对于蝶形网络，这是最好的情况。正如我们将看到的，对于某些流量模式，例如位反转，[4]网络的性能远远不如此处所述。蝶形网络对不良流量模式的敏感度很大程度上是由于从网络的每个输入到每个输出只有一条路径这一事实。我们将看到，具有路径分集的网络在困难的负载下的性能要好得多。

