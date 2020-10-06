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

Kubernetes Controller 是监视集群状态的控制循环，在每个循环中根据需要做出或请求更改。每个 Controller 监控至少一个 Kubernetes 资源类型，负责其全部对象实例的正常功能，试图做出行为将当前的集群状态移动到更接近期望的状态。这些对象一般具有表示所需状态的 Spec 字段和表示当前状态的 Status 字段。Spec 是让用户写入期望的状态，系统可以通过 Spec 读出用户的期望。Status 是系统写入观察到的状态，用户可以从中读出系统当前是什么状态。Controller 用以维护集群的这些“行为”绝大多数通过调用 API Server 得以完成，且大多数同样是基于声明式系统的。举例来说，ReplicaSet 是一个负责调整 Container 数量的 Controller ，但它在需要创建和删除 Container 时仅仅需要在数据库中创建或删除 Pod 对象即可，Pod Controller 会立即关注到这些改动并实际创建出所需的 Container 以完成它的工作。在这个过程中 API Server 提供了在数据库中创建、删除和修改对象的统一接口，但实际工作是由 Controller 完成的。实际上这种通过 Object 的交互是 Controller 之间协作的普遍方式（如 Deployment 与 ReplicaSet 的协作）。

### Controller Manager

从逻辑上讲，每个 Controller 都是一个单独的进程，但为了降低复杂性，它们被编译成同一个二进制文件，并在单个进程中运行（即 Kube Controller Manager）。Controller Manager 初始化共享资源 [Controller Context](https://github.com/kubernetes/kubernetes/blob/release-1.18/cmd/kube-controller-manager/app/controllermanager.go#L289) ，包括 Informer Factory 、Client Factory 、Cloud Provider Interface 、Common Configuration 等，并并行运行所有 Controller 。在函数 [`NewControllerInitializers`](https://github.com/kubernetes/kubernetes/blob/release-1.18/cmd/kube-controller-manager/app/controllermanager.go#L372) 你可以看到 Controller 的入口被初始化，在 [`Run`](https://github.com/kubernetes/kubernetes/blob/release-1.18/cmd/kube-controller-manager/app/controllermanager.go#L159) 中它们被并行启动。多个 Controller Manager 实例之间通过 Leader Election 的方式协作以保证其可用性，这里我们不多赘述 Controller Manager 的细节。

![Kubernetes 控制平台](https://d33wubrfki0l68.cloudfront.net/7016517375d10c702489167e704dcb99e570df85/7bb53/images/docs/components-of-kubernetes.png)

### Controller 设计思想

总结起来，Controller 的工作即监控用户定义的对象并在 Status 与 Spec 不一致时采取相应的行动，这种“监控”是以 Control Loop 的形式完成的。声明式的用户交互以及控制循环式的 Controller 使我们得以让整个控制行为变得无状态（Stateless），而无状态恰恰是 Kubernetes 保证可靠性的关键。<b>这里的无状态并不是指控制系统不存储状态或不依赖当前状态，而是指控制系统没有对过去状态的依赖，或不要求在时间上具有控制的“连续性”。</b>通俗地讲，这样的控制系统可以在任何时候重启，因为它仅仅针对当前状态与期望状态的差异行事，而不依赖之前发生了什么，也就是说它可以在任何时候崩溃，我们只需要保证系统能被及时重新启动。这种只依赖当前状态的“无状态”也可称为基于水平触发的（Level Driven），它的对立面边缘触发的（Edge Driven）或基于事件的。

![Level Driven](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0103.png)

上图选自 [*Programming Kubernetes by Michael Hausenblas, Stefan Schimanski*](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/) ，形象地描述了 Controller 这种 Level Driven 的思想。实际上水平触发虽然不依赖历史状态，但总是显得不如边缘触发那样有效率，因为我们一般必须使用轮询的方法以不断检测边缘，正如图中的“get-compare-update”过程，一个盲目的轮询会导致即便当前状态没有发生变化 Controller 也在不停运行，而不像由事件触发的处理程序那样精准地只在变化发生时运行。另外，Level Driven 的系统对于突发边缘的响应速度（最大延迟）取决于轮询的频率，而增大轮询的频率又会增大系统稳定时轮询的资源浪费。为此，<b>Kubernetes Controller 使用一些边缘触发的事件来触发只基于当前状态的控制流程，从而使整个控制流程具有 Level Driven 的可靠性和 Edge Driven 的快速响应</b>。我们通常称 Controller 是 Level Driven 的，其实这只是对它行为特征的描述，并非对实现原理的描述。

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

因此，Work Queue 将其中的 Item 分为三种状态：等待处理中、正处理中且无新请求、正在处理中且有新请求（Dirty）。当某个 Item 被试图添加到队列，若它本身不在队列中则将它直接添加到队列中，否则 Queue 会先检查它为哪一种 Item ：若为等待处理中，则队列中已存在副本，忽略本次入队请求；若为“处理中且无新请求”，则该 Object 正被处理，但本次处理不一定会兼顾最新发生的状态变化，因此将其状态标记为“正在处理中且有新请求”；若为“正在处理中且有新请求”，则忽略本次入队请求；当每个处理过程（即 Control Loop）结束后，Queue 会检查被处理的 Object 是否有新请求，若有则将这个 Object 重新加入队列并将其改为无新请求。

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;lightbox&quot;:false,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; modified=\&quot;2020-10-05T08:13:16.365Z\&quot; agent=\&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.68\&quot; etag=\&quot;TFWq5-f3_jAEZBapnYFW\&quot; version=\&quot;13.7.7\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;PWjo5SgMpzYFB_Sw65JJ\&quot; name=\&quot;Page-1\&quot;&gt;7Ztdc6I8FMc/jZd9hhBeL1fr7l7sztOZzmz7XEaIQBeJE2PV59NvkCBCiNIqLzrbm5JDEsjv5Jwk/9IRnCy23yhahj+Jj+ORrvnbEXwc6TrQNJ3/Si27zGIDMzMENPJFpcLwHP2P85bCuo58vCpVZITELFqWjR5JEuyxkg1RSjblanMSl5+6RAGWDM8eimXrS+SzMLM6ul3Yv+MoCPMnA8vN7ixQXlmMZBUin2yOTHA6ghNKCMuuFtsJjlN4OZes3VfF3cOLUZywJg22T9PX0Hsf/3p6eJu8fPemLvvxYIhxvKN4LUYs3pbtcgQBJeulqIYpw9s68GiWV9fkFwOH4fJ5gskCM7rjVfKOoGiyOQJsCVt4DNcURiScGhz6KsbNL8TQP4AB6F1gOOEBGY6IF0NmYzk1bOzW0MDzaDiZxMdpL9oIjjdhxPDzEnnp3Q1PC9wWsgV/6iPgl0qEZ1FlXpJR9cfGkNj8O3vj0wBchmgexfGExITu20IfYWfucfuKUfIbH92xPAfP5q1AFd248vxzO0VsKhBfOAuriE3s+EYdYkefQctqEzFw+mbsKBibV2WMAads1zF2LRuidhj3xtSVmL4Q+htTbrswO3wekmqtEd3YpjwRAeiSWr5q1lHrbdlpSk2s2DXR3DFEeX5hn29qRZFQFpKAJCieFtZxmWVR5wchS0HwDTO2Ezt0tGakzBdvI/Z6dP1f2tU/uimKj1vR9b6wywsJH/DrcSFrZubFotm+lLfLBpiO6jOu5GjImnq4wa6HIRrgk3nHrZ8cFMeIRe/l97u+q+Wg6M3V9r27+ujU1oOrDdVGiO9BrZi/93jGs6QVpFdPlHh4tYqSQL73GFG2u+q6Pnc87NVuT2eOaZiFE9tIvlbPeyfDUrgFnnbLNfnPsVXP37fdmdYNf/Xi16k7XHkDIbHuTM8QTEDNHO1Y37Dl5NEClhMeGa6+YcsBfFl0fj7WMi8NSN9wZHXw5vQNFdShHL4d+aB4c4fvM4z71pAc1RptXHcdbnMf9PE02i3iEwf33uQOFbPByB0OUFPrbRVqSm0ococzoDPwwOSOPAWcPQPnm6CzZ2Cn1zOwA4fj6oHJHS24WnHK6cjVKrnjTs7V57Ls0M7V8l+Bb+7PcE2R98ZY3g3I6S7xv6RftfCSFyM+5726Zf9k+lCE9dGozZpR57bG0S+e8EQi/uADZFMvz2ugV2hm+Uu0KoCe78itdJTlN6mjvWcOw77AWboiILiNHyu0r1ESrUIeBlUP8mnLyj4rz/WEJOmqheIoSFIvc7/x/SAcpzM+8lD8RdxYRL6/X9/qIqwcg1JeS3+ahtJprcuwK26oOYgArWY+Vf1+vSiSv8mQfNC1Agh1mUrHCqDbowJo1MMaigLoDkcBdIemALp38GmICupA1ClXJQDekjr14RDv9mOcfAUaljylgDYYeQpog9SnGmIbij4FalD9FajKm7WzqoXbVLUQod6XbAG0vxJVl87uVaMC2h18nHwunfatigCtwfb8CrLIh+B0I5ZAt7yUPRgVyE3FkmpHsNJPy1oJ0FTfEXAbvCex5DMRBo2Kl2vEgm4lFKCpTn039N1H07zWusDOi8U/+WUhVfyrJJz+AQ==&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>

> 所谓“正在处理中且有新请求”，意思是同一个 Key 需要被重新入队，但需要被延迟到本次处理完成，这也就保证 Control Loop 可再处理它一次来处理最新的状态变化，而之所以需要将入队延迟到本次处理完成，是为了防止同一 Item 被多个 Worker 并行处理。

至此，Controller 的一般结构基本解释清楚。在 Informer 、Work Queue 等模块组成的这一整体架构的支持下，某个特定功能的 Controller 只要明确这几个问题：

1. 在哪些时机（时间），Controller 需要对一个 Object 进行检查和可能的处理？对于只关注 Pod 数量的 ReplicaSet 来说，是 Pod 被创建和删除等等导致 Pod 数量发生变化的时机；对于关注 Pod Metrics 的 Horizontal Pod Autoscaler 来说，每隔一段固定时间都需要对每个 Object 进行检查，因为 Metrics 总是在变化，而并非像离散的数量值一般有明显的 Edge 。这些时机决定了我们将以怎样的方式（例如定义哪些 Event Handler）向 Work Queue 中添加 Item 。它们要保证高效，但比高效更重要的是覆盖所有的情况，从而让 Controller 正确运行。
2. 对于一个特定的 Object ，Controller 需要收集怎样的当前状态信息，以及如何使用这些状态信息来做出行为决策。涉及 Object 的全部信息一般都从 Local Cache 中获取以提升效率，有时我们也可能从外部获取一些其他信息。覆盖所有的情况且保证 Control Loop 始终是 Level Driven 的都很重要，这决定了 Control Loop 的逻辑是否有效、可靠。Control Loop 可以在一个周期内也可在多个周期内使当前状态收敛到期望状态，但要注意避免多个周期的重复操作以及非精确的操作引起的状态震荡。

将这些作以总结后，我将介绍一些 ReplicaSet 具体的实现细节以作为前文抽象内容的一个具体示例。
