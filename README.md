# CodeQL Workshop: Find bug in apache struts 2

## 问题描述

非常多类型的漏洞，其挖掘工作本质上都是找到从不安全的用户输入到一个危险的操作的完整路径，CodeQL 对挖掘这类问题非常擅长，大幅度的降低了漏洞挖掘的门槛，只需要定义好什么是不安全的输入以及什么是危险的操作，剩下的工作交给内置的 DataFlow 模块即可完成。

使用 CodeQL 来挖掘漏洞的模式基本可以归纳化为：

1. 分析历史漏洞
2. 归类漏洞，将漏洞按照不同的起点和终点归类
3. 将起点(source)，终点(sink)具体化描述，要具体化到 AST node 级，即source 是某个名为 xxx 函数调用的第一个类型为 yyy 的参数，sink 是 xxxxxx
4. 用 CodeQL 的方式描述出来
5. 运行 Query，分析结果有效性

本次 Workshop 会通过上述步骤给参与者一个使用 CodeQL 挖掘漏洞的直观印象，并从中识别出 S2-057

## 准备步骤

1. 安装 VSCode
2. 安装 CodeQL 的 VSCode 插件->[指北](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/)
3. [克隆初始化 Workspace](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html#using-the-starter-workspace)
4. 打开 Workspace
5. 下载指定 commit 的 Struts2 CodeQL DB
6. 选择该 DB
7. 在`codeql-custom-queries-java`下创建`Workshop.ql`

## Workshop

每一步都有一个提示，提示信息包含了一些 CodeQL 中有用的 API，使用 CodeQL 的时候语法提示很有用，基本上 API 的名字就说清楚了这个 API 是干嘛用的。

### 第一步：分析历史漏洞

主要分析目标：S2-032

#### [S2-032](https://cwiki.apache.org/confluence/display/WW/S2-032)

先看漏洞描述：

> It is possible to pass a malicious expression which can be used to execute arbitrary code on server side when Dynamic Method Invocation is enabled.

在`Dynamic Method Invocation`开启的前提下，恶意的表达式可能通过用户输入注入到服务端上。

在 `org.apache.struts2.dispatcher.Dispatcher`的`serviceAction`中

```java
            UtilTimerStack.push(timerKey);
            String namespace = mapping.getNamespace();
            String name = mapping.getName();
            String method = mapping.getMethod();

            ActionProxy proxy = getContainer().getInstance(ActionProxyFactory.class).createActionProxy(
                    namespace, name, method, extraContext, true, false);
```

传入了 method

执行点在`com.opensymphony.xwork2.DefaultActionInvocation`中的`invokeAction`方法

```java
   protected String invokeAction(Object action, ActionConfig actionConfig) throws Exception {
        String methodName = proxy.getMethod();

        ....

                try {
                    String altMethodName = "do" + methodName.substring(0, 1).toUpperCase() + methodName.substring(1)  + "()";
                    methodResult = ognlUtil.getValue(altMethodName, ActionContext.getContext().getContextMap(), action);
                } 
        ....
    }
```

可见 `methodName` 进入了 ognl 解析的过程中。

该例子里，source 是 methodName，来自于对 ActionProxy 的 getMethod 的调用，sink 是 ognlUtil.getValue。

而 sink 的 ognlUtil 中可能执行 ognl 表达式的方法其实是 OgnlUtil.compileAndExecute。

### 第二步：将 source、sink 具体化

Source: `com.opensymphony.xwork2.ActionProxy` 的 `getMethod`、`getNamespace`、`getActionName` 的返回值

Sink: `com.opensymphony.xwork2.ognl.OgnlUtil`的`compileAndExecute`方法。

### 第三步：用 CodeQL 的形式描述出来

#### Step 1： 得到所有的方法调用 （2 分钟）

<details>
  <summary>Hint</summary>
  CodeQL 中对方法的调用使用 MethodAccess 表示
</details>

#### Step 2： 给结果加一列，查找到每个**方法调用**所调用的方法（2 分钟）

<details>
  <summary>Hint</summary>
  CodeQL 中对方法的表示使用的 Method 类，MethodAccess 存在一个 predicate 叫做 getMethod，可以获取到对应的 method。添加一个 where 语句查询指定条件即可。
</details>

#### Step 3：具体化第二步的查询，使之可以查询出来调用了 getMethod、getNamespace、getAction 方法的地方（2 分钟）

<details>
  <summary>Hint</summary>
  Method 类有一个 predicate 叫 getName，使之等于这几个方法来编写 where 语句即可。
</details>

#### Step 4：具体化第三步查询，使查询到的结果限定在 com.opensymphony.xwork2.ActionProxy 内（4 分钟）

<details>
  <summary>Hint</summary>
  Method 类有 getDeclaringType 可以用于找到方法所在的类（RefType）。RefType 有 hasQualifiedName 用于判断当前类的全名是不是指定 package 与 name。 
</details>

#### Step 5：找到第四步结果的 override（2 分钟）

对这些 source 的调用除了对该方法本身，对该方法的重写也是查找范围内，可能会找到意料之外的结果。

<details>
  <summary>Hint</summary>
  对于 Method n，n.overrides(m) 表示判断方法 n 是否重写了方法 m，n.overrides*(m) 则可用于多级重写的情况。 
</details>

#### Step 6： 将第五步的 ql 重构为 predicate（3 分钟）

复杂的查询重构为 predicate 有助于代码重用，我们意图找到某个方法调用，可以使用如下模板

```
predicate isActionProxySource(MethodAccess ma) {
  /** TODO */
}
```

#### Step 7：寻找对方法 compileAndExecute 的调用 （2 分钟）

近似于第三步

#### Step 8：寻找 compileAndExecute 调用的第一个参数 （2 分钟）

sink 就在第一个参数

<details>
  <summary>Hint</summary>
  MethodAccess 可使用 getArgument(pos)的方式找到第 pos 个参数，pos 从 0 开始。 
</details>

#### Step 9：将上一步的 ql 重构为 predicate（2 分钟）

类似于第六步，使用模板：

```
/* Refactor the logic into a predicate. */
predicate isOgnlSink(Expr arg) {
  /** TODO */
}

```

#### Step 10：数据流分析（5 分钟）

我们现在定义了 source 与 sink，现在希望  codeql 能够帮我们给 source 和 sink 之间连个线，寻找可能的从 source 到 sink 的路径，这也是平时代码审计中最花时间的地方。

在程序分析的领域中，这个步骤叫做数据流分析（Data flow），数据流分析回答了一个问题：程序某个位置的值是否来自于我们感兴趣的某个地方，以及数据的整个传递流程。

考虑这个 C 程序：


```c
int func(int tainted) {
   int x = tainted;
   if (someCondition) {
     int y = x;
     callFoo(y);
   } else {
     return x;
   }
   return -1;
}
```
在这个方法中，我们可以发现数据流是这样传递的：

<img src="https://help.semmle.com/QL/ql-training/_images/graphviz-2ad90ce0f4b6f3f315f2caf0dd8753fbba789a14.png" alt="drawing" width="260"/>

上图中的每个 node 都代表程序中的一个元素，比如 tainted 在程序中先是作为参数，然后作为了赋值表达式的右值。在 codeql 中，AST 节点里拥有对应数据流节点的只有三种情况，分别是表达式、函数参数、变长参数的数组，前两种最为常用。

每条边都代表依次数据流的传递，

初步学习数据流分析的使用，完成如下模板：

```
/**
 * @kind problem
 */
import java
import semmle.code.java.dataflow.DataFlow

class OgnlCfg extends DataFlow::Configuration {
  OgnlCfg() { this = "ognl" }

  override predicate isSource(DataFlow::Node source) {
    /** TODO */
  }

  override predicate isSink(DataFlow::Node sink) {
    /** TODO */
  }
}

from OgnlCfg config, DataFlow::Node source, DataFlow::Node sink
where config.hasFlow(source, sink)
select sink, "潜在的的 OGNL 表达式注入"
```

<details>
  <summary>Hint</summary>
  Node 的 asExpr 方法会返回数据流节点对应的 AST 中的 Expr Node。
  Node 的 asParameter 方法会返回对应 AST 中的 Parameter Node
</details>

#### Step 11：带有路径信息的数据流分析（5 分钟）

在上一步的基础上，看到每个结果的完整路径

<details>
  <summary>Hint</summary>
  利用开头注释 @kind path-problem 提示 CodeQL 该查询需要显示路径信息。
  引入 import semmle.code.java.dataflow.DataFlow 来让 CodeQL 在结果中绘制每一步路径。
  将 source 与 sink 的类型改为 DataFlow::PathNode 以保留路径信息。
  使用 hasFlowPath 代替 hasFlow。
  select 语句格式也需要修改。
</details>

#### Step 12：为 path 设立 barrier（3 分钟）

结果较少的时候可以人工看，结果较多的时候仍然需要想方设法的缩小结果范围，观察结果可以发现，许多条 path 都会经过一个叫做 ValueStackShadowMap 的类，而这个类不会被用到，这一步的任务就是将该类从结果中剔除。请为 OgnlCfg 添加 isBarrier 方法

```
override predicate isBarrier(DataFlow::Node node) {
  ...
}
```

<details>
  <summary>Hint</summary>
  判断当前节点不在 ValueStackShadowMap 类内。RefType 获取类名（不包含包名）使用getName predicate
</details>


#### Step 13：额外的污点分析（6 分钟）

CodeQL 数据流分析存在很多局限性，比如刚才 ppt 里讲到的，数据经过了一些修改就会无法继续跟踪，这时候额外的污点分析可以起到作用，当然还有一些想象不到的，比如说

```java
class A {
	int a;
	int getA() {
		return a;
	}
	void setA(int value) {
		a = value;
	}
}

void FuncA(A b) {
	int c = b.getA();
}

void FuncB() {
	b.setA(3);
}

```

类似这种情况，`FuncB`通过某个属性间接的设置了对象属性 a 的值，也不会被跟踪到，这都需要额外的步骤来处理。这一步让我们先来处理一下本地数据流污染的情况，请为  `OgnlCfg` 添加 `isAdditionalFlowStep`，这个 predicate 主要用于告诉 CodeQL node1 与 node2 之间是否存在数据传递。请使用如下模板，在 TODO 处填入合适的语句。

```
class OgnlCfg extends DataFlow::Configuration {
  OgnlCfg() { this = "ognl" }

  override predicate isSource(DataFlow::Node source) {
    ...
  }

  override predicate isSink(DataFlow::Node sink) {
    ...
  }

  override predicate isBarrier(DataFlow::Node node) {
    ...
  }

  override predicate isAdditionalFlowStep(DataFlow::Node node1, DataFlow::Node node2) {
    /** TODO */
  }
}
```

<details>
  <summary>Hint</summary>
  引入 semmle.code.java.dataflow.TaintTracking 包，使用 TaintTracking::localTaintStep 来判断 node1 与 node2 是否在本地函数内存在污点传递。
</details>
#### 遗留问题 1

上一步遗留了一个污点传递的情况，其实还有很多方面没有考虑完善，限于时间，当做[扩展阅读](https://securitylab.github.com/research/apache-struts-CVE-2018-11776/)留给大家做课后作业。
本 workshop 正是拆解了这篇文章的思路完成的，经过了这次 workshop，完整理解整篇文章应当并不困难了。

#### 遗留问题 2

我们分析 java 应用漏洞的时候，经常会思考到底一个漏洞从一个 http 请求到漏洞点的完整流程是什么样子的，借助 codeql 我们完全可以实现这一点，sink 不改变，我们仔细看 source，其实我们一开始也没探究过 source 为什么是  getNamespace 这些方法，如果我们就是想找到更多的漏洞输入点，写一个更通用的查询是否更好呢？请详细阅读[扩展阅读 2](https://securitylab.github.com/research/ognl-injection-apache-struts/)。从中能够理解针对一个 JAVA 应用寻找某个特定 sink 的完整触发路径的通用方法

#### 遗留问题 3

想了解 C/C++ 相关的内容吗？限于篇幅没能完整的介绍，不过确实是有非常多的相似之处，我个人认为 CodeQL 作为一个工具只要对一个语言会使用对其他语言应该也不成问题，若想了解更多内容，请阅读本人倾情编写的《CodeQL C/C++ 指北》作为扩展阅读 3

### Reference

[CVE-2018-11776 How to find 5 RCEs in Apache struts with CodeQL](https://securitylab.github.com/research/apache-struts-CVE-2018-11776/)
[OGNL injection in Apache Struts: Discovering exploits with taint tracking](https://securitylab.github.com/research/ognl-injection-apache-struts/)
[Apache Struts double evaluation RCE lottery](https://securitylab.github.com/research/apache-struts-double-evaluation/)
[S2-032](https://cwiki.apache.org/confluence/display/WW/S2-032)
[QL language reference](https://codeql.github.com/docs/ql-language-reference/)
[S2-032远程代码执行漏洞分析](https://pino-hd.github.io/2018/06/19/S2-032%E8%BF%9C%E7%A8%8B%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)
[Analyzing data flow in Java](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-java/)
[About data flow analysis](https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/)