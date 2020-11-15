原文地址: [juejin.im/post/684490…](https://juejin.im/post/6844903946058727437)

在团队协作开发中, 各成员需要互相配合来完成功能的开发, 测试和发布. 但各成员习惯各异, 每个人对如何协作的理解也不同, 因此需要一个**协作模型**来规范团队不同职能的人员以及相同职能的人员如何配合.

下面将基于Git来进行工作流层面的分析.

## 传统的Git flow

比较传统的一个Git flow是来自Vincent Driessen的分享, 整体交互图如下:

![16d4a5d51675cd2f.png](https://upload-images.jianshu.io/upload_images/3609683-9b4f1fe1da226956.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> [A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/).

在这个模型中, 有两个主分支和其它功能分支, 主分支包括:

*   **master**: 存放着生产级别的代码.
*   **development**: 存放着最新开发的,可集成的功能的代码.

其他功能分支包括:

*   Feature branches: 基于development创建的分支, 用于新功能的开发.
*   Release branches: 基于development创建的分支, 用于发布新的功能, 稳定运行一段时间后合并到master分支, 作为最终的发布.
*   Hotfix branches: 基于master创建的分支, 用于修复bug.

### 适用场景分析

这个模型适用的场景有:

*   各阶段开发的功能明确: 因为发布是直接从development中创建新的release分支, 若development中有未完成(开发或测试)的功能则会影响版本的发布.
*   大量而低频的发布: 因为development中可能含有多个功能, 因此需要各功能都测试完毕后才能发布. 如桌面应用, 移动应用的开发等.

不适用的场景:

*   少量和高频的发布.
*   多环境部署(如开发,测试和生产). 如Restful API的开发等.

## 改进的Git flow

针对传统Git flow的问题, 改进完后模型如下图:
![16d4a5d516ddad7b copy.jpg](https://upload-images.jianshu.io/upload_images/3609683-70443b62fcba5b30.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


该模型包含三个主分支和其他功能分支, 主分支包括:

*   **development**: 用于新功能的开发端测试(作为开发者自己测试)和集成(如前后端的API集成).
*   **staging**: 测试分支, 已完成开发端测试和集成的功能分支会合并到这里, 待测试人员测试.
*   **production**: 生产分支, 存放生产级别代码; 功能分支通过测试人员在staging环境中的测试后, 将合并到该分支.

> development和staging分支的初始化都基于production分支.

其他功能分支包括:

*   Feature branches: 基于production创建的分支, 用于新功能的开发; 命名格式为**feature/feature-name**, 如**feature/payment**;
*   Bugfix branches: 基于production创建的分支, 用户普通bug的修复, 合并流程同feature分支;命名格式为**bugfix/bug-name**.
*   Hotfix branches: 用于紧急bug的修复, 可直接合并至production分支, 再分别合并到development和staging分支, 命名格式为**hotfix/bug-name**.

> 所有的功能分支都只在开发者本地, 不能上传到远程仓库.

该模型的运行流程如下:

1.  得到一个功能需求如支付, 基于**production**分支创建一个新的功能分支**feature/payment**;
2.  切换到功能分支**feature/payment**进行开发;
3.  本地开发结束, 将**feature/payment**分支合并到**development**分支, 进行开发端测试和集成;
4.  若在开发端测试和集成中发现问题, 切换回**feature/payment**分支去修复问题, 修复完后执行步骤3;
5.  若在开发端测试和集成中没有问题, 将**feature/payment**分支合并到**staging**分支, 让测试人员来测试;
6.  若测试人员在**staging**中发现有关该功能的问题, 切换回**feature/payment**分支, 修复完后执行步骤3;
7.  若测试人员在**staging**中认为该功能已测试完毕, 则将**feature/payment**分支合并到**production**分支.

### 规则

为了保证该模型的顺畅执行, 团队成员还应遵守以下几条规则:

1.  所有的功能分支都需要基于最新的**production**分支创建;
2.  所有的功能分支都只在存在开发者本地, 不能上传到远程仓库, **远程仓库只有主分支**;
3.  主分支(development, staging和production)之间**不能相互合并**;
4.  主分支**只能合并功能分支**(如feature,bugfix和hotfix分支);
5.  功能分支最终要合并到**所有的主分支**中.
6.  每次合并都要留有记录(merge命令带`--no-ff`);

通过以上改进之后, 该模型就可以支持多环境部署, 并且保证功能合并的独立性.

当然, 不同团队的业务场景也有差异, 对各种模型也需要有相应的调整;

总而言之, 没有绝对完美的模型, 场景决定了适用的模型.
