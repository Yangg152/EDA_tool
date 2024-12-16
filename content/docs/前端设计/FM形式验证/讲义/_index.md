---
weight: 41
---

# 形式验证  
## 形式验证介绍  
在IC设计中，需要通过DC工具将设计的RTL代码转换为网表，为了验证所生成的网表与RTL的功能是否一致，需要进行形式验证，形式验证是一种等价性检查。
它是一种静态的比较，会遍历所有的组合保证逻辑等价性。

## 形式验证的位置  
只要设计发生改变(对代码进行改动)
* 综合的网表与RTL对比做形式验证。保证综合过程没有逻辑错误。保证综合后的网表正确。
* 后端网表与综合后的网表对比做形式验证。保证后端没有引入逻辑错误。
* 做ECO的时候，ECO后的网表与ECO后的RTL做形式验证。（ECO当芯片已经流片出去了，工厂只做了一个底层，但金属层还没做可以做metalECO，发现某些容易修的bug后可以利用一些冗余的cell改变某些连线来修掉这个Bug，修改后端网表的同时对RTL也进行相应修改，然后将这两个文件进行LEC比较）

## 形式验证的流程  
1.读取综合产生的svf文件,一般在综合的目录下自动生成。
`注意：只有对比综合前的RTL和综合后的网表时候需要这一步`
在terminal输入fm_shell进入形式验证的环境，之后的命令都是在此环境中运行
```tcl
set_svf  xxx.svf
```
2.读入时序库文件
```tcl
read_db -r xxx.db #时序
```
3.读取RTL文件并设置顶层
```tcl
read_verilog -r  xxx.v        ;# Reference RTL file
set_top xxx
```
4.读取参与比较的网表文件并设置顶层
```tcl
read_verilog -i xxx.v           ;# Implementation Gate level file
set_top xxx
```
5.进行匹配与对比
```tcl
match
verify
```
6.输出报告
```tcl
report_guidance -summary > RPT/summary.rpt
report_failing > RPT/failing.rpt 
```

