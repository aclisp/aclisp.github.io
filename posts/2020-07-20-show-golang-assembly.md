# 看 Golang 的汇编代码

```
dlv test -- -test.run ^TestMapReference$  // 启动调试器运行测试用例
b TestMapReference  // 在测试用例入口处设置断点(支持按<TAB>自动完成)
c                   // 运行
disass              // 反汇编当前位置
```
