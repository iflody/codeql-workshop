# Semmel Codeql c/cpp 指北

## 创建数据库

```
codeql database create codeql --language=cpp --command="autoninja -C out/codeql chrome"
```
该命令如果不加 `--command` 参数则会自动的寻找当前目录下存在的编译系统配置文件，自动开始编译，如果指定了 `--command` 参数就会使用指定命令编译，然后会通过分析系统调用等方式来分析编译行为，构造数据库。

## vscode integration

使用 vscode 来管理、运行 ql 很方便，首先先下载安装 codeql 的 vscode 的[插件](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql)。然后配置 *executable path* 为 *codeql* 的命令绝对地址。在下载一个 [workspace starter](https://github.com/github/vscode-codeql-starter/)，按照仓库内的说明 clone submodule，打开这个 workspace，所有的环境就已经全部配置好了，在对应的目录下编写 ql，在插件窗口的 DATABASES 上点击➕，添加数据库，ql 编写好后按 `ctrl+shift+p`输入 `codeql run` 选择 `run query` 即可运行 ql 查询。

## c/cpp ql library 初步使用

### Example query

```sql
import cpp /* 引入 cpp library */

from Function f /* 从所有的 Function 中筛选 */
where f.isStatic() /* 如果 f 是静态函数的话则满足条件 */
select f, "This is a static function." /* 导出结果 */
```

注释很清楚了。

### 一些可能不清楚的语法

#### 传递闭包与反身传递闭包

```sql
Person getAParent() {
  result = this.parent
}
```

此例中，传递闭包用法为 `getAParent+()`，会递归向上一直调用直到尽头，将所有结果加入集合中，是一种优雅的递归方法。

反身传递闭包用法为 `getAParent*()`，在传递闭包的基础上把自己本身包含进来。及递归次数是 0-n

#### inline-casting

ql 中的类型转换，举例来说：

```sql
from MemberVariable m
where m.getName() = "aaa"
select m.getEnclosingElement().(Class).getName()
```

上例中，getEnclosingElement 的返回值可能是 class 可能是 function 等，这种情况下想要具体处理的话可以使用 inline-casting 将其转换为 Class 类型


### Functions

#### 函数调用相关

1. 函数直接调用

```sql
import cpp

from Function f
where not exists(FunctionCall fc | fc.getTarget() = f)
select f, "This function is never called."
```

2. 函数指针引用

```sql
import cpp

from Function f
where not exists(FunctionCall fc | fc.getTarget() = f)
  and not exists(FunctionAccess fa | fa.getTarget() = f)
select f, "This function is never called, or referenced with a function pointer."
```

3. 寻找符合指定条件的函数调用

```sql
import cpp

from FunctionCall fc
where fc.getTarget().getQualifiedName() = "sprintf"
  and not fc.getArgument(1) instanceof StringLiteral
select fc, "sprintf called with variable format string."
```

4. 更复杂的包含 namespace 的案例

```sql
import cpp
 
from FunctionCall call, Function fcn
where
  call.getTarget() = fcn and
  fcn.getDeclaringType().getSimpleName() = "map" and
  fcn.getDeclaringType().getNamespace().getName() = "std" and
  fcn.hasName("find")
select call
```

5. 两个函数互相调用

```sql
import cpp
 
from Function m, Function n
where
  exists(FunctionCall c | c.getEnclosingFunction() = m and c.getTarget() = n) and
  exists(FunctionCall c | c.getEnclosingFunction() = n and c.getTarget() = m) and
  m != n
select m, n
```

6. 函数重载查找

```sql
import cpp
 
from MemberFunction override, MemberFunction base
where
  base.getName() = "what" and
  base.getDeclaringType().getName() = "exception" and
  base.getDeclaringType().getNamespace().getName() = "mojom" and
  base.getDeclaringType().getNamespace().getParentNamespace() = "blink" and
  override.overrides+(base)
select override
```

#### 实战中遇到的问题记录

##### FunctionCall 查找参数类型

1. Expr getType, getActualType, getUnspecificedType

getType 获取的类型是最简单直接的类型，不会展开 `typedef` 也不会去掉 `const` 这类修饰符，也不会考虑类型转换

getActualType 会展开 `typedef`，去掉修饰符，同时考虑所有类型转换，即最终实际的类型

getUnspecificedType 展开 `typedef`，去掉修饰符，不考虑类型转换。

### 表达式、类型、语句

#### Expr and Type

有各种内置的 c/cpp expr 类型，他们都 extend 了 Expr 类，这里先拿官方教程说明一下：

```sql
import cpp

from AssignExpr e
where e.getRValue().getValue().toInt() = 0
select e, "Assigning the value 0 to something."
```

这里在所有的赋值表达式中寻找赋值为 0 的表达式。

```sql
import cpp

from AssignExpr e
where e.getRValue().getValue().toInt() = 0
  and e.getLValue().getType().getUnspecifiedType() instanceof IntegralType
select e, "Assigning the value 0 to an integer."
```

这个会再校验一下左值的类型，这里值得一提的是  `Type` 类型的 `getUnspecifiedType` 方法，会得到完全展开并去掉冗余后的类型，比如

```cpp
typedef long long i64;
const i64* a;
```

如果这里对 a 的类型调用 `getUnspecifiedType` 方法的话，会得到 `long long*` 类型。`const` 会被 strip 掉。

这里再列一些常用 Expr

1. 数组

```sql
import cpp
 
from ArrayExpr a
where a.getArrayOffset() instanceof PostfixIncrExpr
select a
```
判断了数组的 index 是否是一个包含了后缀表达式的 Expr

2. 类型转换

```sql
import cpp
 
from Cast c
where
  c.getExpr().getType() instanceof FloatingPointType and
  c.getType() instanceof IntegralType
select c
```

3. 三元运算符（...?...:...）

```sql
import cpp
 
from ConditionalExpr e
where e.getThen().getType() != e.getElse().getType()
select e
```

看两种分支返回的表达式类型是否一致

#### Statement

几种常用 Statement

- Stmt - C/C++ statements
	- Loop
		- WhileStmt
		- ForStmt
		- DoStmt
	- ConditionalStmt
		- IfStmt
		- SwitchStmt
	- TryStmt
	- ExprStmt - expressions used as a statement; for example, an assignment
	- Block - { } blocks containing more statements

1. 寻找初始赋值为 0 的循环

```sql
import cpp

from AssignExpr e, ForStmt f
// the assignment is in the 'for' loop initialization statement
where e.getEnclosingStmt() = f.getInitialization()
  and e.getRValue().getValue().toInt() = 0
  and e.getLValue().getType().getUnspecifiedType() instanceof IntegralType
select e, "Assigning the value 0 to an integer, inside a for loop initialization."
```

这里需要注意，`e` 是 Expr，`f.getInitialization()` 是 Statement，这两种类型是不相同的，在使用时针对自己写的判断务必要注意返回值类型，在非子类的情况下也无法使用类型转换。`getEnclosingxxxx` 的方法经常会使用到，使用时结合 IDE 提示根据情况判断需要获取到哪个单元，比如 expr 外不一定是函数包裹的，也有可能还是 expr，所以使用这种类型的方法时一定要明确自己想要的到底是什么类型，必要时可以使用传递闭包。


### Dataflow

#### Dataflow 的一般表达形式

要明确 source 与 sink 的定义：

source 是你想要查找的来源，sink 是你想要知道的数据落点。

一般情况下，想要查找 DataFlow 需要配置一个 Configuration，定义 source 与 sink，以及其他数据可能被传递的方式（比如一个数据，在一个函数中被传给了另一个指针指向的区域，ql 是无法追踪这个污点的，然而可以通过定义配置来把这种额外的污点传递给算进来），还要配置如何排除针对 data 的验证。这里构造一个简单的模型。

```c++
#include <iostream>
#include <stdlib.h>

class A {
	public:
	A(int b) {
		num = b;
	}
	int num;
};

void vulnerable(int c) {
	system("/bin/sh");
}

void test(A a) {
	vulnerable(a.num);
}

int main() {
	A a(12);
	if (a.num > 10) {
		exit(0);
	} else {
		vulnerable(a.num);
	}
	test(a);
}
```

上面这个模型基本包含了 source, sink, 额外的数据验证，通过编写一个 ql 尝试在这段代码里找到一段正确的 Dataflow 基本就能掌握这里的基本用法。

```sql
/**
 * @kind path-problem
 */
 
import cpp
import semmle.code.cpp.dataflow.DataFlow
import DataFlow::PathGraph

class VulClass extends Class {
    Class target;
    VulClass() {
        exists(Class a | a.getName() = "A" and a.getAMemberFunction().getName() = "A" and target = a) and this = target
    }
}

class Configuration extends DataFlow::Configuration {
    Configuration() {
        this = "test configuration"
    }
    override predicate isSource(DataFlow::Node source){
        any(MemberFunction f | f.getName() = "A" and f.getEnclosingElement() instanceof VulClass).getParameter(0) = source.asParameter()
    }

    override predicate isSink(DataFlow::Node sink){
        any(Function fcn | fcn.getName() = "vulnerable").getParameter(0) = sink.asParameter()
    }
}


from Configuration cfg, DataFlow::PathNode source, DataFlow::PathNode sink
where cfg.hasFlowPath(source, sink)
select source, sink, "data flow"
```

这段 ql 中的几个关键点：

1. 开头注释内容是 ql 的 metadata，只有包含了 `@kind path-problem` 的 metadata 才会提供关于 path 的 warning，否则最后查询结果只有 source 和 sink，这样的话无法得知中间路径，所以这个 metadata 一定要有。

2. `import semmle DataFlow::PathGraph` 这个也一定要 import，理由同 1
3. 定义一个 `Configuration`，可以定义上面说的 source、sink，这里我们定义 source 为 `class A` 的构造函数的第一个参数。sink 为 vulnerable 函数的第一个参数，运行查找，即可在 `warning` 中得到两条 path，其中一条经过了 else 语句，然而我们知道，这里其实是不可能经过的，该模型抽象了代码中常见的数据检查的情况，我们需要排除这种情况。这时候就需要在 Config 中定义 `isBarrier`。

```sql
...
    override predicate isBarrier(DataFlow::Node sink) {
        exists(ConditionalStmt condstmt | sink.asExpr().(VariableAccess).getTarget().getAnAccess() = condstmt.getControllingExpr().getAChild*() and condstmt.getAChild+() = sink.asExpr().getEnclosingElement() and not (condstmt.getControllingExpr().getAChild*() = sink.asExpr()))
    }
...
```

这个 barrier 的三个条件：

1. 存在条件判断语句，其访问了 sink 的变量
2. sink 被 条件判断语句包围
3. sink 与 条件判断不重合（有时候 sink 就在 条件判断内不能误判）

至此得到唯一一条 path。

微软的博客中提供了类似的语句，请思考为什么有问题

```
  override predicate isBarrier(DataFlow::Node node) { 
    exists(ConditionalStmt condstmt |  
      // dataflow node variable is used in expression of conditional statement
      //   this includes fields (because FieldAccess extends VariableAccess)
      node.asExpr().(VariableAccess).getTarget().getAnAccess()
                                          = condstmt.getControllingExpr().getAChild*()
      // and that statement precedes the dataflow node in the control flow graph
      and condstmt.getASuccessor+() = node.asExpr()
      // and the dataflow node itself not part of the conditional statement expression
      and not (node.asExpr() = cs.getControllingExpr().getAChild*())
    ) 
  }
```

答案：getASuccessor+ 会把后继以及后继的后继全部包含进来，出了  if 判断的话这个 barrier 就失效了。不够精确，会漏报。