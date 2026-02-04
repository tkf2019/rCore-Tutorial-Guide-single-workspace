## 任务描述

@/home/tkf/arceos-workspace/rCore-Tutorial-in-single-workspace/ 是一个用来教学的组件化操作系统。

@/home/tkf/arceos-workspace/rCore-Tutorial-Guide-in-single-workspace/ 是这个操作系统的教学文档，包含了对于每一章代码（chX）的介绍以及实验安排。

我现在要根据现有的 ch8 的代码和文档来重构 @/home/tkf/arceos-workspace/rCore-Tutorial-Guide-in-single-workspace/ch8 的内容。 

## 任务参考

1. @/home/tkf/arceos-workspace/rCore-Tutorial-in-single-workspace/ch8 是可以正常运行 ch8 测试的代码。

2. @/home/tkf/arceos-workspace/rCore-Tutorial-Guide-in-single-workspace/ch3 是已经按要求完成的文档，只考虑 .rst 文件的内容，不要考虑 .md 文件的内容。

3. @/home/tkf/arceos-workspace/rCore-Tutorial-Guide-in-single-workspace/ch8 是修改前的文档内容，只考虑 .rst 文件的内容，不要考虑 .md 文件的内容。

## 任务要求

1. 对于 @/home/tkf/arceos-workspace/rCore-Tutorial-Guide-in-single-workspace/ch8，新文档应该和旧文档的逻辑、实验安排保持一致。

2. 在阅读 @/home/tkf/arceos-workspace/rCore-Tutorial-in-single-workspace/ch8 代码时，应该充分考虑到相关组件的依赖关系，例如对于 tg-* 组件的依赖，应尽可能讲清楚 ch8 是如何使用这些组件的。

3. 要求只对和框架代码分析的部分进行修改和完善，和代码无关的背景知识应该保留，例如 0intro 的线程定义、1thread-kernel 的线程概念，2lock 的 mutex 前的内容，3semaphore 背景知识，4condvar 的基本思路介绍。

4. 实验的运行命令有改动，现在是直接在 @/home/tkf/arceos-workspace/rCore-Tutorial-in-single-workspace/ch8 中执行 `cargo run` 。

5. 除了任务参考指定的内容外，只在必要情况下参考其他内容，例如需要保持文档风格、查找运行环境等。

6. 不需要运行代码测试，保证文档符合语法规范即可，不需要通过编译来验证语法。
