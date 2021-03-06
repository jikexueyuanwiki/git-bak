# 司令官与副官工作流

这其实是上一种工作流的变体。一般超大型的项目才会用到这样的工作方式，像是拥有数百协作开发者的 Linux 内核项目就是如此。各个集成管理员分别负责集成项目中的特定部分，所以称为副官（lieutenant）。而所有这些集成管理员头上还有一位负责统筹的总 集成管理员，称为司令官（dictator）。司令官维护的仓库用于提供所有协作者拉取最新集成的项目代码。整个流程看起来如下图：司令官与副官工作流  所示：

- 1.一般的开发者在自己的特性分支上工作，并不定期地根据主干分支（dictator上的master）衍合。
- 2.副官（lieutenant）将普通开发者的特性分支合并到自己的 master 分支中。
- 3.司令官（dictator）将所有副官的 master 分支并入自己的 master 分支。
- 4.司令官（dictator）将集成后的master分支推送到共享仓库blessed repository中，以便所有其他开发者以此为基础进行衍合。

![Image of CVCS_3]		
(images/CVCS_3.png)

这种工作流程并不常用，只有当项目极为庞杂，或者需要多级别管理时，才会体现出优势。利用这种方式，项目总负责人（即司令官）可以把大量分散的集成工作委托给不同的小组负责人分别处理，最后再统筹起来，如此各人的职责清晰明确，也不易出错（译注：此乃分而治之）。

以上介绍的是常见的分布式系统可以应用的工作流程，当然不止于 Git。在实际的开发工作中，你可能会遇到各种为了满足特定需求而有所变化的工作方式。我想现在你应该已经清楚，接下来自己需要用哪种方式开展工作了。下 节我还会再举些例子，看看各式工作流中的每个角色具体应该如何操作。
