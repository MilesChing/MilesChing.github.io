---
title: Kubernetes Controller：从 ReplicaSet 开始
date: 2020-10-03T08:33:10+08:00
cover: https://www.instana.com/media/Kubernetes-Vs-Docker-01-01.png
tags: [Kubernetes]
draft: true
---

在 Kubernetes Control Plane 提供的多个原生工作负载（Workload）中，ReplicaSet 是最简单和基础的一个。本文对 Kubernetes Controller 的普适性结构设计和其中一部分细节作了介绍，并借此对 ReplicaSet Controller 的源代码进行了分析。

## Kubernetes 控制平台

Kubernetes 的声明式（Declarative）接口不仅是与命令式接口并列的某种交互方法，而是深深根植于它控制平台的全部设计。在这个声明式交互的系统中，系统管理员通过声明对象（Object）将自己期望的行为或运行方式告知系统，系统通过对现有对象的观察和解析不断改变自己的行为。这里的对象即 Storage 中的一个 Structure，其各字段和语义都已事先被系统和使用者约定，管理员与系统的交互方式并非通过命令语句而仅仅通过对这些 Structure 的创建、删除和修改。对于每种不同的 Structure 类型，Kubernetes 通过一个对应的 Controller 来处理上述观察、解析和系统行为的调整。

> 在机器人技术和自动化技术中，控制循环是调节系统状态的无终止循环。

Kubernetes Controller 是监视集群状态的控制循环，在每个循环中根据需要做出或请求更改。每个 Controller 监控至少一个 Kubernetes 资源类型，负责其全部对象实例的正常功能，试图做出行为将当前的集群状态移动到更接近期望的状态。这些对象一般具有表示所需状态的 Spec 字段和表示当前状态的 Status 字段。Spec 是让用户写入期望的状态，系统可以通过 Spec 读出用户的期望。Status 是系统写入观察到的状态，用户可以从中读出系统当前是什么状态。Controller 用以维护集群的这些 “行为” 绝大多数通过调用 API Server 得以完成，且大多数同样是基于声明式系统的。举例来说，ReplicaSet 是一个负责调整 Container 数量的 Controller ，但它在需要创建和删除 Container 时仅仅需要在数据库中创建或删除 Pod 对象即可，Pod Controller 会立即关注到这些改动并实际创建出所需的 Container 以完成它的工作。在这个过程中 API Server 提供了在数据库中创建、删除和修改对象的统一接口，但实际工作是由 Controller 完成的。实际上这种通过 Object 的交互是 Controller 之间协作的普遍方式（如 Deployment 与 ReplicaSet 的协作）。

### Controller Manager

从逻辑上讲，每个 Controller 都是一个单独的进程，但为了降低复杂性，它们被编译成同一个二进制文件，并在单个进程中运行（即 Kube Controller Manager）。Controller Manager 初始化共享资源 [Controller Context](https://github.com/kubernetes/kubernetes/blob/release-1.18/cmd/kube-controller-manager/app/controllermanager.go#L289) ，包括 Informer Factory 、Client Factory 、Cloud Provider Interface 、Common Configuration 等，并并行运行所有 Controller 。在函数 [`NewControllerInitializers`](https://github.com/kubernetes/kubernetes/blob/release-1.18/cmd/kube-controller-manager/app/controllermanager.go#L372) 你可以看到 Controller 的入口被初始化，在 [`Run`](https://github.com/kubernetes/kubernetes/blob/release-1.18/cmd/kube-controller-manager/app/controllermanager.go#L159) 中它们被并行启动。多个 Controller Manager 实例之间通过 Leader Election 的方式协作以保证其可用性，这里我们不多赘述 Controller Manager 的细节。

![Kubernetes 控制平台](https://d33wubrfki0l68.cloudfront.net/7016517375d10c702489167e704dcb99e570df85/7bb53/images/docs/components-of-kubernetes.png)

### Controller 设计思想

总结起来，Controller 的工作即监控用户定义的对象并在 Status 与 Spec 不一致时采取相应的行动，这种 “监控” 是以 Control Loop 的形式完成的。声明式的用户交互以及控制循环式的 Controller 使我们得以让整个控制行为变得无状态（Stateless），而无状态恰恰是 Kubernetes 保证可靠性的关键。<b>这里的无状态并不是指控制系统不存储状态或不依赖当前状态，而是指控制系统没有对过去状态的依赖，或不要求在时间上具有控制的“连续性”。</b>通俗地讲，这样的控制系统可以在任何时候重启，因为它仅仅针对当前状态与期望状态的差异行事，而不依赖之前发生了什么，也就是说它可以在任何时候崩溃，我们只需要保证系统能被及时重新启动。这种只依赖当前状态的 “无状态” 也可称为基于水平触发的（Level Driven），它的对立面边缘触发的（Edge Driven）或基于事件的。

![Level Driven](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0103.png)

上图选自 [*Programming Kubernetes by Michael Hausenblas, Stefan Schimanski*](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/) ，形象地描述了 Controller 这种 Level Driven 的思想。实际上水平触发虽然不依赖历史状态，但总是显得不如边缘触发那样有效率，因为我们一般必须使用轮询的方法以不断检测边缘，正如图中的 “get-compare-update” 过程，一个盲目的轮询会导致即便当前状态没有发生变化 Controller 也在不停运行，而不像由事件触发的处理程序那样精准地只在变化发生时运行。另外，Level Driven 的系统对于突发边缘的响应速度（最大延迟）取决于轮询的频率，而增大轮询的频率又会增大系统稳定时轮询的资源浪费。为此，<b>Kubernetes Controller 使用一些边缘触发的事件来触发只基于当前状态的控制流程，从而使整个控制流程具有 Level Driven 的可靠性和 Edge Driven 的快速响应</b>。我们通常称 Controller 是 Level Driven 的，其实这只是对它行为特征的描述，并非对实现原理的描述。

> If an API object appears with a marker value of true, you can't count on having seen it turn from false to true, only that you now observe it being true. Even an API watch suffers from this problem, so be sure that you're not counting on seeing a change unless your controller is also marking the information it last made the decision on in the object's status.

### Controller 一般结构

无论 Kubernetes 原生 Controller 或是可被自定义来扩展 Kubernetes 的 Custom Controller 都具有类似的结构。对于这个几乎全部 Controller 都采用的模式，[Writing Controllers](https://github.com/kubernetes/community/blob/8decfe4/contributors/devel/controllers.md#rough-structure) 一文中给出了对应的代码结构（Rough Structure）。另外，我个人认为此文中给出的 11 条 Guideline 对于理解 Controller 的设计和具体实现细节都是有巨大帮助的，推荐阅读。

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;target&quot;:&quot;blank&quot;,&quot;lightbox&quot;:false,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; modified=\&quot;2020-10-05T01:33:08.247Z\&quot; agent=\&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.68\&quot; etag=\&quot;lb98yqWE2qXIio0R3gga\&quot; version=\&quot;13.7.7\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;5JrG2Wsmwt4hK1JBAV3a\&quot; name=\&quot;Page-1\&quot;&gt;7Vpbk6I4FP41Pg4F4aaPre3OdWt6x6p15jEDUTKDxA2x1fn1m0CixKDSo6htDVZ1k0MSyHe+c8mBjjuYrd5SOE/+JjFKO8COVx33sQOAY9uA/xOSdSkJgrAUTCmOZaetYIR/ITVSShc4RrnWkRGSMjzXhRHJMhQxTQYpJUu924Sk+l3ncIoMwSiCqSkd45glpbQLwq38HcLTRN3ZCXrllRlUneVK8gTGZFkRucOOO6CEsPJsthqgVICncCnH/bXn6ubBKMpYkwG96MN6OU6+LD6sAzQev2H/hsM3cpZnmC7kgh+e3nPBCNFnROWDs7VCg8/IgeeN/jLBDI3mMBJXllz3XJawWcpbDj+F+bzUxgSvEH+AvrwPogyt9i7A2cDC+YTIDDG65l3kAE8CKZkUyuayohYpSioaUTIoiTDdzLvFip9IuF4AHTCgG5CMUZKmNcBRsshiAcSj3QC8FE8zfp6iCV9EX4CGOSMfpJiR+XkABa6nIeoGBqJuEJiQgtBrCVPXwPR9NiF0di5EI44Mn8vEdIbjWMx8Flh9HdYaojrBJZnqGaiahp3FD8JbCoxSmOc40gFEK8y+Vs6/CdwtX7YeV1INRWOtGtwa1l+rjcoo0dwOK1pqXPlwKDYc844C+ALIgkbouHtjkE4RO8Y7U6EVjfk1ClMyilLI8LP+uHValHd4IpgvZMMXt6c7NifcIUK5TDmq6uF3JtoMVBM5OxOVOBgTFaTaLPv3eeYbPPuEc/F8HRDAmTDJ8i+XjCGLEoOF3ObY7xouRTn+Bb8XUwkWzcUqi3X7/Y7/KOZaMJKXCYaYeoLTdEBSQot7u5PiOI8HcNwdRdjNYhVozQM4BtbCvkaySShLyJRkMB1upX3dw277fCIiABUY/kCMrWXSJuA9o9NwruY0gqZOI6wnQWN3cJJKA8PYxoT+5JJ/FohLTgqX+0zDsMYz2Iq767R6NeEyDC9oLCo8HoqXU47nfO/q5WZEeqPO5lFfkOsGOirdGlDU1qoKSgBaAkWFkgoow2dURJ93MItPznnPS7kDej0Y5XsHovz5Ie3dLaSKxRqHPYPCF4Ub3C+D5TRd3WVcGW6ztHBncDtAD1zguoCrnPMV5XjX2xjKQsPRHK/00SfkeMVQvrWG60oHuTfZuyP0HN1zOnZvhx7ljGfdvbnuH/405g9oWlkIrsEfv7vjmpywyp/j/UP7YH/Xr69YNO0PbK1/S3zuGgHIIHjbabyxcK9ZIaC9INGgFnhrYbhU5MG8PQRWCCo/14TUc6xAO1qrYZtlsFcLsapoWx7Qfq7OaivcOQyS35SCzNLJa1eQZ3W96q+n6ce3fP24bfWEhnpEZesV7BeOKCm0QB3Iartmudph7iZuSUkeMJT0hCNRfvz8/Yd48wvsj0isfcIBB/YXxGNphFOcTQ0tmkX/nFHyEyllZSRDL3oTUEcEnSotlv49X0+lQFBTuXNqQn57tX/zpeow+68sEmvayu9dNeqjDZW11qkGXFQ1vgH5BbdctgXUPutb5dLL9lzgBjftZZJ7rRcznpli730N+haxihHetfUB9SXJgfc8dV9FeK0ZX3BV49PqHWHTgse2xrEx2QvZntfQ+Paw4EK2Z+aOn+eIQma+EL1v66pLO9RnSSeGNt7cfrRYFmu2n366w/8B&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>

#### Informer

Informer 对 ETCD 中的 Object 变更（创建、删除、修改）进行监控。[Client-go](https://github.com/kubernetes/client-go) 库中实现了多种 Informer ，一般每种类型的资源对应同一个 Informer 实例。Informer 通过 API Server 为某种资源提供的 Watch 方法来监控变更，Watch 方法允许与 API Server 建立一个长连接，当数据库中有变更发生时，API Server 会发送变更的部分从而告知 Informer ，Informer 会将同样的变更应用到自己的本地副本（Local Cache）中。从而 Informer 可以维护一份包含某种资源的全部实例的本地副本，并时刻与 ETCD 同步。

Informer 的存在为 Controller 提供了两个功能：
- 只读地从它维护的本地副本（Local Cache）中 Get 或 List 对象，这会比每次都从 API Server 请求大量对象快得多；
- 通过向它注册 Event Handler ，Controller 也能捕获到数据库中的对象更改，并处理这些对象变更事件。<b>这些事件实际上就是上文所说的 Edge</b> ，见 [Efficient Detection of Changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)。

#### Event Handler

Event Handler 是捕获 Edge 并触发 Level Driven 处理过程的关键。它们其实就是数据库中数据变更事件的处理程序。一般，这些变更事件包括了 Kubernetes Controller 需要被触发的全部时机，因为<b>只有对象当前状态的变化才可能导致当前状态偏离期望状态，假如当前状态始终处于期望态且不发生变化，则该对象不需要处理。</b>

然而在现实情况中，我们并不能保证每个变更事件都被监控到并成功处理。假如 Watch 连接断开那么若干变更事件可能被错失，还有 Controller 本身的崩溃也会导致状态长时间偏离期望值。仅仅注册 Event Handler 使得我们的 Controller 又陷入 Edge Driven 。考虑到这些问题容易想到：<b>轮询是必须的，尽管可以以很低的频率进行。这意味着存在一个频率非常低的轮询，它每隔一段时间便检查当前状态是否是期望状态，若不是则触发控制循环调整当前状态；除此以外，被监控到的变更事件也会触发控制循环，从而保证大多数时候 Controller 都有低响应延迟，而且即便事件被错失 Controller 也能及时触发控制循环来纠正状态。</b>这一低频率轮询是通过 Informer 的 Periodical Resync 实现的。每隔一段时间 Informer 会与 API Server 同步本地缓存中的全部对象，并将检测到的差异转化为普通事件来调用 Event Handler 。即便一些对象没有发生变化，它们仍会被视为发生了 Update 事件；若某对象被发现出现在 ETCD 中却不存在于 Local Cache 中，说明该对象的创建事件被错失，Informer 会产生一个 Create 事件；反之若 Local Cache 中的某对象不存在于 ETCD ，Informer 会产生一个 `DeletedFinalStateUnkown` 事件以表明错失了 Delete 事件，且该对象被 Delete 时的最终状态未知。

> Watches and Informers will “sync”. Periodically, they will deliver every matching object in the cluster to your Update method. This is good for cases where you may need to take additional action on the object, but sometimes you know there won't be more work to do.

回归 Event Handler 的处理程序本身，它实际上过滤掉了不需要处理的情况（例如上面提到的对象本身没有发生变化，但由 Informer 在同步时产生的 Update 事件）；对于每个需要处理的情况，Informer 将实际状态可能偏离期望状态的 Object 的索引（Namespaced Name）放入工作队列。

> Many controllers must trigger off multiple resources (I need to "check X if Y changes"), but nearly all controllers can collapse those into a queue of “check this X” based on relationships. For instance, a ReplicaSet controller needs to react to a pod being deleted, but it does that by finding the related ReplicaSets and queuing those.

#### Work Queue 和 Worker

为了处理不同实例，Queue & Worker 结构被采用。Work Queue 中的 Item 是由 Event Handler 放入的 Object Key ，而每个 Worker 取出 Item 后，对该 Item 指向的 Object 执行一个 Control Loop（在某些 Controller 中将这些 Worker 称为 Reconciler）。Control Loop 是一个完全 Level Driven 的过程，在一次循环中它首先获取待处理对象的当前状态，将它与对象中指定的期望状态相比较，做出决策并采取行动使其收敛到期望状态。

```go
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}
```

值得注意的是，这里的 Work Queue 不同于普遍意义上的任务队列：

- Work Queue 中 Item 的意义在于触发 Worker 执行一个 Control Loop 以检查对应 Object 的最新状态，所以实际上同一 Object Key 在 Queue 中存在多个副本并无意义，某个 Object Key 是否需要被处理以及需要被处理的紧急程度（在队列中的排位）才有意义；
- Work Queue 需要保证同一个 Item 不会被多个 Worker 并行处理，多个 Control Loop 同时调整同一个 Object 会导致问题。

因此，Work Queue 将其中的 Item 分为三种状态：等待处理中、正处理中且无新请求、正在处理中且有新请求（Dirty）。当某个 Item 被试图添加到队列，若它本身不在队列中则将它直接添加到队列中，否则 Queue 会先检查它为哪一种 Item ：若为等待处理中，则队列中已存在副本，忽略本次入队请求；若为 “处理中且无新请求” ，则该 Object 正被处理，但本次处理不一定会兼顾最新发生的状态变化，因此将其状态标记为“正在处理中且有新请求”；若为 “正在处理中且有新请求” ，则忽略本次入队请求；当每个处理过程（即 Control Loop）结束后，Queue 会检查被处理的 Object 是否有新请求，若有则将这个 Object 重新加入队列并将其改为无新请求。

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;lightbox&quot;:false,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; modified=\&quot;2020-10-05T08:13:16.365Z\&quot; agent=\&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.68\&quot; etag=\&quot;TFWq5-f3_jAEZBapnYFW\&quot; version=\&quot;13.7.7\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;PWjo5SgMpzYFB_Sw65JJ\&quot; name=\&quot;Page-1\&quot;&gt;7Ztdc6I8FMc/jZd9hhBeL1fr7l7sztOZzmz7XEaIQBeJE2PV59NvkCBCiNIqLzrbm5JDEsjv5Jwk/9IRnCy23yhahj+Jj+ORrvnbEXwc6TrQNJ3/Si27zGIDMzMENPJFpcLwHP2P85bCuo58vCpVZITELFqWjR5JEuyxkg1RSjblanMSl5+6RAGWDM8eimXrS+SzMLM6ul3Yv+MoCPMnA8vN7ixQXlmMZBUin2yOTHA6ghNKCMuuFtsJjlN4OZes3VfF3cOLUZywJg22T9PX0Hsf/3p6eJu8fPemLvvxYIhxvKN4LUYs3pbtcgQBJeulqIYpw9s68GiWV9fkFwOH4fJ5gskCM7rjVfKOoGiyOQJsCVt4DNcURiScGhz6KsbNL8TQP4AB6F1gOOEBGY6IF0NmYzk1bOzW0MDzaDiZxMdpL9oIjjdhxPDzEnnp3Q1PC9wWsgV/6iPgl0qEZ1FlXpJR9cfGkNj8O3vj0wBchmgexfGExITu20IfYWfucfuKUfIbH92xPAfP5q1AFd248vxzO0VsKhBfOAuriE3s+EYdYkefQctqEzFw+mbsKBibV2WMAads1zF2LRuidhj3xtSVmL4Q+htTbrswO3wekmqtEd3YpjwRAeiSWr5q1lHrbdlpSk2s2DXR3DFEeX5hn29qRZFQFpKAJCieFtZxmWVR5wchS0HwDTO2Ezt0tGakzBdvI/Z6dP1f2tU/uimKj1vR9b6wywsJH/DrcSFrZubFotm+lLfLBpiO6jOu5GjImnq4wa6HIRrgk3nHrZ8cFMeIRe/l97u+q+Wg6M3V9r27+ujU1oOrDdVGiO9BrZi/93jGs6QVpFdPlHh4tYqSQL73GFG2u+q6Pnc87NVuT2eOaZiFE9tIvlbPeyfDUrgFnnbLNfnPsVXP37fdmdYNf/Xi16k7XHkDIbHuTM8QTEDNHO1Y37Dl5NEClhMeGa6+YcsBfFl0fj7WMi8NSN9wZHXw5vQNFdShHL4d+aB4c4fvM4z71pAc1RptXHcdbnMf9PE02i3iEwf33uQOFbPByB0OUFPrbRVqSm0ococzoDPwwOSOPAWcPQPnm6CzZ2Cn1zOwA4fj6oHJHS24WnHK6cjVKrnjTs7V57Ls0M7V8l+Bb+7PcE2R98ZY3g3I6S7xv6RftfCSFyM+5726Zf9k+lCE9dGozZpR57bG0S+e8EQi/uADZFMvz2ugV2hm+Uu0KoCe78itdJTlN6mjvWcOw77AWboiILiNHyu0r1ESrUIeBlUP8mnLyj4rz/WEJOmqheIoSFIvc7/x/SAcpzM+8lD8RdxYRL6/X9/qIqwcg1JeS3+ahtJprcuwK26oOYgArWY+Vf1+vSiSv8mQfNC1Agh1mUrHCqDbowJo1MMaigLoDkcBdIemALp38GmICupA1ClXJQDekjr14RDv9mOcfAUaljylgDYYeQpog9SnGmIbij4FalD9FajKm7WzqoXbVLUQod6XbAG0vxJVl87uVaMC2h18nHwunfatigCtwfb8CrLIh+B0I5ZAt7yUPRgVyE3FkmpHsNJPy1oJ0FTfEXAbvCex5DMRBo2Kl2vEgm4lFKCpTn039N1H07zWusDOi8U/+WUhVfyrJJz+AQ==&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>

> 所谓 “正在处理中且有新请求” ，意思是同一个 Key 需要被重新入队，但需要被延迟到本次处理完成，这也就保证 Control Loop 可再处理它一次来处理最新的状态变化，而之所以需要将入队延迟到本次处理完成，是为了防止同一 Item 被多个 Worker 并行处理。

至此，Controller 的一般结构基本解释清楚。在 Informer 、Work Queue 等模块组成的这一整体架构的支持下，某个特定功能的 Controller 只要明确这几个问题：

1. 在哪些时机（时间），Controller 需要对一个 Object 进行检查和可能的处理？对于只关注 Pod 数量的 ReplicaSet 来说，是 Pod 被创建和删除等等导致 Pod 数量发生变化的时机；对于关注 Pod Metrics 的 Horizontal Pod Autoscaler 来说，每隔一段固定时间都需要对每个 Object 进行检查，因为 Metrics 总是在变化，而并非像离散的数量值一般有明显的 Edge 。这些时机决定了我们将以怎样的方式（例如定义哪些 Event Handler）向 Work Queue 中添加 Item 。它们要保证高效，但比高效更重要的是覆盖所有的情况，从而让 Controller 正确运行。
2. 对于一个特定的 Object ，Controller 需要收集怎样的当前状态信息，以及如何使用这些状态信息来做出行为决策。涉及 Object 的全部信息一般都从 Local Cache 中获取以提升效率，有时我们也可能从外部获取一些其他信息。覆盖所有的情况且保证 Control Loop 始终是 Level Driven 的都很重要，这决定了 Control Loop 的逻辑是否有效、可靠。Control Loop 可以在一个周期内也可在多个周期内使当前状态收敛到期望状态，但要注意避免多个周期的重复操作以及非精确的操作引起的状态震荡。

将这些作以总结后，我将介绍一些 ReplicaSet 具体的实现细节以作为前文抽象内容的一个具体示例。

## ReplicaSet

RepicaSet 是通过一组字段来定义的，包括一个用来识别其所管理的 Pod 的集合的选择算符（Label Selector），一个用来标明应该维护的副本个数的数值（Replicas），一个用来指定应该创建新 Pod 以满足副本个数条件时要使用的 Pod 模板（Pod Template）等等。ReplicaSet 对象的完整结构可在 [`types.go`](https://github.com/kubernetes/kubernetes/blob/release-1.18/staging/src/k8s.io/api/apps/v1/types.go#L666) 查看。如其它对象一样，Spec 定义了用户对于某个 ReplicaSet 对象的期望状态，而 Status 为由 Controller 更新，显示了这个对象的当前状态。

### ReplicaSet Spec

- `replicas`：用户期望的副本数，默认为 1 。如果将全体被当前 ReplicaSet 所管理的 Pod 看作一个集合，Replicas 字段规定了集合的大小。ReplicaSet Controller 需要对这个集合的任何数量变化作出相应，以使其大小等于用户设定的 Replicas 。
- `minReadySeconds`：规定新创建的 Pod 在无容器发生错误的条件下，最少准备多少秒后才可被认为 Available ，默认为 0 。
- `selector`：即 Label Selector ，通过设定此字段规定具有怎样 Label 的 Pod 受到此 ReplicaSet 管理，即 Selector 字段定义了上文提到的 “集合” 。
- `template`：即 Pod Template ，定义了当集合中 Pod 数量不足时，ReplicaSet 创建 Pod 应当使用的对象模板。

### ReplicaSet Status

- `replicas`：当前被 ReplicaSet 管理的 Pod 的数量。
- `fullyLabeledReplicas`：当前被 ReplicaSet 管理的 Pod 中，具有与 Pod Template 完全相同的 Label 的 Pod 的数量。实际上被管理的 Pod 并不需要有与 Template 完全相同的 Label 也可能满足 Label Selector 的约束，这就是为什么这个字段的值可能与 replicas 不同。
- `readyReplicas`：当前被 ReplicaSet 管理的 Pod 中处于 Ready 状态的 Pod 的数量。
- `availableReplicas`：当前被 ReplicaSet 管理的 Pod 中被认为 Available 的 Pod 的数量。
- `observedGeneration`：
- `conditions`：

这里需要注意 “被管理” 、“处于 Ready 状态” 以及 “被认为 Available” 这几种表达方式之间的关系和细微不同，下面这张图片是一个很好的例子：

- **Owned Pod** ：被管理的 Pod（或 ReplicaSet 拥有的 Pod）即 ReplicaSet 中的 “Set” 中的 Pod 。当某个 Pod 与 Label Selector 相匹配它将立即被 ReplicaSet 接收，即被添加到集合中，当集合中的某个 Pod（通过修改 Label 等方式）不再匹配 Label Selector 它将立即被释放，即从集合中移出。某个 Pod 是否被另一个 ReplicaSet 的判断标准是它的 `ownerReference` 字段是否指向该 ReplicaSet。

> 选自 [垃圾收集 - 属主和附属](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents)：某些 Kubernetes 对象是其它一些对象的属主。例如，一个 ReplicaSet 是一组 Pod 的属主。具有属主的对象被称为是属主的附属 。每个附属对象具有一个指向其所属对象的 `metadata.ownerReferences` 字段。你也可以通过手动设置 `ownerReference` 的值，来指定属主和附属之间的关系。

- **Active Pod** ：Active 并非是 Pod 生命周期中的某个阶段，而是 Controller 根据 Pod 的 Phase 自行判断得出的。当某个 Pod 的 Phase 并非 Succeed 或 Failed ，则认为它是 Active 的，见函数 [`IsPodActive`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/controller_utils.go#L923) 。虽然 Label Selector 定义了被 ReplicaSet 管理的合法 Pod 集合，但集合中所有 Active 的 Pod 才真正被 ReplicaSet 所管理，非 Active 的 Pod 即便符合 Label Selector 的定义也会被 ReplicaSet 忽略。

> 关于 Pod 的状态和 Phase 之间的关系可参考 [Pod Lifecycle](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/) 。

- **Ready Pod** ：Ready 为一个 Pod 现有的 Condition 之一，因此 [`IsPodReady`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/api/pod/util.go#L224) 函数仅仅检查 Pod 是否处于该状态。当 Pod 中所有容器准备就绪后，即处于 Ready 状态。

- **Available Pod** ：Available 也是由 Controller 自行判断得出的一个状态，当某 Pod 已处于 Ready 状态的时间超过 `minReadySeconds` ，则认为它是 Available 的。设定 Available 的概念主要就是用于在 Status 中展示当前 Available 的 Pod 数量，因此不要将它与其他状态混淆。

## ReplicaSet Controller 实现细节

ReplicaSet Controller 的总体结构和各模块的职责与上面我们介绍的 Controller 一般结构类似，只是多出了 Expectations 模块。因此我们在这一小节中直接分为初始化和启动流程、Event Handlers 、Workers 和 Expectations 四部分来介绍具体的实现细节。

### 初始化和启动

构造函数 [NewReplicaSetController](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L112) 只是传递参数和初始化日志，其内部调用了 [NewBaseController](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L129) 是 ReplicaSet Controller 真正的构造函数。初始化分为三部分：

首先，初始化 `ReplicaSetController` 实例，包括将输入的 Client（`podControl` 、`kubeClient` 都是 Client-go 中 Interface 提供的调用 API Server 的接口，即对 REST API 的封装）传递给 Controller 、设置 `burstReplicas` 参数（该参数是控制平台启动时设置给 Controller Manager 的，Controller Manager 在初始化 Controller 时将启动参数传入）、初始两个内部模块 `expectations` 及 `queue` 。

```go
rsc := &ReplicaSetController{
	GroupVersionKind: gvk,
	kubeClient:       kubeClient,
	podControl:       podControl,
	burstReplicas:    burstReplicas,
	expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
	queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
}
```

第二，为 Informer 注册 Event Handler 。通过上文我们知道 Event Handler 即数据库中更改事件的回调函数。ReplicaSet Controller 使用了两个 Informer 监控 ReplicaSet 和 Pod 这两种资源。可以看到每个 EventHandler 提供了 `AddFunc` 、`UpdateFunc` 和 `DeleteFunc` 这三个事件。

```go
rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
	AddFunc:    rsc.addRS,
	UpdateFunc: rsc.updateRS,
	DeleteFunc: rsc.deleteRS,
})

podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
	AddFunc: rsc.addPod,
	// This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
	// overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
	// local storage, so it should be ok.
	UpdateFunc: rsc.updatePod,
	DeleteFunc: rsc.deletePod,
})
```

第三，从 Informer 中获取 Lister 和表示 Lister 是否同步完成的函数，仅仅是记录在成员中便于使用。

```go
rsc.rsLister = rsInformer.Lister()
rsc.rsListerSynced = rsInformer.Informer().HasSynced

rsc.podLister = podInformer.Lister()
rsc.podListerSynced = podInformer.Informer().HasSynced
```

Controller Manager 在启动时会在单独的 Go Routine 调用 Controller 的 `Run` 方法：

```go
func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)
	klog.Infof("Starting %v controller", controllerName)
	defer klog.Infof("Shutting down %v controller", controllerName)
```

`Run` 的逻辑也比较简单，首先等待 Informer 中的 Local Cache 首次与 ETCD 同步完成：

```go
	if !cache.WaitForNamedCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}
```

接着使用 `wait.Until` 每秒创建一定数量的 `worker` 处理 Work Queue 中的对象索引。

```go
	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

### Event Handlers

Event Handler 是将 Edge Driven 转化为 Level Driven 的关键，也是触发 Worker 中的 Control Loop 完成 “Check this X” 的关键。这些 Control Loop 何时针对哪个 ReplicaSet 实例执行，取决于 Event Handler 何时将哪个 ReplicaSet 的 Key 放入工作队列，通过这种方式 Event Handler 将某个 Worker “唤醒” 开始针对目标 ReplicaSet 进行控制，也可理解为 Event Handler 将被放入队列的 ReplicaSet “唤醒” 以进行其控制工作。

#### Add ReplicaSet

[`addRS`](https://github.com/kubernetes/kubernetes/blob/bbbab14216ee2256079da2ced5f52f91d08f5d6d/pkg/controller/replicaset/replica_set.go#L284) 在有 ReplicaSet 被创建（Informer 发现从前未出现过的 ReplicaSet）时被调用。它仅仅输出了日志并将被创建的 ReplicaSet 的 Key 直接加入队列，因为新出现的 ReplicaSet 显然有可能处于非期望状态。

```go
func (rsc *ReplicaSetController) addRS(obj interface{}) {
	rs := obj.(*apps.ReplicaSet)
	klog.V(4).Infof("Adding %s %s/%s", rsc.Kind, rs.Namespace, rs.Name)
	rsc.enqueueRS(rs)
}
```

#### Update ReplicaSet

修改 ReplicaSet 的许多字段都会导致这个对象变得需要被处理，如改变 Label Selector 使其管理不同的 Pod 集合、改变 Replicas 使其现有的 Pod 数量不再满足期望等。 [`updateRS`](https://github.com/kubernetes/kubernetes/blob/bbbab14216ee2256079da2ced5f52f91d08f5d6d/pkg/controller/replicaset/replica_set.go#L291) 可能在几种情况下被调用：

1. 某 ReplicaSet 被使用 Update 或 Patch 方法更改，触发 Update 事件；
1. 在 Periodic Resync 的过程中，所有 ReplicaSet 都会被触发 Update 事件；
1. 某个旧 ReplicaSet 被删除（删除之前可能被进行若干次修改），后一个新的有同样 Namespaced Name 的 ReplicaSet 被创建出来，如果删除事件被 Informer 错失的话，它是无法区分新旧 ReplicaSet 的，因此它认为发生了一次 Update；

```go
func (rsc *ReplicaSetController) updateRS(old, cur interface{}) {
	oldRS := old.(*apps.ReplicaSet)
	curRS := cur.(*apps.ReplicaSet)
```

首先处理第三种情况，通过检查每个 Object 唯一的 [UID](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#uids) 就可判断出 `oldRS` 和 `curRS` 是不是同一个对象。对于这种情况，产生一个 `DeletedFinalStateUnknown` 给删除事件的回调 `deleteRS` 。前文已经提到过，`DeletedFinalStateUnknown` 用于表示某个 Object 被删除了，但被删除时的最后状态未知。

> Every object created over the whole lifetime of a Kubernetes cluster has a distinct UID. It is intended to distinguish between historical occurrences of similar entities.

```go
	// TODO: make a KEP and fix informers to always call the delete event handler on re-create
	if curRS.UID != oldRS.UID {
		key, err := controller.KeyFunc(oldRS)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", oldRS, err))
			return
		}
		rsc.deleteRS(cache.DeletedFinalStateUnknown{
			Key: key,
			Obj: oldRS,
		})
	}
```

直接将新的 ReplicaSet 的 Key 进队，这里 “新的” Key 的说法其实只适用于第三种情况，其他情况 Key 应当是不变的。情况一和情况二中 ReplicaSet 都会被直接进队，这里的注释也就是在解释为什么没有过滤掉情况二那些实际上没发生改变但是被触发的 ReplicaSet ，大意就是每隔一段时间就让所有 ReplicaSet 进队一次是更安全的选择，可以防止由于创建或删除 Pod 失败导致对应的 ReplicaSet 的 Update 永远不会被任何事触发（ReplicaSet 创建或删除 Pod 的过程不是阻塞的，稍后我们会提到），且有许多机制例如 Expectations 可以降低这种大量入队处理的开销。

```go
	// You might imagine that we only really need to enqueue the
	// replica set when Spec changes, but it is safer to sync any
	// time this function is triggered. That way a full informer
	// resync can requeue any replica set that don't yet have pods
	// but whose last attempts at creating a pod have failed (since
	// we don't block on creation of pods) instead of those
	// replica sets stalling indefinitely. Enqueueing every time
	// does result in some spurious syncs (like when Status.Replica
	// is updated and the watch notification from it retriggers
	// this function), but in general extra resyncs shouldn't be
	// that bad as ReplicaSets that haven't met expectations yet won't
	// sync, and all the listing is done using local stores.
	if *(oldRS.Spec.Replicas) != *(curRS.Spec.Replicas) {
		klog.V(4).Infof("%v %v updated. Desired pod count change: %d->%d", rsc.Kind, curRS.Name, *(oldRS.Spec.Replicas), *(curRS.Spec.Replicas))
	}
	rsc.enqueueRS(curRS)
}
```

#### Delete ReplicaSet

[`deleteRS`](https://github.com/kubernetes/kubernetes/blob/bbbab14216ee2256079da2ced5f52f91d08f5d6d/pkg/controller/replicaset/replica_set.go#L326) 的触发有两种情况，即 API Server 告知 Informer 有 Object 被删除或 Informer 自行产生的 `DeletedFinalStateUnknown` 。这里只是简单的对参数的类型进行了判断，如果传入的 `obj` 是一个 `DeletedFinalStateUnknown` 那么从中取出真正的 ReplicaSet 进行后面的处理。实际上处理也仅仅是将对应的 ReplicaSet 唤醒。

```go
func (rsc *ReplicaSetController) deleteRS(obj interface{}) {
	rs, ok := obj.(*apps.ReplicaSet)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
			return
		}
		rs, ok = tombstone.Obj.(*apps.ReplicaSet)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a ReplicaSet %#v", obj))
			return
		}
	}

	key, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}

	klog.V(4).Infof("Deleting %s %q", rsc.Kind, key)

	// Delete expectations for the ReplicaSet so if we create a new one with the same name it starts clean
	rsc.expectations.DeleteExpectations(key)

	rsc.queue.Add(key)
}
```

值得注意的是这里第一次出现了 Expectations 的概念，我们暂时忽略这些对于 Expectations 的调用，在 Control Loop 中再来讨论它的作用。

#### Add Pod

对于 Pod 的创建、删除和修改可能带来什么？当一个 Pod 被创建或删除，如果它属于某一个 ReplicaSet 的管辖，那么该 ReplicaSet 就会因为 Pod 数量发生改变而偏离期望状态。当某个 Pod 自身被修改，它可能会由于 Label 的改变而离开原本的 ReplicaSet 而被新的 ReplicaSet 管理，也可能会因为状态的变更（原本 Active 的 Pod 不再 Active 等）而导致它所在的 ReplicaSet 偏离期望状态。

[`addPod`](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L356) 会在某个新 Pod 出现在 ETCD 中时被调用，此时

1. 要么 Pod 被 Controller 中的某个 Worker 创建成功，那么该 Pod 应该原本具有一个指向某 ReplicaSet 的 Owner Reference 。（在 Kubernetes 源码中许多表示 Owner Reference 的函数和变量被命名为 Controller Reference ，但这种名称容易与 Controller 混淆而造成迷惑，所以统一称为 Owner Reference）
1. 要么 Pod 被第三方（用户或其他 Controller）创建，那么该 Pod 可能没有 Owner ，也可能属于某个 ReplicaSet 或属于某个其他类型的 Object 。
1. 要么 Pod 并非刚刚被创建，只是刚刚被 Informer “发现” ，这可能由于许多原因，例如 Controller 刚刚被启动或若干事件被 Watch 错失。此时 Pod 可能处于生命周期的任何阶段。

对于已经被设置 Owner Reference 的 Pod ，除了其 Owner 本身外其他 ReplicaSet 即便拥有匹配的 Label Selector 也不会将其纳入管理，直到 Pod 被 Owner 释放即 Owner Reference 被清除后其他 ReplicaSet 才应当考虑接受并开始管理该 Pod 。因此对于这些 Pod ，`addPod` 仅将它们 Owner Reference 记录的 ReplicaSet（如果它的 Owner 并非 ReplicaSet ，则直接退出）唤醒，如果该 Pod 不再匹配原有 Owner 的 Label Selector ，那么它自然会被 Control Loop 释放。

```go
	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
		if rs == nil {
			return
		}
		rsKey, err := controller.KeyFunc(rs)
		if err != nil {
			return
		}
		klog.V(4).Infof("Pod %s created: %#v.", pod.Name, pod)
		rsc.expectations.CreationObserved(rsKey)
		rsc.queue.Add(rsKey)
		return
	}
```

对于没有 Owner 的 Pod ，`addPod` 会将同一命名空间中，所有 ReplicaSet 中具备与当前 Pod 匹配的 Label Selector 的 ReplicaSet 全部唤醒。道理上看它们都应当管理该 Pod ，但每个 Pod 不能由多个 ReplicaSet 同时管理，具体由哪个 ReplicaSet 接收实际上是随机的，取决于哪个 ReplicaSet 的 Key 会先被 Worker 取出并抢先设置 Owner Reference 。另外，ReplicaSet 只可能管理同一命名空间中的 Pod 这一规则在这里得以体现。

```go
	// Otherwise, it's an orphan. Get a list of all matching ReplicaSets and sync
	// them to see if anyone wants to adopt it.
	// DO NOT observe creation because no controller should be waiting for an
	// orphan.
	rss := rsc.getPodReplicaSets(pod)
	if len(rss) == 0 {
		return
	}
	klog.V(4).Infof("Orphan Pod %s created: %#v.", pod.Name, pod)
	for _, rs := range rss {
		rsc.enqueueRS(rs)
	}
```

#### Update Pod

[`updatePod`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L403) 的核心问题在于如何处理 Label 和 Owner Reference 的 “组合变更” ，这两个字段都允许被外部更改，因此必须对这种复杂的情况加以分析。不过在此之前，它首先过滤掉了 Periodical Resync 产生的大量事件，通过判断 `ResourceVersion` 可以做到这一点，具体原因可参考 [Efficient detection of changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes) 。

```go
func (rsc *ReplicaSetController) updatePod(old, cur interface{}) {
	curPod := cur.(*v1.Pod)
	oldPod := old.(*v1.Pod)
	if curPod.ResourceVersion == oldPod.ResourceVersion {
		// Periodic resync will send update events for all known pods.
		// Two different versions of the same pod will always have different RVs.
		return
	}
```

> ReplicaSet 的 Update Event Handler 没有过滤掉这些事件换取了更可靠的 Controller ，而 Pod 无需这样做，因为 Update Pod 的最终目的也是将 ReplicaSet 入队以表达 “Check this X” 之意，无需定期触发 Update Pod ，因为 Update ReplicaSet 也能起到相同的作用。

对于 Label 和 Owner Reference 的处理思路是这样的：如果 Owner Reference 变化了，这可能表明 Pod 被从它旧的属主那里主动地释放或被动地剥夺了，我们无法区分这两种情况，必须唤醒旧的属主（假如有）以告知它这一事件。类似地，如果 Pod 被指定了一个新的属主，我们同样需要唤醒新属主，并且在这种情况下，Label 的不需要再被考虑，因为已有属主的 Pod 不会被其他 ReplicaSet 管理。

```go
	curControllerRef := metav1.GetControllerOf(curPod)
	oldControllerRef := metav1.GetControllerOf(oldPod)
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	if controllerRefChanged && oldControllerRef != nil {
		// The ControllerRef was changed. Sync the old controller, if any.
		if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
			rsc.enqueueRS(rs)
		}
	}

	// If it has a ControllerRef, that's all that matters.
	if curControllerRef != nil {
		rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
		if rs == nil {
			return
		}
		klog.V(4).Infof("Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
		rsc.enqueueRS(rs)
		// TODO: MinReadySeconds in the Pod will generate an Available condition to be added in
		// the Pod status which in turn will trigger a requeue of the owning replica set thus
		// having its status updated with the newly available replica. For now, we can fake the
		// update by resyncing the controller MinReadySeconds after the it is requeued because
		// a Pod transitioned to Ready.
		// Note that this still suffers from #29229, we are just moving the problem one level
		// "closer" to kubelet (from the deployment to the replica set controller).
		if !podutil.IsPodReady(oldPod) && podutil.IsPodReady(curPod) && rs.Spec.MinReadySeconds > 0 {
			klog.V(2).Infof("%v %q will be enqueued after %ds for availability check", rsc.Kind, rs.Name, rs.Spec.MinReadySeconds)
			// Add a second to avoid milliseconds skew in AddAfter.
			// See https://github.com/kubernetes/kubernetes/issues/39785#issuecomment-279959133 for more info.
			rsc.enqueueRSAfter(rs, (time.Duration(rs.Spec.MinReadySeconds)*time.Second)+time.Second)
		}
		return
	}
```

若 Pod 没有被指定属主，我们需要考虑 Label 的变化，即像 `addPod` 中所作的一样唤醒所有可能作为属主的 ReplicaSet 。

```go
	// Otherwise, it's an orphan. If anything changed, sync matching controllers
	// to see if anyone wants to adopt it now.
	if labelChanged || controllerRefChanged {
		rss := rsc.getPodReplicaSets(curPod)
		if len(rss) == 0 {
			return
		}
		klog.V(4).Infof("Orphan Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
		for _, rs := range rss {
			rsc.enqueueRS(rs)
		}
	}
```

[`deletePod`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L477) 的工作则是直接唤醒被删除的 Pod 的属主。考虑到 Event Handler 的大部分细节已在前面阐述，这里就不占用篇幅了。对于 Pod Event Handler ，只需注意 Pod 若已拥有 Owner ，必须先将此 Owner 唤醒，结合其详细的注释就比较好理解了。

### Worker

Worker 并行运行 Control Loop 。启动方法 `Run` 中每隔一秒会创建出一些运行 `worker` 函数的 Go Routine ，证明它们并不是一些长期运行的 Routine ，不然也不会需要频繁创建。的确如此，[`worker`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L518) 只有三行：

```go
// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (rsc *ReplicaSetController) worker() {
	for rsc.processNextWorkItem() {
	}
}
```

#### Process Next Work Item

看上去当 `processNextWorkItem` 返回 `false` 时 `worker` 就会退出，那么 [`processNextWorkItem`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L523) 就应当是从 Work Queue 中取出单个 Item 并处理的逻辑：

```go
func (rsc *ReplicaSetController) processNextWorkItem() bool {
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

	err := rsc.syncHandler(key.(string))
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}
```

为了让这段代码变得清晰这里稍微解释一下 `queue` 的接口：

- `Get` 自然是从 Queue 中 Pop 一个 Item ，该 Item 被取出的同时会在队列中标记为 “正在处理” ，如果队列为空会返回 `quit` 为真；
- `Done` 表示告知队列某个 Item 已经成功处理完成；
- `AddRateLimited` 和 `Forget` 是 `RateLimitedQueue` 所实现的 `RateLimitingInterface` 中的方法。`RateLimitedQueue` 可对元素的入队次数进行统计，并阻止某一元素次数过多，这里用来限制单个对象的重试次数。
	- `AddRateLimited` 为在现有重试次数允许的情况下将 Item 加入队列，若 Item 已被重试过多次，则进队请求被直接忽略；
	- `Forget` 将对应 Item 的 重试次数清空，在 Item 处理成功后被调用；

因此 `processNextWorkItem` 仍是对 Control Loop 的一个封装：它获取一个新的 Item ，通过 `syncHandler` 对其进行处理，若处理过程中发生错误，则尝试将该 Item 重新加入队列进行重试，重试次数过多的对象将被忽略。只有队列为空时，`processNextWorkItem` 会返回 `false` 来终止 `worker` 。`syncHandler` 的逻辑则是真正的 Control Loop ，它其实是函数 `syncReplicaSet` 。

#### Sync ReplicaSet

[`syncReplicaSet`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L650) 即 Control Loop 。在每个 Loop 中，Controller 获取目标 ReplicaSet 的 Current State 和 Desired State ，并作出具体的行为完成 ReplicaSet 所承诺的功能。

在函数的开始，根据本次 Control Loop 的目标 ReplicaSet 的 Key 从 Local Cache 中拿到目标 ReplicaSet 的最新 Object 。

```go
// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return err
	}
```

> Control Loop 总是在开始时读取目标 Object 的一切 “最新” 状态，这是因为 Control Loop 必须保证其 Edge Driven 特性，也就是说当某些 ReplicaSet 对象被入队，那可能是由于某个原本属于它的 Pod 被删除或它本身的 Label Selector 被更改等，但这些对 Control Loop 是透明的，Control Loop 必须通过获取对象现有状态，观察现有状态与期望状态的差异来决定自己的行为。这也解释了为什么 Work Queue 中仅仅存储 Object Key 而不包含其他信息，以及为什么需要维护 Local Cache 来避免 Control Loop 大量的状态读取给 API Server 造成负担（尽管如此，在某些需要获取最最最新对象状态的场景下还是需要单独从 API Server 进行请求的，Local Cache 与 ETCD 会有一些不同步）。

按照 Control Loop 的职责，我们需要获取目标 ReplicaSet 的当前状态和期望状态。什么叫做一个 ReplicaSet 的当前状态和期望状态？这是根据该 Object 所承诺的功能决定的：

##### 维护正确的 Pod 集合

首先，ReplicaSet 总会按照设定给它的 Label Selector 来接收或释放 Pod ，即正确设定相关 Pod 的 Owner Reference ，且当前被管理的各种类型的 Pod 数量总会被正确设定到 Status 中的字段上；<b>此时，一个 ReplicaSet 当前状态与期望状态之间的差异体现在由 Label Selector 定义的应当被它管理的 Pod 集合与实际上拥有指向它的 Owner Reference 的 Pod 集合的差异。</b>因此 Controller 从 Local Cache 中得到一份处于相同 Namespace 的全部 Pod 的列表，并从中筛选出目标 ReplicaSet 应当管理的 Pod 集合。

```go
selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
if err != nil {
	utilruntime.HandleError(fmt.Errorf("error converting pod selector to selector: %v", err))
	return nil
}

// list all pods to include the pods that don't match the rs`s selector
// anymore but has the stale controller ref.
// TODO: Do the List and Filter in a single pass, or use an index.
allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
if err != nil {
	return err
}
// Ignore inactive pods.
filteredPods := controller.FilterActivePods(allPods)
```

使用 [`claimPods`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L717)（内部其实使用了 `PodControllerRefManager` ，具体的实现还是比较易懂的）即可纠正现有集合：对于不该被管理的 Pod 清除 Owner Reference ，对于该管理但未管理的 Pod 添加 Owner Reference 即可。

```go
// NOTE: filteredPods are pointing to objects from cache - if you need to
// modify them, you need to copy it first.
filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
if err != nil {
	return err
}
```

##### 保证 Pod 集合的大小

其次，ReplicaSet 总会创建或删除 Pod 保证被管理的 Pod 总数等于 Replicas（尽管创建或删除之前或过程中 Pod 数量可能暂时地不满足期望，我们都知道不可能做到 Pod 数量恒定于期望值）；<b>此时，一个 ReplicaSet 当前状态与期望状态之间的差异体现在被它管理的 Pod 数量与 Replicas 之间的差异。</b>

```go
var manageReplicasErr error
if rsNeedsSync && rs.DeletionTimestamp == nil {
	manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
}
```

[`manageReplicas`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L545) 用于处理这种差异。但在此之前，注意到 `rsNeedsSync` 控制了是否调用 `manageReplicas` ，这是一个与 Expectations 模块有关的标志，在此我们仅知道它控制了是否对 Pod 数量进行调整，在后文中介绍 Expectations 时再加以解释。

`manageReplicas` 对 Pod 数量与期望之间的偏差进行计算：

```go
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
```

当 `diff` 为负数时代表被管理的 Pod 数量不足，需要创建新 Pod 。首先，单次 Control Loop 中创建和删除 Pod 的真实数量都需要由参数 `burstReplicas` 限制，该参数为 Controller 的启动参数，是初始化 Controller 时传入的。

```go
if diff < 0 {
	diff *= -1
	if diff > rsc.burstReplicas {
		diff = rsc.burstReplicas
	}
```

接着，Controller 使用一种称作批量启动（Batch Start）的方法创建一定数量的 Pod 。批量启动根据 Pod Template 并行创建大量相同的 Pod ，它将需要被创建的 Pod 数量分为若干个 Batch ，这些 Batch 中包含的 Pod 数量由小到大各不相同，由 `SlowStartInitialBatchSize` 控制最小的 Batch 的大小，而后 Batch 大小逐个倍增。在创建 Pod 时，Controller 按照由小到大的顺序串行创建每个 Batch ，而单个 Batch 之内是并行创建的。Batch Start 本质上是先使用小的 Batch 尝试创建 Pod 这一行为是否会出错，当某个 Batch 创建出错时，Controller 会取消后面更大 Batch 的创建，这样做的好处即避免了大量并行创建 Pod 的请求同时出错，而导致大量请求夹杂着错误信息淹没 API Server 。

```go
// Batch the pod creates. Batch sizes start at SlowStartInitialBatchSize
// and double with each successful iteration in a kind of "slow start".
// This handles attempts to start large numbers of pods that would
// likely all fail with the same error. For example a project with a
// low quota that attempts to create a large number of pods will be
// prevented from spamming the API service with the pod create requests
// after one of its pods fails.  Conveniently, this also prevents the
// event spam that those failures would generate.
successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
	err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
	if err != nil {
		if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
			// if the namespace is being terminated, we don't have to do
			// anything because any creation will fail
			return nil
		}
	}
	return err
})
```

对于 `diff` 为正的情况，Controller 需要在现有被管理 Pod 中删除一些，这个行为同样需要受到 `burstReplicas` 的限制。

```go
} else if diff > 0 {
	if diff > rsc.burstReplicas {
		diff = rsc.burstReplicas
	}
```

通过 [`getPodsToDelete`](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/controller/replicaset/replica_set.go#L804) 方法，Controller 得以从被管理 Pod 中选出一些以在删除时造成较小的 “损失” 。

```go
relatedPods, err := rsc.getIndirectlyRelatedPods(rs)
utilruntime.HandleError(err)

// Choose which Pods to delete, preferring those in earlier phases of startup.
podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)
```

通过阅读其中更深的代码，可以发现 `getPodsToDelete` 是通过对 [`ActivePodsWithRanks`](https://github.com/kubernetes/kubernetes/blob/94833ccdf29146c4908ed1b891a87f2510684ed1/pkg/controller/controller_utils.go#L815) 进行排序，并取排序靠前的 Pod 。排序的比较关系为此处的 [`Less`](https://github.com/kubernetes/kubernetes/blob/94833ccdf29146c4908ed1b891a87f2510684ed1/pkg/controller/controller_utils.go#L836) 方法。

而后，通过开启多个 Go Routine 并行删除选定的 Pod 。

```go
var wg sync.WaitGroup
wg.Add(diff)
for _, pod := range podsToDelete {
	go func(targetPod *v1.Pod) {
		defer wg.Done()
		if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
			// Decrement the expected number of deletes because the informer won't observe this deletion
			podKey := controller.PodKey(targetPod)
			rsc.expectations.DeletionObserved(rsKey, podKey)
			if !apierrors.IsNotFound(err) {
				klog.V(2).Infof("Failed to delete %v, decremented expectations for %v %s/%s", podKey, rsc.Kind, rs.Namespace, rs.Name)
				errCh <- err
			}
		}
	}(pod)
}
wg.Wait()
```
