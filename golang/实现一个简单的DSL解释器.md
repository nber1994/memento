--- 
title: go-实现一个简单的DSL解释器 
date: 2020-07-02
categories: 
- go 
---

# 什么是DSL
DSL 是 Domain Specific Language 的缩写，中文翻译为领域特定语言（下简称 DSL）；而与 DSL 相对的就是 GPL，这里的 GPL 并不是我们知道的开源许可证，而是 General Purpose Language 的简称，即通用编程语言，也就是我们非常熟悉的 Objective-C、Java、Python 以及 C 语言等等。

简单说，就是为了解决某一类任务而专门设计的计算机语言。

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20200702232708668_761509773.png)

* Regex
* SQL
* HTML&CSS

## 共同特点
没有计算和执行的概念；

* 其本身并不需要直接表示计算；
* 使用时只需要声明规则、事实以及某些元素之间的层级和关系；
* 总结起来一句话：表达能力有限，通过在表达能力上做的妥协换取在某一领域内的高效
**那么DSL解释器的主要功能是解释执行DSL**

# 设计原则
实现DSL总共需要完成两部分工作：

设计语法和语义，定义 DSL 中的元素是什么样的，元素代表什么意思
实现 parser，对 DSL 解析，最终通过解释器来执行
那么我们可以得到DSL的设计原则：

## 简单
* 学习成本低，DSL语法最好和部门主要技术栈语言保持一致（go，php）
* 语法简单，删减了golang大部分的语法，只支持最基本的
    * 数据格式，
    * 二元运算符，
    * 控制语句
    * 少量的语法糖
## 嵌入式DSL
* DSL需要嵌入到现有的编程语言中，发挥其实时解释执行且部署灵活的特点 
* 使用json类型的context与外部系统进行通信，且提供与context操作相关的语法糖

# 解释器工作流程
大部分编译器的工作可以被分解为三个主要阶段：解析（Parsing），转化（Transformation）以及 代码生成（Code Generation）

* 解析 将源代码转换为一个更抽象的形式。
* 转换 接受解析产生的抽象形式并且操纵这些抽象形式做任何编译器想让它们做的事。
* 代码生成 基于转换后的代码表现形式（code representation）生成目标代码。

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20200702235740827_1839702338.png)

## 解析
* 词法分析 —— tokenizer     通过一个叫做tokenizer（词素生成器，也叫lexer）的工具将源代码分解成一个个词素。（词素是描述编程语言语法的对象。它可以描述数字，标识符，标点符号，运算符等等。）
* 语法分析 —— parser     接收词素并将它们组合成一个描述了源代码各部分之间关系的中间表达形式：抽象语法树。（抽象语法树是一个深度嵌套的对象，这个对象以一种既能够简单地操作又提供很多关于源代码信息的形式，来展现代码。）

## 转换
* 这个过程接收解析生成的抽象语法树并对它做出改动
* 转换阶段可以改变抽象语法树使代码保持在同一个语言，或者编译成另外一门语言。

## 代码生成
* 生成新的代码，一般是二进制或者汇编

# aki-DSL解释器设计原理

## 解析源代码生成AST
那么想要实现一个脚本解释器的话，就需要实现上面的三个步骤，而且我们发现，承上启下的是AST（抽象语法树），它在解释器中十分重要

好在万能的golang将parse api暴露给用户了，可以让我们省去一大部分工作去做语法解析得到AST，示例代码如下：
```
package main

import (
	"fmt"
	"go/ast"
	"go/parser"
	"go/token"
)

func main() {
	expr := `a == 1 && b == 2`
	fset := token.NewFileSet()
	exprAst, err := parser.ParseExpr(expr)
	if err != nil {
		fmt.Println(err)
		return
	}

	ast.Print(fset, exprAst)
}
```
得到的结果：
```
     0  *ast.BinaryExpr {
     1  .  X: *ast.BinaryExpr {
     2  .  .  X: *ast.Ident {
     3  .  .  .  NamePos: -
     4  .  .  .  Name: "a"
     5  .  .  .  Obj: *ast.Object {
     6  .  .  .  .  Kind: bad
     7  .  .  .  .  Name: ""
     8  .  .  .  }
     9  .  .  }
    10  .  .  OpPos: -
    11  .  .  Op: ==
    12  .  .  Y: *ast.BasicLit {
    13  .  .  .  ValuePos: -
    14  .  .  .  Kind: INT
    15  .  .  .  Value: "1"
    16  .  .  }
    17  .  }
    18  .  OpPos: -
    19  .  Op: &&
    20  .  Y: *ast.BinaryExpr {
    21  .  .  X: *ast.Ident {
    22  .  .  .  NamePos: -
    23  .  .  .  Name: "b"
    24  .  .  .  Obj: *(obj @ 5)
    25  .  .  }
    26  .  .  OpPos: -
    27  .  .  Op: ==
    28  .  .  Y: *ast.BasicLit {
    29  .  .  .  ValuePos: -
    30  .  .  .  Kind: INT
    31  .  .  .  Value: "2"
    32  .  .  }
    33  .  }
    34  }
```
并且，作为一个嵌入式的DSL，我们的设计是依托在golang代码之上运行的，我们不需要代码生成这一个步骤，直接使用golang来解析AST来执行相应的操作

那么，我们的现在的工作就是如何解析AST并做相应的操作即可.

## 解析AST
### AST的结构分析
那么AST是什么结构呢，他大致可以分为如下结构

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20200702235816186_407645260.png)

### 1.ast.Decl 
All declaration nodes implement the Decl interface.
```
var a int //GenDecl
func main()  //FuncDecl
```

### 2.ast.Stmt
All statement nodes implement the Stmt interface.

```
a := 1 //AssignStmt
b := map[string]string{"name":"nber1994", "age":"eghiteen"}

if a > 2 { //IfStmt
	b["age"] = "18" //BlockStmt 
} else {
}

for i:=0;i<10;i++ { //ForStmt
}


for k, v := range b { //RangeStmt
}
return a //ReturnStmt
```


### 3.ast.Expr
All expression nodes implement the Expr interface.
```
a := 1 //BasicLit
b := "string"
a = a + 1 //BinaryExpr
b := map[string]string{} //CompositLitExpr
c := Get("test.test") //CallExpr
d := b["name"] //IndexExpr
```

## 主要思路
通过分析AST结构我们知道，一个ast.Decl是由多个ast.Stmt，并且一个ast.Stmt是由多个ast.Expr组成的，简单来说就是一个树形结构，那么这么一来就好办了，代码大框架一定是递归。

我们自底向上，分别实现对各种类型的ast.Expr，ast.Stmt, ast.Decl的解释执行方法，并把解释结果向上传递。然后通过一个根节点切入，递归方式从上向下解释执行即可

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20200702235906903_821050890.png)



主要代码：
```
//编译Expr
func (this *Expr) CompileExpr(dct *dslCxt.DslCxt, rct *Stmt, r ast.Expr) interface{} {
    var ret interface{}
    if nil == r {
        return ret
    }
    switch r := r.(type) {
    case *ast.BasicLit: //基本类型
        ret = this.CompileBasicLitExpr(dct, rct, r)
    case *ast.BinaryExpr: //二元表达式
        ret = this.CompileBinaryExpr(dct, rct, r)
    case *ast.CompositeLit: //集合类型
        switch  r.Type.(type) {
        case *ast.ArrayType: //数组
            ret = this.CompileArrayExpr(dct, rct, r)
        case *ast.MapType: //map
            ret = this.CompileMapExpr(dct, rct, r)
        default:
            panic("syntax error: nonsupport expr type")
        }
    case *ast.CallExpr:
        ret = this.CompileCallExpr(dct, rct, r)
    case *ast.Ident:
        ret = this.CompileIdentExpr(dct, rct, r)
    case *ast.IndexExpr:
        ret = this.CompileIndexExpr(dct, rct, r)
    default:
        panic("syntax error: nonsupport expr type")
    }
    return ret
}

//编译stmt
func (this *Stmt) CompileStmt(cpt *CompileCxt, stmt ast.Stmt) {
    if nil == stmt {
        return
    }
    cStmt := this.NewChild()
    switch stmt := stmt.(type) {
    case *ast.AssignStmt:
        //赋值在本节点的内存中
        this.CompileAssignStmt(cpt, stmt)
    case *ast.IncDecStmt:
        this.CompileIncDecStmt(cpt, stmt)
    case *ast.IfStmt:
        cStmt.CompileIfStmt(cpt, stmt)
    case *ast.ForStmt:
        cStmt.CompileForStmt(cpt, stmt)
    case *ast.RangeStmt:
        cStmt.CompileRangeStmt(cpt, stmt)
    case *ast.ReturnStmt:
        cStmt.CompileReturnStmt(cpt, stmt)
    case *ast.BlockStmt:
        cStmt.CompileBlockStmt(cpt, stmt)
    case *ast.ExprStmt:
        cStmt.CompileExprStmt(cpt, stmt)
    default:
        panic("syntax error: nonsupport stmt ")
    }
}
```

## 实现runtime context
代码的整体结构有了，那么对于DSL中声明的变量存储，以及局部变量的作用域怎么解决呢

首先，从虚拟内存的结构我们得到启发，可以使用hash表的结构来模拟最基本的内存空间以及存取操作，得益于golang的interface{}，我们可以把不同数据类型的数据存入一个map[string]interface{}中得到一个范类型的数组，这样我们就构建出了一个简单的runtime memory的雏形。

```
type RunCxt struct {
    Vars map[string]interface{}
    Name string
}

func NewRunCxt() *RunCxt{
    return &RunCxt{
        Vars: make(map[string]interface{}),
    }
}

//获取值
func (this *RunCxt) GetValue(varName string) interface{}{
    if _, exist := this.Vars[varName]; !exist {
        panic("syntax error: not exist var")
    }
    return this.Vars[varName]
}

func (this *RunCxt) ValueExist(varName string) bool {
    _, exist := this.Vars[varName]
    return exist
}

//设置值
func (this *RunCxt) SetValue(varName string, value interface{}) bool {
    this.Vars[varName] = value
    return true
}

func (this *RunCxt) ToString() string {
    jsonStu, _ := json.Marshal(this.Vars)
    return string(jsonStu)
}
```
那么，如何实现局部变量的作用域呢？

```
package main

func main() {
    a := 2

    for i:=0;i<10;i++ {
        a++
        b := 2
    }
    a = 3
    b = 3 //error b的声明是在for语句中，外部是无法访问的
}
```

那么，这个runtime context的位置就很重要，我们做如下处理：

每个Stmt节点都有一个runtime context
写入数据时，AssignStmt类型在本Stmt节点中赋值，其他类型新建一个Stmt子节点执行
读取数据时，从本节点开始向上遍历父节点，在runtime context中寻找变量，找到即止
通过这一机制，我们可以得到的效果是：

同一个BlockStmt下的多个Stmt（IfStmt，ForStmt等）处理节点之间的runtime context是互相隔离的
每个子节点，都能访问到父节点中定义的变量

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20200703000004061_50633564.png)


代码实现：

```
type Stmt struct{
    Rct *runCxt.RunCxt //变量作用空间
    Type int
    Father *Stmt //子节点可以访问到父节点的内存空间
}

func NewStmt() *Stmt {
    rct := runCxt.NewRunCxt()
    return &Stmt{
        Rct: rct,
    }
}

func (this *Stmt) NewChild() *Stmt {
    stmt := NewStmt()
    stmt.Father = this
    return stmt
}

//编译stmt
func (this *Stmt) CompileStmt(cpt *CompileCxt, stmt ast.Stmt) {
    if nil == stmt {
        return
    }
    cStmt := this.NewChild()
    switch stmt := stmt.(type) {
    case *ast.AssignStmt:
        //赋值在本节点的内存中
        this.CompileAssignStmt(cpt, stmt)
    case *ast.IncDecStmt:
        this.CompileIncDecStmt(cpt, stmt)
    case *ast.IfStmt:
        cStmt.CompileIfStmt(cpt, stmt)
    case *ast.ForStmt:
        cStmt.CompileForStmt(cpt, stmt)
    case *ast.RangeStmt:
        cStmt.CompileRangeStmt(cpt, stmt)
    case *ast.ReturnStmt:
        cStmt.CompileReturnStmt(cpt, stmt)
    case *ast.BlockStmt:
        cStmt.CompileBlockStmt(cpt, stmt)
    case *ast.ExprStmt:
        cStmt.CompileExprStmt(cpt, stmt)
    default:
        panic("syntax error: nonsupport stmt ")
    }
}
```


## 变量类型与内部变量类型
首先，嵌入式的是golang系统，为了和外部系统保持一个很好地数据类型交互以及数据的准确性，DSL最好也是强类型语言。但是为了简单，我们会删减一些数据类型，保留最基本且最稳定的数据类型
```
func (this *Expr) CompileBasicLitExpr(cpt *CompileCxt, rct *Stmt, r *ast.BasicLit) interface{} {
    var ret interface{}
    switch r.Kind {
    case token.INT:
        ret = cast.ToInt64(r.Value)
    case token.FLOAT:
        ret = cast.ToFloat64(r.Value)
    case token.STRING:
        retStr := cast.ToString(r.Value)
        var err error
        ret, err = strconv.Unquote(retStr)
        if nil != err {
            panic(fmt.Sprintf("syntax error: Bad String %v", cpt.Fset.Position(r.Pos())))
        }
    default:
        panic(fmt.Sprintf("syntax error: Bad BasicList Type %v", cpt.Fset.Position(r.Pos())))
    }
    return ret
}

func (this *Expr) CompileMapExpr(cpt *CompileCxt, rct *Stmt, r *ast.CompositeLit) interface{} {
    ret := make(map[interface{}]interface{})
    var key interface{}
    var value interface{}
    for _, e := range r.Elts {
        key = this.CompileExpr(cpt, rct, e.(*ast.KeyValueExpr).Key)
        value = this.CompileExpr(cpt, rct, e.(*ast.KeyValueExpr).Value)
        ret[key] = value
    }
    return ret
}

func (this *Expr) CompileArrayExpr(cpt *CompileCxt, rct *Stmt, r *ast.CompositeLit) interface{} {
    var ret []interface{}
    for _, e := range r.Elts {
        switch e := e.(type) {
        case *ast.BasicLit:
            ret = append(ret, this.CompileExpr(cpt, rct, e))
        case *ast.CompositeLit:
            //拼接结构体
            compLit := &ast.CompositeLit{
                Type: r.Type.(*ast.ArrayType).Elt,
                Elts: e.Elts,
            }
            ret = append(ret, this.CompileExpr(cpt, rct, compLit))
        default:
            panic(fmt.Sprintf("syntax error: Bad Array Item Type %v", cpt.Fset.Position(r.Pos())))
        }
    }
    return ret
}
```

我们可以看到，DSL数据与go数据类型对应关系为：

| DSL数据类型  |          go数据类型          |   备注    |
| ----------- | --------------------------- | -------- |
| int         | int64                       | 最大范围   |
| float       | float64                     | 最大范围   |
| string      | string                      |          |
| map         | map[interface{}]interface{} | 最大容忍度 |
| array slice | []interface{}{}             | 最大容忍度 |


## DSL与外部系统交互
通过JsonMap与外部系统进行交互，且提供Get(path) Set(path)方法，去动态的访问与修改Json context中的节点

但是外部交互Json又是多种结构类型的，借助于nodejson可以解析动态json结构，通过XX.X格式的路径，来动态的访问和修改json中的字段

解析CallExpr，通过reflect来调用内部函数
```
func (this *Expr) CompileCallExpr(dct *dslCxt.DslCxt, rct *Stmt, r *ast.CallExpr) interface{} {
    var ret interface{}
    //校验内置函数
    var funcArgs []reflect.Value
    funcName := r.Fun.(*ast.Ident).Name
    //初始化入参
    for _, arg := range r.Args {
        funcArgs = append(funcArgs, reflect.ValueOf(this.CompileExpr(dct, rct, arg)))
    }
    var res []reflect.Value
    if RealFuncName, exist:= SupFuncList[funcName]; exist {
        flib := NewFuncLib()
        res = reflect.ValueOf(flib).MethodByName(RealFuncName).Call(funcArgs)
    } else {
        res = reflect.ValueOf(dct).MethodByName(funcName).Call(funcArgs)
    }
    if nil == res {
        return ret
    }
    return res[0].Interface()
}
```

# 成果
[https://github.com/nber1994/akiDsl](https://github.com/nber1994/akiDsl)




