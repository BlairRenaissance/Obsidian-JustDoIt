# 类图

## 箭头类型

UML类图有六种箭头类型：继承、实现、关联、聚合、组合、依赖。

将箭头分为继承相关（继承、实现）、关联相关（关联、组合、聚合）、临时关系（依赖）：
- 继承关系：实线三角继承，虚线三角实现
- 结构关系：实心菱形组合，空心菱形聚合
- 强关系和临时关系：实线关联，虚线依赖

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/20250324114146685.png)

- 车的类图结构为`<<abstract>>`，表示车是一个抽象类；
- 它有两个继承类：小汽车和自行车；它们之间的关系为实现关系，使用带空心箭头的虚线表示；
- 小汽车为与SUV之间也是继承关系，使用带空心箭头的实线表示；
- 小汽车与发动机之间是组合关系，使用带实心箭头的实线表示；
- 学生与班级之间是聚合关系，使用带空心箭头的实线表示；
- 学生与身份证之间为关联关系，使用一根实线表示；
- 学生上学需要用到自行车，与自行车是一种依赖关系，使用带箭头的虚线表示；

**易混淆解释**

关联：想象两个人用实心铁链绑在一起（长期连接）
依赖：想象临时用虚线绳子拉东西（用完即断）

## 箭头绘制

1. 继承关系：  使用 `<|--` 表示继承关系。  

```
class ParentClass  
class ChildClass  
ParentClass <|-- ChildClass  
```

2. 实现关系：  使用 `<|..` 表示实现接口的关系。  

```
interface MyInterface  
class MyClass  
MyInterface <|.. MyClass  
```

3. 组合关系：  使用 `*--` 表示组合关系。  
 
```
class Whole  
class Part  
Whole *-- Part  
```

4. 聚合关系：  使用 `o--` 表示聚合关系。  

```
class Container  
class Component  
Container o-- Component  
```

5. 关联关系：  使用 `<--` 表示关联关系。  

```
class ClassA  
class ClassB  
ClassA <-- ClassB : association
```


6. 依赖关系：  使用 `<..` 表示依赖关系。  

```
class Client  
class Service  
Client ..> Service : uses
```

## 枚举类

在PlantUML中，可以使用 `enum` 关键字来定义枚举类。以下是一个示例：

```
@startuml 
enum Status {     
	NEW     
	IN_PROGRESS     
	COMPLETED 
}

class Task { 
	-name: String 
	-status: Status 
	+getStatus(): Status 
} 

Task --> Status : has
@enduml
```

## 注释

**方法一：使用注释**

在类的定义中添加注释来表示文件名：  
```
@startuml  
  
class User {  
+name: String  
+email: String  
+login(): void  
' 文件名注释  
' @file: User.java  
}  
  
@enduml  
```
在类的定义中使用单引号（'）添加注释，表示文件名。这种方法简单直接，但在图形化展示中不可见。  

**方法二：使用标签**

使用PlantUML的 note 标签来为类添加文件名信息：  
```
@startuml  
  
class User {  
+name: String  
+email: String  
+login(): void  
}  
  
note right of User  
File: User.java  
end note  
  
@enduml  
```
使用 note 标签可以在类图中以可视化的方式展示文件名。note 标签可以放置在类的左侧、右侧、上方或下方（使用 left of、right of、top of、bottom of）。


# 时序图

## 关键字

### `activate` & `deactivate` 

在 UML 序列图中，activate 和 deactivate 用于表示对象的激活期（Activation Period）。

**`activate`**
- 在对象生命线上标记一个长条（矩形），表示对象开始执行某个方法（进入激活状态）。
- 对应方法开始执行的时间段。

**`deactivate`**
- 结束激活状态，表示对象完成当前方法的执行（退出激活状态）。
- 对应方法执行结束的时间点。

**示例**
```
activate SrcDataMacro4K
SrcDataMacro4K -> SrcDataMacro4K : createRenderObject()
SrcDataMacro4K -> VectorRoadMacro4K : VectorRoadMacro4K()

activate VectorRoadMacro4K
VectorRoadMacro4K -> VectorRoadMacro4K : CheckRenderUnits()

activate VectorRoadMacro4K
VectorRoadMacro4K -> VectorRoadMacro4K : UpdatePassLaneGroup()
VectorRoadMacro4K -> VectorRoadMacro4K : BuildRenderUnits()

deactivate VectorRoadMacro4K
deactivate SrcDataMacro4K
```