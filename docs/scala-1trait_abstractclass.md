# Scala trait 和 abstract class 的区别
## Trait:
> scala 没有接口，但是trait比接口强大;
> 一个类可以继承多个 trait，用with连接;
> trait中定义的方法可以实现，也可以没有实现;
> trait中可以定义具体字段和抽象字段;
> 使用trait实现模板模式: 在一个特质中，具体方法依赖于抽象方法，而抽象方法可以放到继承trait的子类中实现，这种设计方式也称为模板模式;

```scala
trait Logger {
    def log(msg:String)
    def info(msg:String) = log("INFO:" + msg)
    def warn(msg:String) = log("WARN:" + msg)
    def error(msg:String) = log("ERROR:" + msg)
  }

  class ConsoleLogger extends Logger {
    override def log(msg: String): Unit = {
      println(msg)
    }
  }

  def main(args: Array[String]): Unit = {
    val logger = new ConsoleLogger
    logger.info("信息日志")
    logger.warn("警告日志")
    logger.error("错误日志")
  }
```
> 对象混入trait
>> scala中可以将trait混入到对象中，就是将trait中定义的方法、字段添加到一个对象中
>> 语法: val/var 对象名 = new 类 with 特质
```scala
trait Logger {
    def log(msg:String) = println(msg)
  }

  class UserService

  def main(args: Array[String]): Unit = {
    val service = new UserService with Logger
    service.log("混入的方法")
  }
```
> trait实现调用链模式
>> 类继承了多个trait后，可以依次调用多个trait中的同一个方法，只要让多个trait中的同一个方法在最后都依次执行super关键字即可。类中调用多个tait中都有这个方法时，首先会从最右边的trait方法开始执行，然后依次往左执行，形成一个调用链条。
```scala
trait HandlerTrait {
   def handle(data:String) = println("处理数据...")
}
​
trait DataValidHanlderTrait extends HandlerTrait {
   override def handle(data:String): Unit = {
       println("验证数据...")
       super.handle(data)
  }
}
trait SignatureValidHandlerTrait extends HandlerTrait {
   override def handle(data: String): Unit = {
       println("校验签名...")
       super.handle(data)
  }
}
class PayService extends DataValidHanlderTrait with SignatureValidHandlerTrait {
   override def handle(data: String): Unit = {
       println("准备支付...")
       super.handle(data)
  }
}
def main(args: Array[String]): Unit = {
   val service = new PayService
   service.handle("支付参数")
}
// 程序运行输出如下：
// 准备支付...
// 检查签名...
// 验证数据...
// 处理数据...
```
> trait继承class
>> trait也可以继承class的。特质会将class中的成员都继承下来。
```scala
class MyUtil {
   def printMsg(msg:String) = println(msg)
}
trait Logger extends MyUtil {
   def log(msg:String) = printMsg("Logger:" + msg)
}
class Person extends Logger {
   def sayHello() = log("你好")
}
def main(args: Array[String]): Unit = {
   val person = new Person
   person.sayHello()
}
```
> 为什么Java中采用多接口而没有多继承???
>> 接口只定义了你做什么，而不关心你怎么做；
>> 如果是多继承，有可能两个类定义的两个方法是做同一件事情的，那么他们的子类就无法选择到底去用哪一个。

> 接口、抽象类区别：
>> interface强调特定功能的实现，具有哪些功能，而abstract class强调所属关系.
>> interface是完全抽象的，只能声明方法，而且只能声明public的方法，不能声明private及protected的方法，不能定义方法体，也不能声明实例变量。

## Abstract Class
> 抽象类只能单继承
> 抽象类于java抽象类类似，可以有抽象方法，也可以有非抽象方法；
> 抽象类有带参数的构造函数，trait不可以
