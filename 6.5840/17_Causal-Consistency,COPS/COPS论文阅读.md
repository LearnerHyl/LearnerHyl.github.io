---
layout: default
title: COPS/Causal-Consistency
description: 论文阅读
---


# Don't Settle for Eventual:Scalable Causal Consistency for Wide-Area Storage with COPS

# Abstract

支持复杂在线应用（如社交网络）的地理复制、分布式数据存储必须提供一个“始终在线”的体验，其中操作总是以低延迟完成。当今的系统通常为实现这些目标而牺牲强一致性，向客户端暴露不一致性并增加复杂的应用逻辑。在这篇论文中，我们确定并定义了一种一致性模型——因果一致性与收敛冲突处理（causal+），这是在这些约束下实现（分布式、始终在线、低延迟操作）的最强大的一致性模型。

我们介绍了 COPS 的设计和实现，这是一种在广域网范围内提供此因果一致性模型的键值存储系统。<u>COPS 的一个关键贡献是其可扩展性，可以在整个集群中存储的键之间强制执行因果依赖关系，而不是像以前的系统那样使用单个服务器。COPS 的核心方法是追踪并在暴露写入操作之前显式检查本地集群中键之间的因果依赖关系是否满足。</u>

此外，在 COPS-GT 中，我们引入了 get 事务，以便在不锁定或阻塞的情况下获得多个键的一致视图。我们的评估表明，COPS 完成操作的时间不到一毫秒，在每个集群使用一个服务器时，其性能与先前的系统相似，并且在增加每个集群服务器数量时扩展性良好。评估还表明，对于常见工作负载，COPS-GT 在延迟、吞吐量和扩展性方面与 COPS 类似。 

在这篇论文中，我们讨论了 COPS 这个支持因果一致性的分布式键值存储系统的设计与实现，以解决复杂在线应用中的一致性与性能问题。COPS 利用跟踪因果依赖关系和显式检查来提高系统的可扩展性和性能。论文还介绍了 COPS-GT，用于在保持一致性的同时获得多个键的视图。评估结果表明，COPS 和 COPS-GT 是实现低延迟、高吞吐量和良好可扩展性的有效系统。

# 1.Introduction

分布式数据存储是现代互联网服务的基本构建模块。理想情况下，这些数据存储应具有强一致性，始终可用于读取和写入，并能在网络分区期间继续运行。然而，CAP 定理证明了不可能创建一个同时实现这三个属性的系统[13, 23]。因此，现代网络服务在很大程度上选择以牺牲强一致性为代价，支持可用性和分区容忍性[16, 20, 30]。鉴于这种选择还使这些系统能够为客户端操作提供低延迟和高可扩展性，这也许并不奇怪。此外，许多早期大规模互联网服务（通常关注网络搜索领域），并没有看到对更强一致性的迫切需求，尽管随着互动服务（如社交网络应用）的兴起，这种状况正在发生变化[46]。<u>我们将具有这四个属性（可用性、低延迟、分区容忍性和高可扩展性）的系统称为 ALPS 系统。</u>

由于 ALPS 系统必须牺牲强一致性（即线性一致性），我们寻求在这些约束下可以实现的最强一致性模型。更强的一致性是可取的，因为它使系统的编程理解更容易。在本文中，我们考虑具有收敛冲突处理的因果一致性模型(`causal consistency with convergent conflict`)，我们将其称为因果+一致性(`causal+`)。许多先前被认为实现较弱因果一致性的系统[10, 41]实际上实现了更有用的因果+一致性，尽管没有一种方法是以可扩展的方式实现的。 

> causal+ consistency是causal consistency的强化版：有了收敛冲突处理的能力。

因果+一致性的因果组件确保数据存储遵循操作之间的因果依赖关系[31]。考虑这样一个场景：用户将图片上传到网站，图片被保存，然后将该图片的引用添加到用户的相册中。引用“依赖于”图片的保存。<u>在因果+一致性下，这些依赖关系始终得到满足。与那些较弱保证的系统（如最终一致性）不同，程序员无需处理可以获得图片引用但无法获得图片本身的情况。</u>

<u>因果+一致性的收敛冲突处理组件确保副本永远不会永久分离，并且对同一个键的冲突更新在所有站点上都以相同的方式处理。</u>与因果一致性结合时，此属性确保客户端仅看到键的逐渐更新的版本。相比之下，最终一致性系统可能会突显无序的版本。通过结合因果一致性和收敛冲突处理，因果+一致性确保客户端看到一个因果正确的、无冲突的、始终逐渐更新的数据存储。 

我们的 COPS 系统（`Clusters of Order-Preserving Servers`)提供因果+一致性，并旨在支持由少量的大规模数据中心托管的复杂在线应用，每个数据中心都由前端服务器（COPS 的客户端）和后端键值数据存储组成。<u>COPS 以线性化方式在本地数据中心执行所有 put 和 get 操作，然后在后台以因果+一致性顺序将数据复制到其他数据中心。</u>因此，COPS 结合了因果一致性和收敛冲突处理，为复杂在线应用提供了因果正确、无冲突并始终能正常运行的数据存储解决方案。

我们详细介绍了两个版本的 COPS 系统。常规版本的 COPS 为数据存储中的个别项提供可扩展的因果+一致性，即使它们的因果依赖关系分布在本地数据中心的许多不同机器上。这些一致性特性成本较低：COPS 的性能和开销与之前的系统（如基于日志交换的系统[10, 41]）相似，即使在提供更大规模的可扩展性时也是如此。

我们还详细介绍了一种扩展版的系统，COPS-GT，它还提供了 get 事务，使客户端可以获得多个键的一致视图。即使在完全线性化的系统中，也需要 get 事务来获得多个键的一致视图。我们的 get 事务不需要锁定，非阻塞，最多需要两轮并行的数据中心内请求。据我们所知，COPS-GT 是第一个实现非阻塞可扩展 get 事务的 ALPS 系统。 <u>这些事务确实带来了一定的代价：与 COPS 的常规版本相比，COPS-GT 在某些工作负载（例如写入密集型）方面的效率较低，对长时间的网络分区和数据中心故障的鲁棒性也较差。</u>

对 ALPS 系统的可扩展性要求使 COPS 与先前的因果+一致性系统之间产生了最大的区别。之前的系统要求所有数据都适合放在一台机器上[2, 12, 41]，或者所有可能一起访问的数据都能完整的放在一台机器上[10]。相比之下，COPS 中存储的数据可以分布在任意大小的数据中心中，依赖关系（或 get 事务）可以跨数据中心内的多台服务器。据我们所知，COPS 是第一个实现因果+（从而实现因果）一致性的可扩展系统。

本文的贡献包括： 

- 我们明确指出了分布式数据存储的四个重要属性，并用它们定义 ALPS 系统。 
- 我们命名并正式定义了因果+一致性。 
- 我们展示了 COPS 的设计和实现，这是一个可扩展的系统，高效地实现了因果+一致性模型。
- 我们在 COPS-GT 中提出了一种非阻塞、无锁的 get 事务算法，最多通过两轮本地操作为客户端提供多个键的一致视图。 
- 我们通过评估显示，COPS 在所有测试工作负载下具有低延迟、高吞吐量以及很好的扩展性；并且 COPS-GT 对于常见工作负载具有类似的性能表现。

# 2. ALPS systems and trade-offs

我们关注的是能够支持如今许多最大互联网服务的基础设施。与传统的分布式存储系统（以小范围的局域操作为重点）形成鲜明对比的是，这些服务通常以广域部署为特点，跨足数个至几十个数据中心，如图1所示。每个数据中心包含一组应用级客户端，以及这些客户端进行读写操作的后端数据存储。<u>对于许多应用程序（以及本文考虑的场景），在一个数据中心中写入的数据将被复制到其他数据中心。</u>

![image-20230820102107407](https://s2.loli.net/2023/08/20/ZRScmdhexJi7GjM.png)

> 图 1：现代 Web 服务的通用架构。多个地理分布式数据中心都有应用程序客户端，这些客户端从分布在所有数据中心的数据存储中读取和写入状态。这种架构允许在不同地理位置的用户能够访问相同的数据和服务，从而支持全球范围内的高可用性和低延迟访问。多个数据中心之间的数据同步和复制也有助于提高数据的冗余性，确保在面临数据丢失或硬件故障时，服务仍能正常运行。

通常，这些客户端实际上是代表远程浏览器运行代码的 Web 服务器。尽管本文从应用客户端（即 Web 服务器）的角度考虑一致性，如果浏览器通过单个数据中心访问服务，如我们所期望的，那么它将享有类似的一致性保证。

这样的分布式存储系统具有多个，有时是相互竞争的目标：提供“始终在线”的用户体验[16]的可用性、低延迟和分区容忍性；可伸缩性，以适应不断增长的负载和存储需求；以及足够强大的一致性模型，简化编程并为用户提供他们期望的系统行为。稍微深入地说，理想的属性包括：

1. 可用性（`availability`）。发给数据存储的所有操作都能成功完成。没有操作会无限期阻塞或返回表示数据不可用的错误。
2. 低延迟（`Low Latency`）。客户端操作“快速”完成。商业服务级目标建议几毫秒的平均性能和几十毫秒或数百毫秒的最差性能（即99.9分位数）[16]。
3. 分区容忍性（`Partition Tolerance`）。数据存储在网络分区下继续运行，例如将亚洲数据中心与美国数据中心分开的分区。
4. 高可扩展性（`High Scalability`）。数据存储线性扩展。将 N 个资源添加到系统中会使整体吞吐量和存储容量增加 O(N)。
5. 更强的一致性（`Stronger Consistency`）。理想的数据存储将提供线性化（有时被非正式地称为强一致性），它规定操作似乎在操作调用和完成之间的单个瞬间在整个系统中生效[26]。在提供线性化的数据存储中，只要客户端在一个数据中心完成对一个对象的写操作，所有其他数据中心对该对象的读操作就会反映其新写入的状态。线性化简化了编程——分布式系统提供了一个一致的镜像——用户体验到他们期望的存储行为。较弱的，最终一致性模型（在许多大型分布式系统中很常见）直观性较差：后续读取可能不仅无法反映最新值，而且跨多个对象的读取可能反映出旧值和新值的不一致混合。

<u>CAP 定理证明，具有可用性和分区容忍性的共享数据系统无法实现线性化[13,23]。</u>低延迟（定义为小于副本之间最大广域延迟的延迟）也被证明与线性化[34]和顺序一致性[8]不兼容。为了在 ALPS 系统的需求和可编程性之间取得平衡，我们在下一节中定义了一种中间一致性模型。

# 3. Causal+ Consistency

为了定义具有收敛冲突处理的因果一致性（因果+一致性），我们首先描述它操作的抽象模型。我们将考虑限制在键值数据存储中，有两个基本操作：`put(key, val)` 和 `get(key)=val`。这些相当于共享内存系统中的写入和读取操作。值存储并从逻辑副本中检索，每个副本都承载整个键空间。在我们的 COPS 系统中，单个逻辑副本对应于一个完整的本地节点集群。

我们模型中的一个重要概念是操作之间潜在因果关系的概念[2, 31]。三个规则定义了潜在因果关系，用 "->" 表示：

1. 执行线程（`Execution Thread`）。如果 a 和 b 是一个执行线程中的两个操作，那么如果操作 a 在操作 b 之前发生，则 a->b。
2. 获取自（`Gets From`）。如果 a 是一个 put 操作，b 是一个返回 a 写入的值的 get 操作，那么 a->b。
3. 传递性（`Transitivity`）。对于操作 a，b 和 c，如果 a->b 和 b->c，那么 a->c。

这些规则建立了同一执行线程内操作之间的潜在因果关系，以及通过数据存储进行交互的执行线程之间的操作潜在因果关系。<u>我们的模型（像很多其他模型一样）不允许线程直接通信，而要求所有通信都通过数据存储进行。</u>

![image-20230820105240997](https://s2.loli.net/2023/08/20/DHnLXjzaxh7URuE.png)

图 2 中的示例执行演示了所有三个规则。执行线程规则给出了 get(y)=2->put(x,4)；gets from 规则给出了 put(y,2)->get(y)=2；传递性规则给出了 put(y,2)->put(x,4)。尽管有些操作在实际时间里紧随 put(x,3) 发生，但没有其他操作依赖于它，因为没有其他操作读取它写入的值，也没有在相同的执行线程中紧随它进行。

## 3.1 Definition

### 3.1.1 Causal Consistency

我们将因果+一致性定义为两个属性的组合：因果一致性和收敛冲突处理。我们在这里简要介绍直观定义，并附录A中给出正式定义。

因果一致性要求副本上的get操作返回的值与->(因果性)[2]所定义的顺序一致。换句话说，执行写入值的操作看起来必须在因果上执行之前的所有操作之后发生。例如，在图2中，必须表现为 put(y,2) 在 put(x,4)之前发生，而 put(x,4) 在 put(z,5)之前发生。如果客户端2先看到get(x)=4，后看到get(x)=1，那么因果一致性将受到破坏。

<u>因果一致性不对并发操作进行排序。如果 a-/>b 和 b-/>a，则 a 和 b 是并发的。</u>通常，这允许实现具有更高效率：两个无关的put操作可以按任意顺序复制，避免在它们之间需要串行化点。然而，如果 a 和 b 都是对相同键的put操作，那么它们是冲突的。

<u>冲突有两个原因令人不悦。首先，由于冲突没有被因果一致性排序，因此冲突允许副本永远发散</u>[2]。例如，如果a是put(x,1)，b是put(x,2)，那么因果一致性允许一个副本永远为x返回1，另一个副本永远为x返回2。其次，冲突可能代表需要特殊处理的异常情况。例如，在购物车应用程序中，如果两个人同时登录同一个帐户并将商品添加到购物车中，理想情况下最终希望购物车中包含两件商品。

### 3.1.2 Convergent conflict

收敛冲突处理要求所有副本中的所有冲突put以相同的方式处理，使用处理函数 h。此处理函数h必须具有结合性和交换性，以便副本按照它们接收到的顺序处理冲突写入，并且这些处理的结果将收敛（例如，一个副本的h(a, h(b, c))和另一个副本的h(c, h(b, a))一致）。

处理冲突写入的一种常见方法是采用最后写入者获胜规则（也称为Thomas的写入规则[50]），该规则声明其中一个冲突写入发生在更晚的时间，并覆盖“较早”的写入。另一种常见的处理冲突写入的方法是将它们标记为冲突，并要求通过其他方式进行解决，例如，通过像Coda[28]那样的直接用户干预，或者通过像Bayou[41]和Dynamo[16]那样的编程过程。

<u>所有可能的收敛冲突处理形式都避免了第一个问题，即冲突更新可能会不断发散，因为它们确保副本在交换操作后达到相同的结果。另一方面，关于冲突的第二个问题——应用程序可能希望对冲突进行特殊处理，只有通过使用更显式的冲突解决程序才能避免。</u>

这些显式程序为应用程序提供了更大的灵活性，但需要额外的程序员复杂性和/或性能开销。尽管COPS可以配置为显式检测冲突更新并应用某些应用程序定义的解决方案，但COPS的默认版本使用最后写入者获胜规则。

## 3.2 Causal+ vs. Other Consistency Models

这块可以看自己写的博客，关于一致性的一些说明，说的比这里更通俗一些。[共识、线性一致性、顺序一致性、最终一致性、强一致性概念区分 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/618127949)

分布式系统文献定义了几种流行的一致性模型。按强度递减，它们包括：

- 线性化（或强一致性）[26]，维持全局实时排序（除了满足顺序一致性，内部操作还符合全局时钟的顺序）；
- 顺序一致性[32]，确保至少有一个全局顺序（有点类似于数据库中的serializability的隔离级别）；
- 因果一致性[2]，保证依赖操作之间的部分顺序（不保证并发的顺序）；
- FIFO（PRAM）一致性[34]，仅保留执行线程的部分顺序，而不是线程之间的顺序（顺序一致的弱化，只专注于某个线程本身，而不关注线程之间的操作顺序正确与否）；
- 每键顺序一致性(`Per-Key Sequential`)[15]，确保对于每个单独的键，所有操作都具有全局顺序，相对于FIFO，进一步的放宽要求，现在是确保对某一个键的执行顺序，不一定保证某个执行线程内部的顺序。
- 最终一致性，一个当前使用的“包罗万象”的术语，表示最终收敛于某种类型的协议。

![image-20230821092820030](https://s2.loli.net/2023/08/21/bBUnxS7zPlWL8Js.png)

> 图3： 此图表示一致性模型的光谱，左侧的一个模型具有较强的一致性。加粗的模型已被证明与ALPS系统不兼容。

我们引入的因果+一致性位于顺序一致性和因果一致性之间，如图3所示。它比顺序一致性弱，但顺序一致性在ALPS系统中被证明是不可实现的。然而，它比因果一致性和每键顺序一致性更强，并且对ALPS系统是可实现的。Mahajan等人[35]同时定义了类似的因果一致性增强措施；具体细节请参阅第7节。

为了说明因果+模型的实用性，考虑两个例子。首先，让爱丽丝试着和鲍勃分享一张照片。爱丽丝上传照片然后将其添加到她的相册中。然后鲍勃查看爱丽丝的相册，期望看到她的照片。在因果一致性和因果+一致性下，如果相册有指向照片的引用，那么鲍勃必须能够查看照片。<u>在每键顺序一致性和最终一致性下，相册可能有指向尚未写入的照片的引用。</u>

> 在每键顺序一致性下，对于每个键，它确实确保所有写入操作都遵循全局顺序，但不能确保相册键的写入操作与照片键的对应操作之间的顺序关系。
>
> 这意味着在某些情况下，相册可能已写入引用照片的内容，但照片本身尚未完成写入。当Bob尝试访问这个引用时，他只能看到引用而不是实际的照片。虽然每键顺序一致性为单个键提供了顺序保证，但跨多个相关键的操作时不能提供可靠的关联。因此，在每键顺序一致性下，Bob可能在相册中找到未写入的照片引用。
>
> 在这个例子中，每个键代表不同的数据，一个键关联着相册中的照片引用，另一个键用于存储实际的照片数据。由于每键顺序一致性只能确保单个键内的操作顺序，因此跨多个键的操作（例如相册引用和照片本身）之间的顺序不能得到保证。这就可能导致引用一个尚未写入的照片的情况。

第二个例子，考虑卡罗尔和丹尼尔都更新一个活动的开始时间。原来的时间设定为晚上9点，卡罗尔将其改为晚上8点，丹尼尔同时将其改为晚上10点。普通的因果一致性允许两个不同副本在收到两个put操作之后永远返回不同的时间。因果+一致性要求副本以收敛的方式处理此冲突。如果使用最后写入者获胜策略，则可能是丹尼尔的10点或卡罗尔的8点获胜。如果使用更明确的冲突解决策略，则可以将键标记为冲突状态，并且未来对其进行的操作可以返回8点和10点以及解决冲突的说明。

如果数据存储是顺序一致性的或可线性化的，那么在这些更强模型中，也可能对键进行两个同时更新。然而，在这些强模型中，可以实现互斥算法（如原始顺序一致性论文[32]中提到的兰波特算法），这些算法可用于避免完全产生冲突。

## 3.3 Causal+ in COPS

COPS系统中使用了两个抽象概念——版本（versions）和依赖项（dependencies），帮助我们理解因果+一致性。我们将一个键具有的不同值称为该键的版本（versions of a key），表示为key~version~。在COPS中，版本被分配的方式确保了如果xi->yj，那么i < j。<u>一旦COPS中的副本返回某键的第i个版本xi，因果+一致性确保它接下来只返回该版本或在因果流程中较晚出现的版本（注意，处理冲突的过程在因果更新冲突解决之后才发生）^1^。</u>因此，COPS中的每个副本总是返回键的非递减版本。我们将这称为因果+的一致性进步特性。

> 要证明这一点，可以通过反证法考虑以下场景：假设一个副本首先返回xi，然后返回xk ，其中i ≠ k且xi-/>xk。因果一致性确保如果在返回xi之后返回xk，那么xk-/>xi，因此xi和xk存在冲突。但是，如果xi和xk发生冲突，那么收敛冲突处理确保一旦它们同时出现在一个副本上，它们的处理h(xi, xk)，该处理在因果上晚于二者，将替代xi或xk返回，这就与我们的假设相矛盾。 即会返回一个函数handler(xi,xk)，返回之后才会进行处理。
>
> 因此，可以得出结论，在因果一致性下，一旦副本首先返回xi，接下来返回的只可能是xi的因果依次版本，例如经过冲突处理的版本。在这种情况下，副本不会返回两个同时发生的且产生冲突的版本。

因果一致性要求所有在给定操作之前发生的操作，看起来都在给定操作之前生效。换句话说，如果xi->yj，则必须在yj之前写入xi。我们称这些先于的值为依赖项（dependencies）。更正式地说，当且仅当put(xi)->put(yj)时，我们称yj依赖于xi。这些依赖项实际上是写操作因果顺序的反向过程。**COPS在复制过程中通过只在写入其所有依赖项之后才写入一个版本来提供因果+一致性。**

## 3.4 Scalable Causality

据我们所知，本文是首次命名并正式定义因果+一致性的。有趣的是，一些之前被认为实现因果一致性的系统[10,41]实际上实现了更强的因果+一致性保证。

然而，这些系统并没有设计和提供可扩展的因果（或因果+）一致性，因为它们都使用了一种日志序列化和交换的方式。逻辑副本上的所有操作都按序列化顺序写入单一日志，通常用版本向量[40]进行标记。不同的副本然后交换这些日志，使用版本向量来建立可能的因果关系并检测不同副本上的操作之间的并发性。

基于日志交换的序列化限制了副本的可扩展性，因为它依赖于每个副本中的单个序列化点来建立顺序。因此，要么限制键之间的因果依赖为可以存储在一个节点上的键集[10,15,30,41]，要么单个节点（或复制的状态机）必须为整个集群的所有操作提供提交顺序和日志。

正如下面所示，COPS通过采取不同的方法实现了可扩展性。每个数据中心的节点负责处理键空间的不同分区，但系统可以跟踪和执行存储在不同节点上的键之间的依赖关系。COPS在与每个键版本相关的元数据中明确编码依赖关系。当键在远程复制时，接收数据中心在提交传入的版本之前进行依赖性检查。

# 4. SYSTEM DESIGN OF COPS

COPS是一个实现因果+一致性并具备所需ALPS属性的分布式存储系统。该系统有两个不同的版本。第一个版本我们简称为COPS，它提供了一个因果+一致的数据存储。第二个版本名为COPS-GT，它在COPS的基础上增加了对get事务的支持，从而提供了更多功能。通过get事务，客户端可以请求一组键，并从数据存储中获得一致性快照的相应值。由于实现get事务的一致性属性需要额外的元数据，因此给定部署必须专门作为COPS或COPS-GT运行。

这意味着中途切换版本的成本会很大，因为需要增加或删除大量的逻辑。

## 4.0 local linearizability and wide casual+ consistency

是的，论文中提到的因果+一致性涉及所有数据中心，而这里提到的本地线性一致性是指单个地区（数据中心）内的一致性。两者之间的关系是：本地线性一致性是确保跨所有数据中心的因果+一致性所必需的。

在单个数据中心内，本地线性一致性确保读取操作遵循写入操作的顺序，即对于任意两个操作，它们按照发生顺序执行，并且每个读取操作要么看到最近的写入操作，要么看到一个稍晚的写入操作。这意味着在同一个数据中心，读操作总是看到一个单一的、更新的数据视图。

跨数据中心的因果+一致性则意味着一个操作发生在数据中心A，在因果关系上不远的一个操作在数据中心B也会看到这个操作。这样系统能更好地处理跨数据中心的并发操作及其冲突。

以下是两者之间存在的冲突类型:

1. 数据中心之间的延迟和通信问题：由于网络延迟和不一致的带宽，因果+一致性在跨数据中心的操作中可能会受到挑战，导致广域网络中的通信瓶颈和冲突。
2. 同时写入相同键的冲突：在多个数据中的不同客户端在同一时间尝试修改同一个键时，可能会发生冲突。在这种情况下，需要利用因果关系和因果+一致性规则来解决此类冲突。

请注意，在因果+一致性和本地线性一致性之间，不会存在直接的冲突。它们一起协同工作，以在单个数据中心内提供严格的一致性保证，并在分布式环境中保持因果关系，允许跨越各数据中心更好地处理冲突和并发操作。

## 4.1 Overview of COPS(local linearizability and wide casual+ consistency)

![image-20230821101748624](https://s2.loli.net/2023/08/21/pSPYRmsVzqjDn65.png)

图4展示了COPS架构。客户端库向其客户端提供put/get接口，并确保操作正确标记为因果依赖关系。键值存储在集群之间复制数据，确保仅在满足其依赖关系后才在本地集群中提交写操作。在COPS-GT中，还会存储每个键的多个版本和依赖元数据。

COPS架构的主要组件包括：

1. 客户端库：负责对外暴露put和get接口，确保操作被正确地标记为因果依赖关系。客户端库使应用程序能够通过简单的接口与COPS系统进行交互。
2. 键值存储：它在集群之间复制数据，以确保强因果一致性。写操作仅在满足相关依赖项后才能在本地集群中提交。这最大程度地减少了因为写操作冲突导致的不一致。
3. COPS-GT：在COPS基础上增加了对获取（get）事务的支持。COPS-GT为每个键存储多个版本以及相关的依赖关系元数据。这使得客户端能够在保持一致性的同时获取值的不同版本，并且帮助解决冲突和并发问题。

COPS架构总体上通过客户端库、键值存储和（对于COPS-GT）多版本存储，确保了因果一致性和跨数据中心的高可用性。

COPS是一个键值存储系统，设计用于跨少数几个数据中心运行，如图4所示。每个数据中心都有一个本地COPS集群，包含存储数据的完整副本^2^。<u>COPS的客户端是一个使用COPS客户端库直接调用COPS键值存储的应用程序。客户端仅与同一数据中心内运行的本地COPS集群通信。</u>

> 完全复制的假设确实简化了我们的介绍。然而，可以想象只复制总数据存储的部分内容的集群，对于其他部分，这种集群可能会根据配置规则牺牲低延迟。
>
> 通过部分复制，我们可以根据不同数据的重要性和访问频率来配置集群。在这种情况下，热点数据和关键数据可能会在多个集群中完全复制，而对于不经常访问的数据，可以采用部分复制方式。这有助于降低存储和网络带宽开销。然而，实现部分复制也有其挑战，例如在访问未复制的数据时处理延迟、配置复杂性以及在数据存储规模和集群之间平衡负载。

每个本地COPS集群都设置为线性一致性（强一致性）的键值存储[5, 48]。可以通过将键空间划分为N个线性一致性分区（每个分区可以位于单个节点或单个节点链上）并让客户端独立访问每个分区来实现线性可扩展的系统。线性一致性的组合性[26]确保整个系统仍保持线性一致性。**由于我们期望在一个集群内部的延迟非常低且没有分区——尤其是在现代数据中心网络中支持冗余路径的趋势[3, 24]，因此本地线性一致性是可以接受的，而跨广域网络则不同。**另一方面，COPS集群之间的复制是异步进行的，以确保客户端操作的低延迟和面对外部分区时的可用性。

`System Components`：COPS系统主要由两个软件组件组成：

1. 键值存储（`Key-value store`）：COPS系统的基本构建模块是提供线性化操作（强一致性）的标准键值存储。COPS对标准键值存储进行了两方面的扩展，COPS-GT还添加了第三个扩展。

   - 每个键值对都有关联的元数据。在COPS中，这些元数据是一个版本号。在COPS-GT中，它既包含一个版本号，也包含一系列依赖项（其他键及其相应的版本）。

   - 键值存储在其键值接口中导出了另外三个操作：按版本获取（`get by version`）、在...之后执行写入（`put after`）和依赖检查（`dep check`），这些操作描述如下。这些操作使COPS客户端库和支持因果+一致性的异步复制过程以及支持`get`事务成为可能。

   - 对于COPS-GT，系统保留了旧版本的键值对，而不仅仅是最近的put操作的写入，以确保它可以提供get事务。在第4.3节中将进一步讨论保留旧版本的细节。

2. 客户端库（`Client library`）：客户端库向应用程序提供了两个主要操作：通过`get`（在COPS中）或`get trans`（在COPS-GT中）进行读取操作，通过put进行写入操作^3^。客户端库还通过客户端库API中的context参数维护有关客户端当前依赖关系的状态。

   > 本文使用不同的固定宽度字体用于面向客户端的API调用(例如get)和数据存储API调用(例如按版本get)。正常字体函数是client用的，加粗的是data store API。

`Goals`：COPS设计的目标是在资源和性能开销方面，提供类似于现有最终一致性系统的因果+一致性。因此，COPS和COPS-GT必须：

1. 最小化一致性保持复制的开销（`Minimize overhead of consistency-preserving replication`）：COPS必须确保值以因果+一致的方式在集群之间进行复制。然而，简单的实现可能需要检查值的所有依赖项。通过利用因果依赖关系固有的图结构，我们提出了一种只需要进行少量检查的机制。（DAG-有向无环图）
2. （COPS-GT）最小化空间需求（`Minimize space requirements.`）：COPS-GT存储每个键的（可能的）多个版本以及它们关联的依赖元数据。COPS-GT通过积极的垃圾回收来修剪旧状态（见第5.1节）。
3. （COPS-GT）确保get trans操作迅速完成（`Ensure fast get_trans operations.`）：COPS-GT中的get事务确保返回的值集具有因果+一致性（满足所有依赖）。简单的算法可能会阻塞和/或需要无限数量的get轮次才能完成。这两种情况都与ALPS系统的可用性和低延迟目标不兼容；我们提供了一个get trans算法，最多在两轮本地get by version操作中完成。

为了实现这些目标，COPS和COPS-GT在保持一致性和性能开销方面进行了优化，以适应分布式存储系统的需求。这使得COPS和COPS-GT成为建立高可用性和低延迟应用程序的理想选择。

## 4.2 The COPS Key-Value Store

与传统的〈键，值〉`〈key, val〉`-元组存储不同，COPS需要跟踪已写入值的版本以及在COPS-GT情况下的依赖项。在COPS中，系统为每个键存储最新的版本号和值。在COPS-GT中，系统将每个键映射到一个版本条目列表，每个版本条目包括〈版本，值，依赖〉`〈version,value, deps〉`。<u>deps字段是版本的零个或多个依赖项的列表；每个依赖项都是一个`〈key, val〉`对。</u>结合上面的图4,会更好理解一些。

每个COPS集群都维护其键值存储的副本。为了可扩展性，我们的实现使用一致性哈希[27]将键空间在集群的节点间进行划分，尽管使用其他技术（例如，基于目录的方法[6，21]）也是可能的。为了容错，每个键在少量节点上使用链式复制[5，48，51]进行复制。在某个集群内部的中的节点上执行的gets和puts操作具有线性化特征（强一致性）。操作在本地集群中执行后立即返回到客户端库；集群之间的操作是异步执行的。

COPS中的每个存储的键在每个集群中都有一个主节点。我们将在所有集群中对于某个键的一组主节点称为该键的等效节点。在实践中，COPS的一致性哈希为每个节点分配了一些不同的键范围的责任。<u>键范围在不同数据中心的大小和节点映射可能不同，但给定的节点需要与之通信的等效节点总数与数据中心数量成正比（即不同数据中心的节点间通信并非全对全）。</u>

> 这句话的意思是，在不同的数据中心，键范围的大小和节点映射可能有所不同，但是某个节点需要与之通信的等效节点的总数与数据中心的数量成正比。 
>
> 因此，正如您的例子中所提到的，如果在某个集群中修改了某个键的主节点，那么这个修改不仅需要在集群内的副本之间进行异步复制，还需要在其他集群的等效节点之间进行异步复制。这个结构的目的是优化跨多个数据中心的通信效率和数据复制。

在写入操作在本地完成之后，主节点会将其放入一个复制队列，然后异步发送到远程等效节点。这些节点依次等待直到本地集群中满足值的依赖关系，然后在本地提交该值。此依赖项检查机制确保写入操作以因果一致的顺序进行，读取操作永不阻塞。

## 4.3 Client Library and Interface

COPS客户端库提供了一个简单直观的编程接口。图5展示了在照片上传场景下使用这个接口的示例。

```pseudocode
# Alice's Photo Upload
ctx_id = createContext() // Alice logs in 
put(Photo, "Portuguese Coast", ctx_id)
put(Album, "add &Photo", ctx_id);
deleteContext(ctx_id) // Alice logs out

# Bob's Photo View
ctx_id = createContext() // Bob logs in 
"&Photo" ← get(Album, ctx_id) 
"Portuguese Coast" ← get(Photo, ctx_id)
deleteContext(ctx_id) // Bob logs out
```

在示例中客户端API包含四个操作：

1. `ctx id←createContext()`
2. `bool←deleteContext(ctx id)`
3. `bool←put(key, value, ctx_id)`
4. `value←get(key, ctx_id) [在COPS中] 或 〈values〉 ← get trans (〈keys〉, ctx id) [在COPS-GT中]`

客户端API与传统键值接口有两点不同。首先，COPS-GT提供了`get trans`，它在单次调用中返回多个键值对的一致视图。<u>其次，所有函数都需要一个上下文参数（`ctx_id`），库在内部使用该参数跟踪每个客户端操作之间的因果依赖关系[49]。</u>上下文定义了因果+ "执行线程"。一个单独的进程可能包含许多独立的执行线程(例如，同时为1000多个独立连接提供服务的Web服务器)。通过分离不同的执行线程，COPS避免了因混淆它们而产生的错误依赖关系。

接下来，我们将描述COPS-GT客户端库中保持的状态，以在get事务中确保一致性。然后，我们将展示如何在COPS中存储更少的依赖状态。

COPS-GT客户端库（`COPS-GT Client Library`）。COPS-GT中的客户端库将客户端的上下文存储在一张包含`〈key, version, deps〉`条目的表格中。客户端通过API中的上下文ID (ctx id)来引用它们的上下文^4^。

> 在库中保持状态并传递ID是一种设计选择；也可以将整个上下文表编码为不透明的块，并在客户端和库之间传递它，以使库变为无状态。这样可以简化库的实现，但可能增加客户端和库之间交互的数据容量和复杂性。因此，设计选择需要权衡利弊。

当客户端从数据存储中获取一个键时，库将这个键及其因果依赖关系添加到上下文中。当客户端put操作放入一个值时，库将输入的依赖关系设置为当前上下文中每个键的最新版本。成功将值放入数据存储后，将返回分配给写入值的版本号v。然后，客户端库将这个新条目，〈key, v, D〉添加到上下文中。

![image-20230822152649197](https://s2.loli.net/2023/08/22/ECHnQDgJAcM5oey.png)

> 图6：客户端上下文的因果依赖关系示例图。箭头表示因果关系（例如，x3依赖于w1）。表格列出了每个值的所有依赖关系以及用于最小化依赖检查的“最近”依赖关系。

因此，上下文包括了客户端会话中先前读取或写入的所有值，以及所有这些依赖关系的依赖关系，如图6所示。这引发了关于此因果图潜在大小的两个担忧：(i) 存储这些依赖关系的状态要求，包括在客户端库和数据存储中；(ii) 在集群间复制写入时必须进行的潜在检查次数，以确保因果一致性。为了减轻客户端和数据存储跟踪依赖所需的状态，COPS-GT提供了垃圾收集功能，描述在第5.1节中，一旦它们提交给所有COPS副本，就移除依赖项。

为了减少在复制过程中的依赖性检查数量，客户端库确定了一些服务器可以使用的潜在优化。考虑图6中的图。y1依赖于x3，因此间接依赖于w1。如果存储节点在提交y1时确定x3已经被提交，那么它可以推断出w1也已经被提交，因此无需明确检查它。类似地，尽管z4直接依赖于t2和v6，但提交节点只需要检查v6，因为v6本身依赖于t2。

我们将必须检查的依赖项称为最近的依赖项（`Nearest Deps`），如图6中的表所示^5^。为了让服务器使用这些优化，客户端库首先计算写入的依赖项列表中的最近依赖项，并在发出写入时相应地标记它们。最近的依赖性足以让键值存储提供因果+一致性；完整的依赖性列表仅在COPS-GT中提供`get_trans`操作时才需要。

> 从图论的角度来说，一个值的最近依赖关系`Nearest Deps`，是指在因果图中到该值的最长路径长度为1的依赖关系。这意味着最近的依赖项是直接影响某个值的那些项，而不是通过其他依赖项间接影响该值的项。通过仅关注最近依赖关系，可以简化因果关系检查，从而提高系统的效率。

COPS客户端库（`COPS Client Library`）。COPS客户端库需要的状态和复杂性要显著降低，因为它只需要了解最近的依赖关系，而不是所有的依赖关系。因此，它不存储或检索其获取到的任何值的依赖关系：检索到的值比其任何依赖关系都更近，使它们变得不必要。

<u>因此，COPS客户端库只存储〈key, version〉条目。对于一个get操作，检索到的〈key, version〉将被添加到上下文。对于一个put操作，库使用当前上下文作为最近的依赖关系，清空上下文，然后只用这个put重新填充它。这个put依赖于之前的所有键-版本对，并且比它们更近。</u>

> 根据上述描述，您可以这样理解： 对于get()操作，COPS客户端库将获得的<key, version>添加到相应ctx_id的上下文中。而对于put()操作，库将当前上下文视为最近的依赖，这是基于客户端一致性的因果关系保证。
>
> 在执行put()操作之前，客户端应首先完成所有与新值相关的get()操作。因此，当执行put()时，当前上下文包含客户端在执行更新操作之前所了解的所有相关值，以及它们的版本信息。由于这种依赖关系是基于先前操作的特定上下文，在某种程度上可以确保它们是最近的依赖。
>
> 尽管标准模型不能保证绝对的实时一致性，但在因果一致性的前提下，这种近似是足够保证更新操作符合期望的一致性模型的。

所以，当put()操作成功时，上下文的内容将被清除，并且当前put()操作的内容将作为唯一依赖项被重新填充到上下文中。这意味着当前put()操作的内容被视为最近的依赖关系。这是因为通过清空当前上下文并添加当前put()操作的<key, version>条目，客户端库确保了紧跟在该put()操作之后的任何其他操作都依赖于此最近的操作。这有助于保持因果一致性，因为在同一上下文中执行的所有操作将考虑到新的put()操作并建立正确的因果依赖关系。

## 4.4 Writing Values in COPS and COPS-GT

基于客户端库和键值存储的描述，我们现在介绍在 COPS 中写入一个值所涉及的步骤。在 COPS 中，所有写入操作首先发送到客户端的本地集群，然后异步传播到远程集群。键值存储提供了一个统一的 API 调用以支持这两个操作：

`〈bool,vers〉 ← put_after (key, val, [deps], nearest, vers=∅)`

用于向本地集群写入数据（`writes to local cluster`）。当客户端调用 `put(key, val, ctx id)` 时，库计算出依赖项 deps 的完整集合，并将其中一些依赖项元组标识为值的最近依赖项。然后，库无版本参数地调用 put_after（即，设置 version=∅）。

> 因为在一个local cluster中，我们认为它当前的context中的内容在某种程度上代表了它的最近的依赖，因此不需要再读取当前key的deps，也是为了提高效率。

在 COPS-GT 中，库将 deps 包括在 put after 调用中，因为依赖项必须与值一起存储；在 COPS 中，库只需要包括 nearest，不包括 deps。^6^

> 我们使用括号符号([])来表示参数是可选的;可选参数在COPS-GT中使用，但在COPS中不使用。

本地集群中键的主存储节点为键分配一个版本号并将其返回给客户端库。<u>我们将每个客户端限制为一个未决（未完成的）的 put；这是必要的，因为后续的 put 必须知道较早 put 的版本号，以便它们可以依赖于它们。</u>

> 我们将每个客户端限制为一个未决的 put 的原因是要确保进程中的写操作能够按顺序发生。如果同时允许多个 put 操作，那么操作之间的因果关系将变得不明确，可能会导致一致性问题和意外的行为。
>
> 限制每个客户端只有一个未决的 put 操作并不是说在运行时客户端只能执行一个 put 操作。事实上，客户端可以执行多个 put 操作，但在同一时间，每个客户端只允许有一个正在执行的 put 操作。
>
>  这意味着客户端需要按顺序处理 put 操作。当一个 put 操作执行完成并提交后，客户端才能开始另一个 put 操作。这种未决 put 限制方法使得写入操作保持因果顺序，从而确保因果一致性。 简而言之，每个客户端在任何时候只能有一个 put 操作正在执行，但在整个程序运行期间，它可以执行多个 put 操作。

put_after 操作确保只有在依赖项列表中的所有条目都被写入后，val 才提交给每个集群。在客户端的本地集群中，这个属性自动成立，因为本地存储提供线性一致性。（如果 y 依赖于 x，那么 put(x) 必须在 put(y) 发出之前提交。）然而，在远程集群中，这一点并不可靠，我们将在下面讨论。

主存储节点使用 Lamport 时间戳[31]为每次更新分配一个唯一的版本号。节点将版本号的高位设置为其 Lamport 时钟，低位设置为其唯一节点标识符。<u>Lamport 时间戳允许 COPS 为每个键生成一个全局顺序。这个顺序隐式实现了最后写入者获胜的收敛冲突处理策略</u>。COPS 也能够显式地检测和解决冲突，我们将在第5.3节中讨论。<u>请注意，因为 Lamport 时间戳提供了一种遵守潜在因果关系的所有分布式事件的部分排序（`partial ordering`）方式，因此这种全局排序与 COPS 的因果一致性相容。</u>

> 什么是partial ordering?什么是 total ordering，如何进行过渡？

集群间的写入复制（`Write replication between clusters`）：在本地写入提交后，主存储节点使用一系列 `put_after` 操作异步地将该写入复制到不同集群中的等效节点；然而，在这里，主节点在 `put_after` 调用中包括键的版本号。与本地 put after 调用一样，在 COPS-GT 中包括 deps 参数，而在 COPS 中不包括。这种方法具有很好的扩展性，避免了单一序列化点的需求，但要求接收更新的远程节点只在其依赖项已提交到同一集群后才提交更新。

为了确保这个属性，一个从其他集群接收 `put_after` 请求的节点必须确定值的**最近依赖项**是否已经在本地集群中满足。它通过向那些负责这些依赖项的本地节点发起检查来实现：

`bool ← dep check (key, version)`

当节点收到 dep check 时，它检查本地状态，确定依赖项值是否已经被写入。如果是这样，它立即响应该操作。如果没有，它将阻塞，直到需要的版本被写入。

如果对最近依赖项的所有 dep_check 操作成功，处理 put after 请求的节点将提交已写入的值，使其在其本地集群中的其他读和写操作中可用。（如果任何 dep check 操作超时，处理 put after 的节点重新发出它，可能会发送给一个新节点，如果发生故障的话。）最近依赖项的计算方式确保了在值提交前所有依赖项都已满足，这反过来确保因果一致性。

### 4.4.1 partial ordering and total ordering

在分布式一致性模型中，部分排序（`partial ordering`）和全排序（`total ordering`）是两个重要的概念。它们描述了一个系统中事件的发生顺序。

部分排序（partial ordering）： 部分排序是指一组事件中的某些事件之间存在顺序关系，而其他事件可能没有明确的顺序关系。这意味着在系统中，仅针对一部分事件可以确定它们之间的次序。因果一致性（causal consistency）就是这样一个基于部分排序的一致性模型。在因果一致性中，只要存在因果关系（一个事件对另一个事件产生直接或间接影响）的事件，就需要保持它们的发生顺序。然而，没有因果依赖关系的事件可能并没有一个明确的顺序。

全排序（total ordering）： 全排序是指一组事件中的所有事件之间都存在明确的顺序关系。在分布式系统中，全排序通常需要额外的机制或协议来确定所有事件的执行顺序。在全排序中，系统中的每对事件都可以比较，从而可以确切地确定它们之间的先后次序。线性一致性（linearizability）和序列一致性（sequential consistency）是基于全排序的一致性模型。

从部分排序到全排序： 部分排序和全排序是一致性模型的两个端点，理论上，系统可以通过引入额外的同步和协调机制在这两者之间进行平衡。要从部分排序过渡到全排序，可以通过使用以下方法增加系统的同步程度：

1. 引入分布式锁或协议，例如：Paxos、Raft 或 Zab，这样可以对所有事件明确排序。
2. 使用原子广播（atomic broadcast）或全局逻辑时钟（如：Lamport timestamps 或 Vector clocks）以全局顺序传播事件。

然而，从部分排序过渡到全排序意味着支付更高的延迟和可用性代价，因为全局顺序需要在网络上进行更多的协调和沟通。在实际应用中，根据系统需求选择合适的一致性模型（部分排序还是全排序）并进行权衡是非常重要的。

## 4.5 Reading Values in COPS

与写操作一样，读操作也在本地集群中得到满足。客户端使用适当的上下文调用 get 库函数；库接着在本地集群中负责键的节点发出读请求：

`〈value, version, deps〉 ← get_by_version (key, version=LATEST)`

此读取可以请求键的最新版本或特定的较旧版本。请求最新版本相当于常规的单键 get；请求特定版本是启用 get 事务所必需的。因此，COPS 中的 get_by_version 操作总是请求最新版本。<u>收到响应后，客户端库将 `〈key,version[,deps]〉` 元组添加到客户端上下文中，并返回 value 给调用代码。deps 仅存储在 COPS-GT 中，而不是 COPS 中。</u>

## 4.6 Get Transactions in COPS-GT

由于使用单键 get 接口读取一组依赖键无法确保因果+一致性，即使数据存储本身是因果+一致的，COPS-GT 客户端库提供了 get_trans 接口。我们通过扩展相册示例以包括访问控制，说明这个问题，其中 Alice 首先将她的相册 ACL 更改为"仅限朋友"，然后编写她的旅行新描述，并将更多照片添加到相册中。

自然而然地（但错误地！）实现读取 Alice 专辑的代码可能会（1）获取 ACL，（2）检查权限，然后（3）获取专辑描述和照片。这种方法包含了一个简单的`time to check to time to use(从检查的时间到实际使用的时间)`竞争条件：当 Eve 访问相册时，她的 `get(ACL)` 可能返回旧的 ACL，允许任何人（包括 Eve）阅读，但她的 `get(album contents)` 可能返回"仅限朋友"的版本。

一个简单的解决方案是要求客户端按照它们的因果依赖关系相反的顺序执行单键 get 操作：如果客户端在 get(ACL) 之前执行 get(album)，上述问题将不会发生。然而，这个解决方案也是错误的。想象一下，更新专辑后，Alice 认为一些照片过于私人，于是她（3）删除了这些照片并重新写了描述，然后（4）再次将 ACL 标记为公开。这个简单方法具有不同的`检查时间到使用时间`错误，其中 get(album) 获取私有专辑，随后的 get(ACL) 获取"公共"ACL。

<u>简而言之，上例中的核心问题在于：ACL 和ACL专辑中的条目之间没有正确的规范排序。</u>

相反，更好的编程接口允许客户端获取多个键的因果+一致视图。实现此类保证的标准方法是在事务中读取和写入所有相关键；然而，这需要所有组合键的单一序列化点，COPS 为了实现更高的可扩展性和简单性而避免了这一点。相反，COPS 允许独立写入键（元数据中的明确依赖关系），并提供了 get_trans 操作以检索多个键的一致视图。

获取事务（`Get transactions`）：允许客户端以因果一致性的方式检索多个值。客户端可以通过调用 get_trans，传入所需的键集，例如 get_trans(〈ACL, album〉)。<u>根据发起请求的时间和地点，此 get 事务可以返回不同的 ACL 和 album 组合，但永远不会返回〈ACL~public~, Album~personal~〉。</u>

![image-20230823100507973](https://s2.loli.net/2023/08/23/NKjFdrlCma8kZ6U.png)

COPS 客户端库通过两轮算法实现 get 事务，如图 7 所示。在第一轮中，库向本地集群发出 n 个并发的 get_by_version 操作，每个客户端在 get trans 中列出一个键。因为 COPS-GT 在本地提交写操作，所以本地数据存储保证了这些显式列出的键的依赖项已经得到满足，即它们已经在本地写入，读取它们会立即返回。然而，这些显式列出的、独立检索的值可能彼此不一致（即这些值之间可能不符合casual+ consistency，但是值本身在写入时是满足casual+一致性的），如上所示。

<u>每个 get by version 操作返回一个〈value,version, deps〉元组，其中 deps 是一组键和版本。然后，客户端库检查每个依赖项条目 〈key, version〉。如果客户端没有请求依赖键，或者如果它确实请求了该键但检索到的版本大于等于依赖列表中的版本，那么该结果的因果依赖关系已经满足。</u>

> 在这段话中所提到的情景是客户端库在获取 get by version 结果后检查因果依赖关系是否满足。也就是说，客户端会审查结果，以确保返回值的依赖关系是满足的。
>
> 客户端库检查每个依赖项条目时，会遇到两种可能的情况：
>
> 1. 客户端没有请求依赖项键。在这种情况下，因果依赖关系是满足的，因为客户端并不关心相关的键。
> 2. 客户端请求了依赖项键。在这种情况下，需要比较检索到的版本和依赖项列表中的版本。如果检索到的版本 ≥ 依赖项列表中的版本，则因果依赖关系满足。
>
> 但是，某些情况下，客户端库可能会检索到比依赖项列表中的版本较小的版本。这意味着，尽管我们也请求了依赖项键，但相关的键并未满足因果依赖关系。这种情况可能的原因有：
>
> 1. 同步延迟：在分布式系统中，不同副本间的数据同步可能会引入延迟。在某些情况下，客户端可能会查询到一个尚未接收到最新数据的节点，从而返回一个较旧的版本。
> 2. 分区：在网络分区期间，某些节点可能会被隔离，无法及时更新。当客户端从这些节点查询数据时，可能会获取到较旧的版本。
>
> 在这些情况下，客户端库无法确定因果依赖关系是否满足。实际上，客户端库需要保证查询到的结果满足因果一致性。为了解决这个问题，客户端库可以执行以下操作：
>
> 1. 重新查询：客户端可以尝试从其他节点重新查询依赖项键，以获取更新的版本。
> 2. 等待最新数据：客户端可以等待一段时间，并期望系统最终达到一致状态。在一定时间内重试查询可能会得到满足因果依赖关系的数据。
>
> 简而言之，如果客户端检索到的版本小于依赖项列表中的版本，可能是由于分布式系统中的同步延迟或网络分区等因素导致的。为了解决这个问题，客户端库需要采取额外的措施，以确保查询结果满足因果一致性。

> 当检索到的键版本小于依赖项键版本时，这意味着在查询时，在那个时间点，数据存储中可能不存在依赖项键的相应版本。在这种情况下，因果一致性不会被满足。
>
> 这可能是由于数据在不同副本中的同步延迟，或者是网络分区导致某些节点不能及时更新。这时，因果依赖关系不满足，客户端库需要采取措施以获取一致的结果。例如，客户端可以重新查询其他节点以获取更新的依赖项键版本，或稍等片刻，期望分布式系统最终达到一致状态。

对于所有尚未满足的键，库会发出第二轮并发的 get by version 操作。请求的版本将是第一轮中任何依赖列表中看到的最新版本。这些版本满足了第一轮中的所有因果依赖关系，因为它们大于等于所需版本。此外，由于依赖关系是可传递的，而这些第二轮版本都是第一轮检索到的版本所依赖的，因此它们不会引入需要满足的任何新依赖关系。**注意，这一步操作在伪代码中并没有给出。**

<u>此算法允许 get trans 返回第一轮中检索到最新时间戳时数据存储的一致性视图。仅当客户端需要读取比第一轮中检索到的版本更新的版本时，才会执行第二轮。这种情况仅在 get 事务中涉及的键在第一轮期间更新时发生。因此，我们预期第二轮将不常出现。</u>

在我们的示例中，如果 Eve 在 Alice 的写操作期间发出 get_trans 请求，算法的第一轮 get 可能会检索到公共 ACL 和私有专辑-〈ACL~public~, Album~personal~〉。然而，私有专辑依赖于"仅限朋友"的 ACL，因此第二轮将获取这个更新后的 ACL 版本，使 get_trans 给客户端返回因果+一致性的值集。

> 回顾一下上例，Alice的相册ACL属性原来是public，之后Alice将相册权限修改为only-friends，之后往相册中添加照片内容。再之后，Alice 认为一些照片过于私人，于是她（3）删除了这些照片并重新写了描述，然后（4）再次将 ACL 标记为公开。

数据存储的因果+一致性为 get 事务算法的第二轮提供了两个重要特性。首先，get by version 请求将立即成功，因为所请求的版本必须已经存在于本地集群中。其次，新的 get by version 请求将不会引入任何新的依赖关系，因为由于传递性，在第一轮中这些依赖关系已经知道了。

第二个属性说明了为什么get事务算法在第二轮中指定了一个显式的版本，而不仅仅是获取最新的版本：<u>在有活动并发写入的情况下，如果客户端仅请求最新版本而不是指定显式版本，可能会出现不稳定的依赖关系。这意味着，新版本可能会引入更新的依赖关系，这种情况可能会无限地持续下去。</u>

而通过在第二轮中指定一个明确的版本，可以确保满足已知的依赖关系，同时避免引入未知的新依赖关系。综上所述，通过在第二轮中指定一个显式版本，get 事务算法可以更有效地确保因果+一致性，避免潜在的依赖关系问题，并在面对并发写入时提供稳定的结果。

# 5. Garbage,Faults,and confilicts

本节介绍了 COPS 和 COPS-GT 的三个重要方面：它们的垃圾收集子系统，减少了系统中额外状态的数量；它们对客户端、节点和数据中心故障的容错设计；以及它们的冲突检测机制。

## 5.1 Garbage Collection Subsystem

COPS 和 COPS-GT 客户端存储元数据；COPS-GT 服务器额外保留键的多个版本和依赖项。如果不进行干预，随着键更新和插入，系统的空间占用将不受限制地增长。COPS 垃圾收集子系统删除不需要的状态，保持系统总大小在适当范围内。第 6 节评估了维护和传输这些额外元数据所带来的开销。

版本垃圾收集（`Version Garbage Collection`）（仅限 COPS-GT） 

存储内容（`What is stored`）：COPS-GT 存储每个键的多个版本，以满足客户端的 get by version 请求。 

清理原因（`Why it can be cleaned`）：get trans 算法限制了完成 get 事务所需的版本数量。算法的第二轮仅对第一轮返回的版本较晚的版本发出 get by version 请求。为了便于及时进行垃圾收集，COPS-GT通过可配置参数 trans_time（在我们的实现中设置为 5 秒）来限制 get trans 的总运行时间。 （如果超时触发，客户端库将重新启动get trans调用，并使用更新的键版本满足事务；我们预计只有当集群中有多个节点同时发生崩溃时，才会发生这种超时情况。）

> trans_time：规定的get_trans操作的最大可运行持续时间，超过这个时间要重新运行get_trans。注意这个trans_time参数，将贯穿GC的整个过程。

清理时间（`When it can be cleaned`）：<u>在写入一个键的新版本后，COPS-GT 只需将旧版本保留 trans_time 加上一个用于时钟偏移的小差值。</u>在此时间之后，不会再出现对旧版本的 get by version 请求，垃圾收集器可以将其移除。

空间开销（`Space Overhead`）：空间开销由在 trans_time 内可以创建的旧版本数量限定。此数字由节点能够承受的最大写入吞吐量决定。我们的实现中每个节点可以承受 105MB/s 的写入流量，需要额外的 525MB 缓冲空间（可能）来存储旧版本。<u>此开销是每台机器的，不会随集群规模或数据中心数量增长。</u>

依赖项垃圾收集（`Dependency Garbage Collection`）（仅限 COPS-GT） 

存储内容（`What is stored`）：存储依赖项以允许get事务获得数据存储的一致视图。

为什么可以清理（`Why it can be cleaned`）：一旦与旧依赖关联的版本不再用于get事务操作的正确性，COPS-GT就可以收集这些依赖关系。

为了说明何时get事务操作不再需要依赖项信息，考虑值z2依赖于x2和y2。一个获得x,y和z的事务要求：如果返回了z2，那么x~≥2~和y~≥2~也必须一并返回（这里说的是版本>=2）。<u>因果一致性确保在z2被写入之前，x2和y2都已经写入。因果+一致性的始终推进的属性确保一旦x2和y2被写入，他们或更后期的版本将总是通过get操作返回。</u>因此，当z2已经写入所有数据中心，且过了trans time时间后，任何返回z2的get事务都将返回x≥2和y≥2。因此，z2的依赖项可以进行垃圾收集（指x2与z2）。

> 根据规则，如果data store中存在更新的版本，请求超时后，下一次请求就会请求更新的版本。

什么时候可以清理（`When it can be cleaned`）：在数据中心中提交一个值后的trans time秒后，COPS-GT可以清理一个值的依赖项（回想一下，提交是强制满足其依赖项的过程，这里的依赖项是针对于特定版本的）。COPS和COPS-GT都可以进一步设置该值的`never-depend`标志。为了清理依赖项，每个远程数据中心在写入提交并达到超时时间后通知原始数据中心。一旦所有数据中心确认，原始数据中心就会清理自己的依赖项，并通知其他数据中心同样执行。<u>为了减少清理依赖项所占用的带宽，副本仅在键的这个版本在经过trans time秒时间后是最新的版本时通知原始数据中心；**如果不是，那么无需收集依赖项，因为整个版本将被收集^7^。**</u>

> 作者提到，他们正在研究以这种方式收集依赖项是否相较于在全局检查点时间之后收集依赖项（下文讨论）提供足够的优势，从而证明其消息成本是合理的。
>
> 所以意思就是旧版本的依赖项会用全局时间检查点时间的策略收集。
>
> 在分布式数据存储系统中，实时垃圾收集有助于维护系统性能和最佳状态。本文的作者正在探讨在特定时间上收集依赖项（trans_time）与全局检查点时间后收集依赖项之间的权衡。实时收集可能带来更低的延迟和更高效的系统，但可能导致额外的通信开销。与此相反，使用全局检查点时间可能需要更少的消息传递，但有可能积累更多无用数据，从而降低系统性能。
>
> 通过研究这两种方法，作者试图确定它们之间的最佳平衡，以确定哪种方法在确保系统性能和收集效率方面更具优势。确保最佳垃圾收集策略对于提高分布式存储系统的可扩展性和可维护性至关重要。

上面那段话中我有太多的疑问，因此解答如下：

1. “如果它不是最新版本，那么就没有必要收集依赖项，因为整个版本会被收集。”？

>  在COPS-GT系统中，我们将只关注任何给定键的最新版本。当系统决定要清理依赖项时，目标是清理与最新版本相关的依赖项，以便系统可以正确地执行其他操作。
>
> 这句话指的是：如果某个版本在trans_time后不是最新版本（有更高版本的键存在），那么就没有必要收集这个旧版本的依赖项。这是因为在实际操作中，系统将重点关注最新版本的依赖项。一旦旧版本被新版本覆盖，即使系统没有收集旧版本的依赖项，对整体系统性能和一致性也不会产生影响。系统将继续处理最新版本的依赖，而不会关注已过时的旧版本依赖。
>
> 系统针对最新版本和旧版本各有一套主要的GC策略：
>
> 旧版本的依赖项主要通过全局检查点策略进行收集和清理，而trans_time策略主要用于收集最新版本的依赖项。
>
> 全局检查点策略是在整个系统中，所有节点都达成的最新的Lamport时间戳。这个时间戳作为一个审查标准，来收集和清理旧版本的依赖项。此策略不仅用于共识，还用于在客户端和服务器节点上减少依赖项数量。
>
> 与此同时，trans_time策略主要关注最新版本的依赖项。系统在trans_time秒后，收集并清理与最新键值版本关联的依赖项。这有助于减少因持续清理过程而产生的通信开销，同时保持系统的因果一致性。
>
> 因此，COPS-GT系统充分利用全局检查点策略和trans_time策略，共同收集和清理不同版本的依赖项。这两种策略相辅相成，确保系统一致性，同时实现分布式存储系统的各种性能优化。

2. 如何确定键的版本是否是最新的？

> 在分布式存储系统中，当事务提交一个值时（对应于一个键的特定版本），会有一定的时间（trans_time秒）以确保所有依赖项满足因果关系并到达所有副本。这种方法确保了全系统的一致性。一旦过了trans_time秒，我们可以认为一个键的新版本已经成功地覆盖了之前的版本，并确保所有副本都接收到了更新版本。
>
> 系统通过因果关系和Lamport时间戳确定目标键的版本是否为最新版本。

> 如何确定某个键的版本是否是最新的？
>
> 在COPS-GT分布式存储系统中，在提交写入操作或事务时，系统会为键分配一个Lamport时间戳。Lamport时间戳是一种逻辑时间戳，用于表示事件发生的顺序，确保分布式系统中的因果一致性。每当客户端执行一个操作，Lamport时间戳就会递增。
>
> 当一个新的键值被写入时，它会包含一个时间戳，表示其在系统中的版本。副本会根据这些时间戳来比较各个版本，并将具有较新时间戳的版本视为最新版本。当新版本到达副本时，副本会更新其关于该键的信息。在传输时间（trans_time）之后，各个副本应该已经接收并存储了新版本。
>
> 因此，系统通过比较各个数据中心副本上存储的时间戳来确定目标键的某个版本是否为最新版本。在满足因果关系和传输时间（trans_time）后，新写入的版本应该在所有副本中广泛传播并被视为最新版本。

3. 旧版本的依赖项该如何被清理呢？

> COPS-GT系统在一定程度上提高依赖项清理效率。例如，在清理新版本之后，由于客户端导致的通信开销会减少，因此也有可能减少旧版本的依赖关系数量。这主要是通过以下两种方式实现的：
>
> 1. 一旦某个键的版本成功提交到所有数据中心，COPS会将该键版本标记为never-depend（永不依赖），以表明客户端无需表示对其的依赖（因为已经写入成功了，所以这个值是必定满足依赖的，因此它不再需要任何其他依赖，就可以正常工作）。此时，客户端库会立即从客户端上下文的依赖项列表中删除这些never-depend项，并可逐步回收以前版本的依赖项。
> 2. COPS存储节点从put操作（写入操作）中删除不必要的依赖项。当节点接收到一个put操作时，它会检查依赖项列表中的每个条目，并删除版本号早于全局检查点时间的条目。检查点时间是整个系统中所有节点都满足的最新的Lamport时间戳。COPS键值存储数据库将此检查点时间返回给客户端库（例如，作为put操作的响应），从而允许库在客户端上下文中清除这些依赖项。
>
> 总之，COPS-GT系统尽量在客户端和存储节点上清理依赖关系。虽然关注最新版本的依赖项，但是对于旧版本，通过全局检查点过程和never-depend策略，系统也会逐步清理它们。

空间开销（`Space Overhead`）：在正常操作下，依赖项在trans time加上一个往返时间后进行垃圾收集。**依赖项仅在键的最新版本上进行收集；较早版本的键已经如上所述进行了垃圾收集。**

在分区期间，无法对最近版本的键的依赖项进行收集。这是COPS-GT的局限性，尽管我们预计较长的分区事件是非常罕见的。为了说明为什么这种让步对于`get`事务正确性是必要的，请考虑值b2依赖于值a2：如果b2对a2的依赖过早地被收集，那么后期产生的一个值c2可能在没有a2的明确依赖下产生条件依赖于b2——从而依赖于a2。然后，如果a2、b2和c2都在短时间内复制到一个数据中心，get事务的第一轮就可能获得a1（a的早期版本）和c2，然后将这两个值返回给客户端，因为它不知道c2依赖于更新的a2。

> 此时得到的c2的依赖项的版本就是错误的：因为c2直接依赖于b2，因此根据传递性，C2也依赖于a2，但是a2已经被回收，只有a1，所以c2会得到错误的依赖。

客户端元数据垃圾回收（`Client Metadata Garbage Collection`）（COPS + COPS-GT）

存储内容（`What is Stored`）：COPS客户端库使用ctx id在客户端会话（执行的单个线程）期间跟踪所有操作。与上文中讨论的位于键值存储本身的依赖信息相反，这里讨论的依赖项是客户端元数据的一部分，存储在客户端库中。在这两个系统中，自上一个put操作以来的每个get都会增加一个最近的依赖项。另外，在COPS-GT中，get trans操作返回的所有新值及其依赖项以及所有put操作都会添加常规依赖项。如果客户端会话持续时间很长，附加到更新的依赖项数量会变得很大，从而增加了COPS需要存储的依赖项元数据的大小。

为什么可以清除（`Why it can be cleaned`）：与上述依赖跟踪一样，客户端只需在依赖项被保证在任何地方得到满足之前跟踪依赖项。

何时可以清除（`When it can be cleaned`）：COPS通过两种方式减少客户端状态（上下文）的大小。**首先，如上所述，一旦put后成功提交到所有数据中心，COPS会将该键版本标记为`never-depend`，以表明客户端无需表示对其的依赖。**用`get_by_version`获取的结果包括此标志，客户端库会立即从客户端上下文的依赖项列表中删除never-depend项。<u>此外，此过程具有传递性：一个敢never-depend密钥所依赖的任何东西必须已被标记为永远不依赖，因此它也可以从上下文中回收。</u>

> 比如说，c依赖于b，b依赖于a，b被成功写入后，存储给b打上了never-depend的标记，所以当我们读取c以后，发现依赖项列表中有b存在，这是不是意味着我可以删除依赖项b，即客户端知道c依赖于b，但是b的依赖是满足的，因此我不需要再跟踪这个依赖b？
>
> 是的，你的理解是正确的。在你给出的例子中，c依赖于b，b依赖于a。一旦b成功写入并被标记为never-depend，这意味着b已经满足了它所需要的所有依赖（即a）。因此，在读取c时，我们可以看到b存在于依赖列表中，并且已经被标记为never-depend。
>
> 这意味着我们可以从客户端状态（或者说上下文）中移除对b的追踪。因为虽然c确实依赖于b，但由于b已经满足了所有必要的依赖并且被标记为never-depend，所以我们不再需要跟踪这个依赖项。客户端只需要知道它当前还有哪些未满足的依赖项即可。
>
> 总之，在COPS数据库系统中，“清理”过程就是移除那些已经满足了所有必要条件并且被标记为never-depend的键值对。
>
> 所以本质上：是的，在 COPS 系统中完成一次成功写入操作并随后将一个键标记为 never-depend 标志后，客户端被允许立即停止跟踪此项依赖。

其次，COPS存储节点从`put_after`操作中删除不必要的依赖项。当节点接收到put后时，它会检查依赖项列表中的每一项，并删除版本号早于全局检查点时间global checkpoint time的项目。此检查点时间是整个系统中所有节点都满足的最新的Lamport时间戳。COPS键值存储数据库将此检查点时间返回给客户端库（例如，以响应put后），允许库清除上下文中的这些依赖项。^8^

> 在客户端和cops存储中，使用全局时间戳方式清除旧版本的依赖。
>
> 由于未完成的读取，客户端和服务器还必须等待几秒钟才能使用新的全局检查点时间。

为了计算全局检查点时间（`global checkpoint time`），每个存储节点首先确定其负责的键范围内任何未完成(`pending`)的`put_after`的最旧的Lamport时间戳。（换句话说，它确定在所有副本中不保证满足的最古老的键的时间戳。）然后，它联系其他数据中心中的等效节点（处理相同键范围的节点），这些节点两两交换最小Lamport时间，并记住已观察到的任何副本的最旧Lamport时钟。

在此步骤结束时，所有数据中心都具有相同的信息（这里的相同值的是定义上的相同）：每个节点都知道其键范围中的全球最旧的Lamport时间戳。然后，数据中心内的节点通过gossip协议去获取周围数据节点中每个键范围的最小值，以找到其中任何一个观察到的最小Lamport时间戳。此周期性过程在我们的实现中每秒进行10次，并且对性能几乎没有影响。

注意：Lamport 时间戳是一种逻辑时钟系统，用于跟踪事件发生顺序而不依赖物理时钟。

> 节点在数据中心之间使用类似于gossip的协议来交换它们负责的键范围的检查点时间，并比较所有节点的检查点时间以获得全局最小检查点时间。
>
> 这个过程有两个主要步骤：
>
> 1. 每个存储节点首先确定其主键范围内任何待处理put_after操作的最旧Lamport时间戳（也就是说，确定在所有副本中不能保证满足条件的最旧键值对应时间戳）。然后，它会联系其他数据中心中负责相同键范围的节点，并与这些节点交换他们各自记录下来的最小Lamport时间。
> 2. 在第一步结束后，每个数据中心内部都会有一个全局最旧Lamport时间戳（这里是只针对某个节点负责的键范围的，并不是真正意义上全局）。然后，一个数据中心内部各个节点之间会使用gossip-like协议进行通信和比较，找出这些全局最旧Lamport时间戳里面最小（也就是说，在所有记录里面“最老”的）那一个。
>
> 总结起来，每个节点首先确定其自身负责区域内全局最旧Lamport时间戳，并与处理相同键范围的其他数据中心节点交换信息。然后，在数据中心内部各个节点之间找出其中最小（即“最老”）那一个时间戳，作为全局检查点时间。

## 5.2 Fault Tolerance

COPS对客户端、节点和数据中心的故障具有弹性。在以下讨论中，我们假设故障是`fail-stop`：组件在遇到故障时会停止运行，而不是错误地或恶意地操作，而且故障是可以检测的。

客户端失败（`Client Failures`）。COPS的键值接口意味着每个客户端请求（通过库）都由数据存储系统独立且原子性地处理。从存储系统的角度看，如果一个客户端失败了，它就会停止发出新的请求；没有必要进行恢复。从客户端的角度看，COPS的依赖性跟踪使得处理其他客户端的失败变得更容易，因为它确保了如引用完整性等属性。考虑照片和相册例子：如果一个客户端在写入照片后、但在写入对照片的引用之前失败了，数据存储仍然会处于一致状态。永远不会出现有对照片的引用但却没有写入照片本身这样不一致情况。

键值节点故障（`Key-Value Node Failures`）。COPS可以使用任何底层的容错线性化（强一致性）键值存储。我们在FAWN-KV [5] 节点的独立集群之上构建了我们的系统，这些节点在一个集群内使用链复制[51]来掩盖节点故障。因此，我们描述了COPS如何使用链复制来提供对节点故障的容忍。

与FAWN-KV的设计类似，每个数据项都存储在一致哈希环上连续的R个节点组成的链中，put_after操作被发送到适当链条的头部，沿着链条传播，然后在尾部提交，然后尾部确认操作。`get_by_version`操作被发送到尾部，由尾部直接响应。

服务器发出的操作稍微复杂一些，因为它们是由不同链条的节点发出和处理的。本地集群中的尾节点将put_after操作复制到每个远程数据中心中头结点。远程头结点然后发送dep check操作（本质上是读取操作）给他们本地集群中适当位置（即对应键范围）尾结点。一旦这些返回（如果该操作没有返回，则会触发超时并重新发出dep check），远程头结点将值沿着(远程)链传播到远程尾结点，并提交该值，并向原始数据中心确认该操作完成。

依赖的垃圾收集遵循类似于交叉锁定链条模式, 尽管我们为了简洁起见省略了细节内容。版本垃圾收集在每个节点上本地进行，并且可以像单个节点情况一样运行。全局检查点时间计算, 用于客户端元数据垃圾收集, 每个tail正常更新其对应键范围最小值。

数据中心故障（`Datacenter Failures`）。COPS的分区容忍设计也提供了对整个数据中心故障（或分区）的弹性。面对这样的故障，COPS会像正常情况一样继续运行，但有几个关键的不同。

首先，任何在失败的数据中心中产生但尚未复制出去的`put_after`操作将会丢失。这是允许低延迟本地写入返回速度比数据中心之间传播延迟更快带来的不可避免成本。如果数据中心只是被分区而没有失败，那么不会丢失任何写入操作。相反，它们只会被延迟直到分区恢复。

> 支持在因果+模型中提交所需的数据中心数量的灵活性仍然是未来工作的一个有趣方面。
>
> 在当前设计中，COPS系统通常会将每个写入操作复制到所有数据中心以保证全局一致性。然而，这种方式可能会带来较高的延迟和资源消耗。如果能够支持对提交所需数据中心数量的灵活配置，那么就可以根据具体情况调整复制策略，以在保证一致性的同时优化系统性能。
>
> 例如，在某些情况下，我们可能只需要将写入操作复制到部分数据中心即可满足一致性要求。或者，在面临网络分区或数据中心故障时，我们可以临时减少提交所需的数据中心数量以继续服务。
>
> 这种灵活性需要通过更复杂、更智能的协议和算法来实现，并且需要考虑各种可能出现的场景和问题。例如：如何选择最佳的复制目标？如何处理不同数据中心间可能存在的网络延迟差异？如何确保在减少了提交所需数据中心数量后仍然能够保证全局一致性？等等。这些都是未来研究需要解决的问题。

其次，在活跃的数据中心中需要为复制队列提供的存储空间将增长。他们无法向失败的数据中心发送put_after操作，因此COPS无法对那些put_after操作涉及到的deps进行垃圾回收。系统管理员有两个选项：如果分区可能很快恢复，则允许队列增长；或者重新配置COPS以不再使用失败的数据中心。

第三，在COPS-GT系统下，如果一个数据中心发生故障，则依赖性垃圾收集不能继续进行，直到该分区恢复或者系统被重新配置以排除失败的数据中心。

## 5.3 Conflict Detection

<u>冲突发生在对给定键有两个“同时”（即，不在同一上下文/执行线程中）写入的情况。</u>默认的COPS系统使用最后写入者获胜策略来避免冲突检测。通过比较版本号来确定“最后”的写入，这使我们可以避免冲突检测以提高简单性和效率。我们认为这种行为对许多应用程序都是有用的。然而，也有一些其他应用程序，在更明确的冲突检测方案下会变得更容易理解和编程。对于这些应用程序，COPS可以配置为检测冲突操作，然后调用某些特定于应用程序的收敛冲突处理器。

我们将带有冲突检测的COPS称为COPS-CD，并向系统添加了三个新组件。

首先，所有put操作都携带着先前版本（`previous version`）的元数据，该元数据指示了在写入时本地集群可见的该键最近一次的先前版本（此先前版本可能为空）。

其次，所有put操作现在都隐式依赖于那个先前版本，这确保了新版本只会在其旧版之后被写入。这种隐式依赖意味着需要一个额外dep check操作，尽管它开销低并且总是在本地机器上执行。

第三点, COPS-CD拥有一个由应用程序指定的收敛性处理器, 当发现一个冲突时就会调动它。

COPS-CD遵循一个简单的处理过程来确定一个键（具有先前版本prev）新的put操作（new版本）是否与键当前可见版本curr冲突：

当且仅当new和curr冲突时，prev ≠ curr。

我们省略了完整的证明，但在这里提供直观理解。在正向方向上，我们知道prev必须在new之前写入，即 `prev≠curr`，并且为了使curr可见而不是prev，我们必须有curr > prev这是由于因果+的进展性质。但是因为prev是new最近的因果先前版本，我们可以得出结论 curr-/>new。另外，因为curr在new之前写入，它不能在它之后产生因果关系，所以 new-/>curr 因此他们冲突（二者没有因果关系）。

反向来看, 如果new和curr冲突, 那么就会有 curr-/>new. 根据定义, 我们知道 prev ->new, 所以就可以推导出 curr≠prev.

# 6.Evaluation

不看，后期需要实验再看。

# 7. Related Work

我们将相关工作分为四类：ALPS系统、因果一致性系统、线性化系统和事务性系统。

`ALPS System`：这个日益拥挤的类别包括最终一致性键值存储，如亚马逊的 Dynamo [16]、LinkedIn 的 Project Voldemort [43] 和广受欢迎的 memcached [19]。Facebook 的 Cassandra [30] 可以配置为使用最终一致性来实现 ALPS 属性，或者可以牺牲 ALPS 属性以提供线性化。对我们的工作产生关键影响的是雅虎的 PNUTS [15]，它提供了每键顺序一致性（尽管他们将此命名为每记录时间线一致性）。然而，PNUTS 不在键之间提供任何一致性；实现这种一致性引入了 COPS 需要解决的可扩展性挑战。

因果一致系统（`Causally Consistent Systems`）：许多先前的系统设计师都认识到了因果一致性的效用。Bayou [41] 提供了一个 SQL-like 接口给单机副本，实现了因果+ 一致。Bayou 在本地处理所有读写操作；它没有考虑我们考虑的可扩展目标。

TACT[53] 是一个使用顺序和数值边界来限制系统中副本发散度的因果+ 一致性系统。ISIS[12] 系统利用虚拟同步概念[11]为应用程序提供一个因果广播原语(CBcast)。

CBcast 可以直接用于提供一个因果一致键值存储。通过因果内存共享信息的副本也可以提供一个因果一致性的ALP 键值存储。<u>然而, 这些系统都需要单机副本, 因此不具备可扩展能力。</u>

PRACTI [10] 是一个支持部分复制的因果+一致ALP系统，它允许副本只存储一部分键，从而提供了一定的可扩展性。然而，每个副本——以及提供因果+一致性的键集——仍然受限于单机可以处理的范围。

懒惰复制[29]最接近 COPS 的方法。懒惰复制明确标记更新及其因果依赖，并等待这些依赖满足后再在副本上应用它们。这些依赖通过一个与我们的客户端库类似的前端进行维护和附加到更新上。然而，懒惰复制的设计假设副本限于单机：每个副本都需要一个能够（i）创建所有副本操作的顺序日志、（ii）将该日志传播给其他副本、（iii）将其操作日志与其他副本合并，并最终（iv）按照因果顺序应用这些操作。

最后，在并行理论工作中，Mahajan等人[35]定义了<u>实时因果(RTC)一致性</u>，并证明了它是在始终可用系统中可以达到的最强大的一致性。RTC比因果+更强大，因为它强制实施了一个实时要求：<u>如果因果-并发写入实时不重叠，则早期写入不可能在后期写入之后进行排序。</u>这种实时要求有助于捕捉系统隐藏的潜在因果关系(例如，带外消息传递[14])。

> "因果并发写入"通常是指两个或更多的写操作，它们之间没有明确的因果关系。也就是说，这些操作可以被视为同时（或并发）进行的，因为我们无法确定它们之间的顺序。这可能是由于它们在不同节点上独立执行，或者它们在相同节点上执行但彼此没有直接依赖关系。
>
> 对于这样的并发操作，因果一致性模型（如COPS使用的）允许它们以任何顺序出现。只有当一个操作明确地依赖另一个操作（即存在因果关系）时，才需要保证特定的顺序。
>
> 所以，“因果并发写入”通常指那些我们不能清楚地确定其间关系、可能以任何顺序出现的写入操作。

相比之下, 因果+没有实时要求, 这使得更高效率地执行变得可能. 值得注意地是, COPS高效率地"最后写者胜出"规则导致产生了一个符合因果+, 但不符合RTC 一致性系统. 而`return-them-all`（保留并发写入的所有版本）冲突处理程序将提供这两个属性。

> 因果+一致性模型（Causal+）并不需要满足实时要求，这为更高效的实现提供了可能。特别是，COPS中高效的“最后写者胜出”规则会导致一个满足因果+但不满足RTC一致性的系统，而“全部返回”冲突处理器则可以同时提供这两个属性。
>
> 具体来说，“最后写者胜出”规则在处理并发写入操作时只保留了最新（即最后完成）的操作结果，而忽略了<u>先完成但晚到达</u>的操作。这种做法简化了冲突解决过程，并减少了需要存储和传输的数据量，从而提高了系统效率。
>
> 然而，“全部返回”冲突处理器在遇到并发写入时会保留所有结果，并将它们一起返回给客户端。这样可以确保所有操作按照它们在物理时间上发生的顺序被看到，从而满足RTC一致性要求。当然，这种做法需要更多存储和传输资源，并可能增加客户端处理复杂性。
>
> 总体来说，在设计分布式系统时需要根据应用需求和资源限制权衡各种因素，包括一致性模型、实现复杂度、系统效率等等。

`Linearizable Systems`：线性化可以通过使用单个提交点（如主备份系统[4, 39]，它们可能会通过两阶段提交协议[44]积极地复制数据）或使用分布式协议（例如 Paxos [33]）来提供。quorum（仲裁）系统不是到处复制内容，而是确保读写集合的交集以实现线性化[22, 25]。

<u>正如前面提到的，CAP定理指出，线性化系统不能具有低于其往返数据中心延迟的延迟</u>；直到最近，它们才被用于广域操作，并且只有在可以牺牲ALPS的低延迟时才会这样做[9]。CRAQ [48]在写争用少时可以在局域网内完成读取，但其他情况下需要广域操作以确保线性化。

> 线性一致性（Linearizability）是一种强一致性模型。在线性一致系统中，所有操作都必须以它们实际发生的顺序被看到。这意味着如果一个操作在另一个操作之前完成，那么任何观察者都应该看到第一个操作在第二个操作之前完成。
>
> 为了满足这个要求，线性一致系统必须等待所有数据中心确认写入操作才能将其视为已完成。这就导致了线性一致系统无法具有低于其往返数据中心延迟的延迟。换句话说，如果一个请求需要从客户端发送到数据中心A然后再转发到数据中心B，并且每个数据中心都需要确认写入成功，那么总延迟就至少等于客户端到A、A到B以及B回复A和A回复客户端四段路程的总时间。
>
> 所以，在保证强一致性（如线性化）的同时还要求低延时是非常困难的。设计者通常需要根据具体应用需求进行权衡选择。
>
> 当发生网络分区时，我们只能确保CP或者AP成立，CA是不可能的。

事务（Transactions）。与大多数文件系统或键值存储不同，数据库社区长期以来一直通过使用读写事务来考虑跨多个键的一致性。在许多商业数据库系统中，单个主服务器在键之间执行事务，然后将其事务日志懒派给其他副本，可能跨越广域网。通常，这些异步副本是只读的，与COPS的任意写入副本不同。如今的大规模数据库通常会将数据分区（或分片）到多个DB实例[17, 38, 42]上，就像在一致性哈希中一样。事务仅在单个分区内应用，而COPS可以建立节点/分区之间的因果依赖关系。

几种数据库系统支持跨分区和/或数据中心（两者都被视为独立站点）的事务。例如，R*数据库[37]使用进程树和2PL进行多站点事务处理。然而，这种2PL阻止了系统保证可用性、低延迟或分区容忍性。Sinfonia [1]通过轻量级两阶段提交协议为分布式共享内存提供"迷你"事务处理，但仅考虑单个数据中心内的操作。最后, Walter [47],一个新近出现的针对广域网的键值存储器, 提供了跨键(包括写入操作, 这是COPS所没有的) 的事务一致性，并包含允许在某些场景下，在单个站点内执行事务处理操作优化功能.。

但是，在COPS专注于可用性和低延迟时,Walter则强调事务保证：确保键之间存在因果关系可能需要跨广域网进行两阶段提交. 此外，在COPS中, 可扩展数据中心是首要设计目标, 而Walter目前由单台机器组成(作为事务序列化点).

# 8. Conclusion

今天的大规模广域系统为客户端提供“始终在线”的低延迟操作，但代价是一致性保证较弱，应用程序逻辑复杂。本文提出了一种可扩展的分布式存储系统COPS，它在不牺牲ALPS特性的情况下提供因果+一致性。在每个集群中暴露其写操作之前，COPS通过跟踪和显式检查因果关系是否得到满足来实现因果+一致性。基于COPS，我们构建了COPS-GT，引入了get事务功能使客户能够获得多个键值的一致视图；COPS-GT还整合了一些优化措施以减少状态记录，最小化多轮协议，并减少复制开销。我们的评估显示，COPS和COPS-GT都能提供低延迟、高吞吐量和可扩展性。

[back](../../index.html).