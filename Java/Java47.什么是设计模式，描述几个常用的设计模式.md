# 什么是设计模式，描述几个常用的设计模式

|  **类型**  |                    **设计模式（Github）**                    | **常用** |                           **博客**                           |
| :--------: | :----------------------------------------------------------: | :------: | :----------------------------------------------------------: |
| **创建型** | [单例模式(Singleton Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/01_singleton) |    ✅     | [Go 设计模式 01-单例模式](https://lailin.xyz/post/singleton.html) |
|            | [工厂模式(Factory Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/02_factory) |    ✅     | [Go 设计模式 02-工厂模式&DI 容器](https://lailin.xyz/post/factory.html) |
|            | [建造者模式(Builder Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/03_builder) |    ✅     | [Go 设计模式 03-建造者模式](https://lailin.xyz/post/builder.html) |
|            | [原型模式(Prototype Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/04_prototype) |    ❌     | [Go 设计模式 04-原型模式](https://lailin.xyz/post/prototype.html) |
| **结构型** | [代理模式(Proxy Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/05_proxy) |    ✅     | [Go 设计模式 06-代理模式(generate 实现类似动态代理)](https://lailin.xyz/post/proxy.html) |
|            | [桥接模式(Bridge Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/06_bridge) |    ✅     | [Go 设计模式 07-桥接模式](https://lailin.xyz/post/bridge.html) |
|            | [装饰器模式(Decorator Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/07_decorator) |    ✅     | [Go 设计模式 08-装饰器模式](https://lailin.xyz/post/decorator.html) |
|            | [适配器模式(Adapter Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/08_adapter) |    ✅     | [Go 设计模式 09-适配器模式](https://lailin.xyz/post/adapter.html) |
|            | [门面模式(Facade Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/09_facade) |    ❌     | [Go 设计模式 10-门面模式](https://lailin.xyz/post/facade.html) |
|            | [组合模式(Composite Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/10_composite) |    ❌     | [Go 设计模式 11-组合模式](https://lailin.xyz/post/composite.html) |
|            | [享元模式(Flyweight Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/11_flyweight) |    ❌     | [Go 设计模式 12-享元模式](https://lailin.xyz/post/flyweight.html) |
| **行为型** | [观察者模式(Observer Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/12_observer) |    ✅     | [Go 设计模式 13-观察者模式(实现简单的 EventBus)](https://lailin.xyz/post/observer.html) |
|            | [模板模式(Template Method Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/13_template) |    ✅     | [Go 模板模式 14-模板模式](https://lailin.xyz/post/template.html) |
|            | [策略模式(Strategy Method Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/14_strategy) |    ✅     | [Go 设计模式 15-策略模式](https://lailin.xyz/post/strategy.html) |
|            | [职责链模式(Chain Of Responsibility Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/15_chain) |    ✅     | [Go 设计模式 16-职责链模式(Gin 的中间件实现)](https://lailin.xyz/post/chain.html) |
|            | [状态模式(State Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/16_state) |    ✅     | [Go 设计模式 17-状态模式](https://lailin.xyz/post/state.html) |
|            | [迭代器模式(Iterator Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/17_iterator) |    ✅     | [Go 设计模式 18-迭代器模式](https://lailin.xyz/post/iterator.html) |
|            | [访问者模式(Visitor Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/18_visitor/visitor.go) |    ❌     | [Go 设计模式 19-访问者模式](https://lailin.xyz/post/visitor.html) |
|            | [备忘录模式(Memento Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/19_memento) |    ❌     | [Go 设计模式 20-备忘录模式](https://lailin.xyz/post/memento.html) |
|            | [命令模式(Command Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/20_command) |    ❌     | [Go 设计模式 21-命令模式](https://lailin.xyz/post/command.html) |
|            | [解释器模式(Interpreter Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/21_interpreter) |    ❌     | [Go 设计模式 22-解释器模式](https://lailin.xyz/post/interpreter.html) |
|            | [中介模式(Mediator Design Pattern)](https://github.com/mohuishou/go-design-pattern/blob/master/22_mediator) |    ❌     | [Go 设计模式 23-中介模式](https://lailin.xyz/post/mediator.html) |

### 单例模式

单例模式采用了 饿汉式 和 懒汉式 两种实现，个人其实更倾向于饿汉式的实现，简单，并且可以将问题及早暴露，懒汉式虽然支持延迟加载，但是这只是把冷启动时间放到了第一次使用的时候，并没有本质上解决问题，并且为了实现懒汉式还不可避免的需要加锁。