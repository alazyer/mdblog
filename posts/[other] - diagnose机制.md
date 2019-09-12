# Diagnose机制



## 基本概念



- HealthChecker，所有检查的入口，可以添加Component
- HealthReport，所有报告的入口，可以添加ComponentReport
- Component，所有需要进行检查的组件，需要实现了Diagnose方法，返回ComponentReport
- ComponentReport，Component的Diagnose方法返回结果，包含：status、name、message、suggestion和latency信息



## 一些步骤



- 在程序入口处，实例化一个HealthChecker，并添加需要检查的组件Component
- 所有需要被检查的功能/包，需要定义一个实现了Diagnose方法的对象
- 调用HealthChecker的Check方法会自动调用所添加组件的Diagnose方法获取ComponentReport，

最终生成HealthReport



## 公共库职责

- 定义HealthChecker对象、HealthReport对象、Component接口和ComponentReport对象
- HealthChecker实现Add、Check方法
- HealthReport实现Add方法
- 定义Component接口时，执行需要实现Diagnose方法，返回ComponentReport结果
- ComponentReport实现Check、AddLatency和MarshalJSON（非必须，因为是用做REST API返回，所以需要实现）方法



## 实现Component接口

- 一般定义一个对象，然后为该对象添加Diagnose方法
- 把对象自身需要检查所需要的东西（某个对象实例）在构建新对象实例时传入
- Diagnose方法中使用构建实例时传入的东西，调用提供的方法，判断Component的状态，从而生成ComponentReport