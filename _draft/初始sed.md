最近在复习shell脚本，看到《linux命令行与shell脚本编程大全》第19章关于sed的介绍，下面做了一些用法总结。


## 1 初识sed
sed编辑器被称作流编辑器，它和vim这种的交互式文本编辑器不同，是根据命令来处理数据流中的数据。会执行下列操作：

* 一次从输入中读取一行数据（重复该操作直到全部行被读取完）
* 根据编辑器命令匹配数据
* 按照命令修改数据流中的数据
* 将新数据输出到STDOUT(标准输出)

其中sed命令的可以从命令行中输入，也可以从一个命令文件中读取。
> sed -e script 添加script中指定的命令

> sed -f file  添加file中指定的命令

### 1.1 在命令行定义sed命令
来看一个简单的文本替换示例：

``` shell
echo "hi,my name is xxx" | sed 's/xxx/kinsomy/'

#修改文件
sed 's/xxx/kinsomy/' data.txt

#执行多个命令 用-e选项,分号隔开
sed 's/xxx/kinsomy/; s/***/hhh/' data.txt
```

将echo输出的数据通过管道输入sed中，然后用s命令进行替换，用第二个斜杠后的数据替换掉第一个斜杠后匹配的数据。

> **注意：sed操作文本文件中的数据，仅仅是将修改的数据输出到STDOUT，但是并不会修改文件本身的数据**

### 1.2 从文件读取命令
在一个文件`script.sed`中定义一系列的命令,方便复用。
```
s/*/a
s/x/b
s/-/+
```

```shell
# -f选项指定命令文件
sed -f script.sed data.txt
```

## 2 sed基础

### 2.1更多替换选项
上面的例子`echo "hi,my name is xxx" | sed 's/xxx/kinsomy/'`只会替换每一行中匹配到的第一个数据，但是一行数据中若有多个匹配项，则不能全部被替换掉。

``` shell
echo "hi,my name is xxx, i am xxx" | sed 's/xxx/kinsomy/'

#输出
hi,my name is kinsomy, i am xxx
```

这个时候可以使用一些替换标记substitution flag来设置替换的模式。替换标记跟在替换字符串之后。

`s/pattern/replacement/flags`

* 数字，表示将替换掉第几处被匹配到的数据

```shell
echo "hi,my name is xxx, i am xxx" | sed 's/xxx/kinsomy/2'

#输出 第二个xxx被替换成kinsomy
hi,my name is xxx, i am kinsomy
```

* g,表示替换所有匹配到的数据
```shell
echo "hi,my name is xxx, i am xxx" | sed 's/xxx/kinsomy/g'

#输出 第二个xxx被替换成kinsomy
hi,my name is kinsomy, i am kinsomy
```

* p,表示会打印出被匹配出来的行
```shell
echo "hi,my name is xxx, i am xxx" | sed 's/xxx/kinsomy/p'
#输出
hi,my name is kinsomy, i am xxx
hi,my name is kinsomy, i am xxx
```

* w,将替换后输出保存到指定文件

```shell
echo "hi,my name is xxx, i am xxx" | sed 's/xxx/kinsomy/w output.txt'
```


