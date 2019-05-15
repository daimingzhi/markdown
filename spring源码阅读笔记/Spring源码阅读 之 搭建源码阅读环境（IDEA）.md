Spring源码阅读 之 搭建源码阅读环境（IDEA）

##### 检出源码：

GitHub:https://github.com/spring-projects/spring-framework.git

可以按如下步骤：（须确保Git已正确安装）

1. Git正确安装后，在桌面上右击Git bash here,打开Git命令行窗口
2. 执行命令:git clone https://github.com/spring-projects/spring-framework.git
3. 克隆到桌面后用直接用idea打开目录
4. 切换到5.1.x分支 

##### 解决spring-cglib-repack.jar跟spring-objenesis-repack.jar缺少问题：

缺失原因：

​				为了避免第三方class的冲突，Spring把最新的cglib和objenesis给重新打包了（repack）,它并没有在源码里提供这部分代码，而是直接将其放在jar包中，这就导致了我们拉取代码时出现编译错误。

解决办法：

1. 首先，官网拉取下来的代码使用gradle管理的，所以我们先要将idea跟gradle整合，并配置环境变量，具体步骤可以自行百度，很简单
2. 在源码目录下打开黑窗口执行以下两个命令：gradle objenesisRepackJar跟gradle cglibRepackJar
3. 可以看到会在Spring-framework\spring-core\build\libs目录下生成jar包

![buildjar](H:\markdown\images\buildjar.png)







