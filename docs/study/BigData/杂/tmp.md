# 杂

# 1. MPP

> [MPP概述_trocp的专栏-CSDN博客_mpp数据库](https://blog.csdn.net/trocp/article/details/86687206)
>
> [MPP架构_迷路剑客个人博客-CSDN博客_mpp架构](https://blog.csdn.net/baichoufei90/article/details/84328666)
> [MPP 与 Hadoop是什么关系？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/22037987)

# 2. Data Lake

​	感觉就是弱化存储介质的差异性，统一管理，需要有完善的管理形式，方便集中读写数据、管理数据等。

​	一般需要对外抽象数据访问形式，使得能够简易地接入各种数据处理组件，而不过多地关心底层数据存储的差异性。

> [Hello from Apache Hudi | Apache Hudi](https://hudi.apache.org/)
>
> [Data lake - Wikipedia](https://en.wikipedia.org/wiki/Data_lake)
>
> [数据湖 | 一文读懂Data Lake的概念、特征、架构与案例_Focus on Bigdata-CSDN博客_datalake](https://blog.csdn.net/u011598442/article/details/106610486)
>
> [数据湖（Data Lake） 总结 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/91165577)
>
> [DataLake 基本概念_test_fy的博客-CSDN博客_datalake](https://blog.csdn.net/test_fy/article/details/80881958)

# 3. Data Fabric

​	感觉就是解决数据数据分布散乱的问题，类似K8s管理多个容器，Data Fabric概念管理多个复杂数据源（不是指存储介质or存储引擎，是对数据更高维度的抽象）。

​	比如某些服务器可能过去承载了PB级别的某种类型数据（可以是某类业务数据，多种形式的存储介质整合的产物），现在需要扩展，但是不方便迁移这些大量数据，就索性不迁移，但是通过一些服务注册、发现机制，使得这部分数据更易被读写、管理。

> [IBM Cloud Pak for Data - 中国 | IBM](https://www.ibm.com/cn-zh/products/cloud-pak-for-data)
>
> [Data Fabric (数据经纬）: 下一个IT的风口？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/400354162)