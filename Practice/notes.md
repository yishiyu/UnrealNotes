# 实践笔记

## 1. 数据流动

HUD/动画蓝图和人物之间数据的流动应该是单向的

- C++实现基类,定义设置/获取数据的函数.控制类来调用
- 定义接口蓝图,控制类根据接口调用
- 控制类可以是 Character/Controller 也可以是 HUD/动画蓝图,但是只能有一个控制类


