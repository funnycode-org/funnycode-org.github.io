---
title: "闲聊设计模式 | 模板方法模式"
subtitle: ""
date: 2019-11-09T19:00:33+08:00
lastmod: 2020-03-09T16:25:11+08:00
draft: false
author: "铁城"
authorLink: ""
description: "本文介绍设计模式之模板方法模式"

tags: [java,golang,模板方法模式]
categories: [设计模式]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

<!--more-->

## 一、前言

[Template Pattern 模板方法](https://zh.wikipedia.org/wiki/%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95) 来自 Wiki 百科的介绍：

> **模板方法模型**是一种行为设计模型。**模板方法**是一个定义在父类别的方法，在**模板方法**中会呼叫多个定义在父类别的其他方法，而这些方法有可能只是抽象方法并没有实作，**模板方法**仅决定这些抽象方法的执行顺序，这些抽象方法的实作由子类别负责，并且子类别不允许覆写模板方法。

模板方法是属于设计模式的*行为型模式*

模板方法模式按照我的白话文理解：

首先定一个“抽象类”，它有一个模板方法A，定义可能需要子类实现的方法B，C，D...，然后在A方法里面编排好了B，C，D等的位置，当实现类调用A的时候，会调用各自实现的B，C，D方法（某些方法有默认实现，子类可以不实现），从而在一样的大流程里面进行不一样的操作。

## 二、简单实例

**故事背景**

阳光明媚的一天，玩码部落来了一群腿长一米八的MM，它们来自台湾，杭州及北京，她们将介绍各自家乡是如何准备丰富的晚餐的。

### 2.1 Java版本

#### 2.1.1 定义接口

```java
public interface Dinner {

    /**
     * 晚餐
     */
    void doDinner();

}
```

定义一个接口，就一个无参的 `void` 方法。

#### 2.1.2 定义抽象方法

```java
public abstract class AbstractDinner implements Dinner {

    protected String name;

    public AbstractDinner(String name) {
        this.name = name;
    }

    private void eat() {
        System.out.printf("%sMM说：开吃喽", name).println();
    }

    protected boolean foodEnough() {
        return true;
    }

    protected void doShopping() {
        System.out.println("门口小贩买菜");
    }

    protected abstract void beforeCooking();

    protected abstract String doCooking();

    protected abstract void afterCooking();

    @Override
    public void doDinner() {
        if (!foodEnough()) {
            doShopping();
        }

        beforeCooking();
        System.out.println(doCooking());
        afterCooking();

        eat();
    }

}
```

定义 `AbstractDinner` 实现接口，它自身有五个方法，默认实现的 `foodEnough` 和 `doShopping`，以及抽象方法 `beforeCooking`、`doCooking` 和 `afterCooking`。

`doDinner ` 编排了这些方法的流程或者说定义了各阶段的步骤。

#### 2.1.3 定义实现类

```java
public class BeijingDinner extends AbstractDinner {

    public BeijingDinner(String name) {
        super(name);
    }

    @Override
    protected void beforeCooking() {
        System.out.printf("%sMM 在洗菜切菜", name).println();
    }

    @Override
    protected String doCooking() {
        return name + "MM 在做" + name + "菜";
    }

    @Override
    protected void afterCooking() {
        System.out.printf("%sMM 让你去品尝", name).println();
    }

}

public class TaiwanDinner extends AbstractDinner {

    public TaiwanDinner(String name) {
        super(name);
    }

    @Override
    protected boolean foodEnough() {
        // 每次都买食物
        return false;
    }

    @Override
    protected void doShopping() {
        System.out.println("生鲜超市购买，一定要买茶叶蛋");
    }

    @Override
    protected void beforeCooking() {
        System.out.printf("%sMM 在洗菜切菜", name).println();
    }

    @Override
    protected String doCooking() {
        return name + "MM 在做" + name + "菜";
    }

    @Override
    protected void afterCooking() {
        System.out.printf("%sMM 让你去品尝", name).println();
    }

}

public class HangzhouDinner extends AbstractDinner {

    public HangzhouDinner(String name) {
        super(name);
    }

    @Override
    protected boolean foodEnough() {
        // 每次都买食物
        return false;
    }

    @Override
    protected void beforeCooking() {
        System.out.printf("%sMM 在洗菜切菜", name).println();
    }

    @Override
    protected String doCooking() {
        return name + "MM 在做" + name + "菜";
    }

    @Override
    protected void afterCooking() {
        System.out.printf("%sMM 让你去品尝", name).println();
    }

}
```

定义了三个实现类，都实现了3个抽象方法。另外 `TaiwanDinner` 重写了另外两个方法，`HangzhouDinner` 只重写了 `foodEnough`。

#### 2.1.4 运行例子

代码：

```java
public class DinnerDemo {

    public static void main(String[] args) {
        System.out.println("---准备台湾餐---");
        Dinner dinner1 = new TaiwanDinner();
        dinner1.doDinner();
        System.out.println("---准备杭州餐---");
        Dinner dinner2 = new HangzhouDinner();
        dinner2.doDinner();
        System.out.println("---准备北京餐---");
        Dinner dinner3 = new BeijingDinner();
        dinner3.doDinner();
    }

}
```

输出结果：

```text
---准备台湾餐---
生鲜超市购买，一定要买茶叶蛋
台湾MM 在洗菜切菜
台湾MM 在做台湾菜
台湾MM 让你去品尝
台湾MM说：开吃喽
---准备杭州餐---
门口小贩买菜
杭州MM 在洗菜切菜
杭州MM 在做杭州菜
杭州MM 让你去品尝
杭州MM说：开吃喽
---准备北京餐---
北京MM 在洗菜切菜
北京MM 在做北京菜
北京MM 让你去品尝
北京MM说：开吃喽
```

### 2.2 Golang 版本

> 声明下：在 golang 中，由于不存在抽象类和真正的继承，所以只能通过一个基础类来充当抽象类，子类通过组合基础类来实现通用方法的继承。

#### 2.2.1 定义接口

```go
type Dinner interface {
	DoDinner()
}
```

#### 2.2.2 定义抽象类

```go
type AbstractCooking struct {
	foodEnough    func() bool
	doShopping    func()
	beforeCooking func()
	doCooking     func() string
	afterCooking  func()
	Name          string
}

func (d *AbstractCooking) DoDinner() {
	if !d.foodEnough() {
		d.doShopping()
	}
	d.beforeCooking()
	fmt.Println(d.doCooking())
	d.afterCooking()
	d.eat()
}

func (d *AbstractCooking) eat() {
	fmt.Println(fmt.Sprintf("%sMM说：开吃喽", d.Name))
}
```

这里和 Java 不一样的地方是 go 的结构体可以拥有 func() 属性（也可以拥有接口属性）。

实现 `Dinner` 的方法 `DoDinner` , 编排了一系列的方法。

#### 2.2.3 定义实现类

```go
type HZDinner struct {
	AbstractCooking
}

func NewHZDinner(name string) *HZDinner {
	c := new(HZDinner)
	c.Name = name
	// 选择实现的
	c.AbstractCooking.foodEnough = c.foodEnough
	c.AbstractCooking.doShopping = doShopping
	// 必须实现的
	c.AbstractCooking.beforeCooking = c.beforeCooking
	c.AbstractCooking.doCooking = c.doCooking
	c.AbstractCooking.afterCooking = c.afterCooking
	return c
}

func (c *HZDinner) foodEnough() bool {
	return false
}

func (c *HZDinner) beforeCooking() {
	println(fmt.Printf("%sMM 在洗菜切菜", c.Name))
}

func (c *HZDinner) doCooking() string {
	return fmt.Sprintf("%sMM 在做%s菜", c.Name, c.Name)
}

func (c *HZDinner) afterCooking() {
	println(fmt.Printf("%sMM 让你去品尝", c.Name))
}

type TWDinner struct {
	AbstractCooking
}

func NewTWDinner(name string) *TWDinner {
	c := new(TWDinner)
	c.Name = name
	// 选择实现的
	c.AbstractCooking.foodEnough = c.foodEnough
	c.AbstractCooking.doShopping = c.doShopping
	// 必须实现的
	c.AbstractCooking.beforeCooking = c.beforeCooking
	c.AbstractCooking.doCooking = c.doCooking
	c.AbstractCooking.afterCooking = c.afterCooking
	return c
}

func (c *TWDinner) foodEnough() bool {
	return false
}

func (c *TWDinner) doShopping() {
	fmt.Println("生鲜超市购买，一定要买茶叶蛋")
}

func (c *TWDinner) beforeCooking() {
	println(fmt.Printf("%sMM 在洗菜切菜", c.Name))
}

func (c *TWDinner) doCooking() string {
	return fmt.Sprintf("%sMM 在做%s菜", c.Name, c.Name)
}

func (c *TWDinner) afterCooking() {
	println(fmt.Printf("%sMM 让你去品尝", c.Name))
}

type BJDinner struct {
	AbstractCooking
}

func NewBJDinner(name string) *BJDinner {
	c := new(BJDinner)
	c.Name = name
	// 选择实现的
	c.AbstractCooking.foodEnough = foodEnough
	c.AbstractCooking.doShopping = doShopping
	// 必须实现的
	c.AbstractCooking.beforeCooking = c.beforeCooking
	c.AbstractCooking.doCooking = c.doCooking
	c.AbstractCooking.afterCooking = c.afterCooking
	return c
}

func (c *BJDinner) beforeCooking() {
	println(fmt.Printf("%sMM 在洗菜切菜", c.Name))
}

func (c *BJDinner) doCooking() string {
	return fmt.Sprintf("%sMM 在做%s菜", c.Name, c.Name)
}

func (c *BJDinner) afterCooking() {
	println(fmt.Printf("%sMM 让你去品尝", c.Name))
}
```

#### 2.2.4 定义默认实现方法

```go
func foodEnough() bool {
	return true
}

func doShopping() {
	fmt.Println("门口小贩买菜")
}
```

为什么有独立的默认方法，因为 struct 里面定义了 `foodEnough() bool` 和 `doShopping()` 两个方法，go 里面是不能重名的，因此不能再写属于 `AbstractCooking` 的方法。

```go
func (d *AbstractCooking) foodEnough() bool {
	return true
}
```

这个方法如何写了，去掉了 struct 的 `foodEnough() bool`，那么创建实现类的时候就没办法 `c.AbstractCooking.foodEnough = c.foodEnough` 进行 func() 赋值，从而 `d.foodEnough()` 会一直调用 `AbstractCooking` 下的 `foodEnough()`，实现类没办法自定义实现了。

#### 2.2.5 运行例子

代码：

```go
func TestTemplate1(t *testing.T) {
	fmt.Println("---准备台湾餐---")
	d1 := NewTWDinner("台湾")
	d1.DoDinner()
	fmt.Println("---准备杭州餐---")
	d2 := NewHZDinner("杭州")
	d2.DoDinner()
	fmt.Println("---准备北京餐---")
	d3 := NewBJDinner("北京")
	d3.DoDinner()
}
```

输出结果：

```text
---准备台湾餐---
生鲜超市购买，一定要买茶叶蛋
台湾MM 在洗菜切菜
台湾MM 在做台湾菜
台湾MM 让你去品尝
台湾MM说：开吃喽
---准备杭州餐---
门口小贩买菜
杭州MM 在洗菜切菜
杭州MM 在做杭州菜
杭州MM 让你去品尝
杭州MM说：开吃喽
---准备北京餐---
北京MM 在洗菜切菜
北京MM 在做北京菜
北京MM 让你去品尝
北京MM说：开吃喽
```

### 2.3 Golang版本2

> 上面例子是在 struct 中定义，相当于是抽象方法的意思，而这版本把那部分方法都定义到了接口。

#### 2.3.1 定义接口

```go
type Dinner2 interface {
	foodEnough() bool
	doShopping()
	beforeCooking()
	doCooking() string
	afterCooking()
}
```

#### 2.3.2 定义抽象类

```go
type AbstractDinner struct {
}

func (AbstractDinner) foodEnough() bool {
	return true
}

func (AbstractDinner) doShopping() {
	fmt.Println("门口小贩买菜")
}

func (AbstractDinner) beforeCooking() {
}

func (AbstractDinner) doCooking() string {
	return ""
}

func (AbstractDinner) afterCooking() {
}
```

实现 `Dinner2` 接口，和下面等价

```go
type AbstractDinner struct {
	Dinner2
}

func (AbstractDinner) foodEnough() bool {
	return true
}

func (AbstractDinner) doShopping() {
	fmt.Println("门口小贩买菜")
}
```

#### 2.3.3 定义实现类

```go
type HangzhouDinner struct {
	AbstractDinner
}

func NewHangzhouDinner(name string) Dinner2 {
	return &HangzhouDinner{
		AbstractDinner{
			Name: name,
		},
	}
}

func (d *HangzhouDinner) foodEnough() bool {
	return false
}

func (d *HangzhouDinner) beforeCooking() {
	fmt.Println(fmt.Sprintf("%sMM 在洗菜切菜", d.Name))
}

func (d *HangzhouDinner) doCooking() string {
	return fmt.Sprintf("%sMM 在做%s菜", d.Name, d.Name)
}

func (d *HangzhouDinner) afterCooking() {
	fmt.Println(fmt.Sprintf("%sMM 让你去品尝", d.Name))
}

type BeijingDinner struct {
	AbstractDinner
}

func NewBeijingDinner(name string) Dinner2 {
	return &BeijingDinner{
		AbstractDinner{
			Name: name,
		},
	}
}

func (d *BeijingDinner) beforeCooking() {
	fmt.Println(fmt.Sprintf("%sMM 在洗菜切菜", d.Name))
}

func (d *BeijingDinner) doCooking() string {
	return fmt.Sprintf("%sMM 在做%s菜", d.Name, d.Name)
}

func (d *BeijingDinner) afterCooking() {
	fmt.Println(fmt.Sprintf("%sMM 让你去品尝", d.Name))
}

type TaiwanDinner struct {
	AbstractDinner
}

func NewTaiwanDinner(name string) Dinner2 {
	return &TaiwanDinner{
		AbstractDinner{
			Name: name,
		},
	}
}

func (d *TaiwanDinner) foodEnough() bool {
	return false
}

func (d *TaiwanDinner) doShopping() {
	fmt.Println("生鲜超市购买，一定要买茶叶蛋")
}

func (d *TaiwanDinner) beforeCooking() {
	fmt.Println(fmt.Sprintf("%sMM 在洗菜切菜", d.Name))
}

func (d *TaiwanDinner) doCooking() string {
	return fmt.Sprintf("%sMM 在做%s菜", d.Name, d.Name)
}

func (d *TaiwanDinner) afterCooking() {
	fmt.Println(fmt.Sprintf("%sMM 让你去品尝", d.Name))
}
```

#### 2.3.4 定义模板方法

````go
func DoDinner(d Dinner2) {
	if !d.foodEnough() {
		d.doShopping()
	}

	d.beforeCooking()
	fmt.Println(d.doCooking())
	d.afterCooking()
  d.eat()
}

func (ad AbstractDinner) eat() {
	fmt.Println(fmt.Sprintf("%sMM说：开吃喽", ad.Name))
}
````

如果想把抽象方法放到结构体上，也可以如下：

```go
type Dinner2 interface {
	...
	DoDinner(d Dinner2)
}

func (ad AbstractDinner) DoDinner(d Dinner2) {
	if !d.foodEnough() {
		d.doShopping()
	}

	d.beforeCooking()
	fmt.Println(d.doCooking())
	d.afterCooking()
	ad.eat()
}

func (ad AbstractDinner) eat() {
	fmt.Println(fmt.Sprintf("%sMM说：开吃喽", ad.Name))
}
```

到时候调用的话就 DoDinner 改成 `d1 := NewTaiwanDinner("台湾") d1.DoDinner(d1)`。

#### 2.3.5 运行例子

代码：

```go
func TestTemplate2(t *testing.T) {
	fmt.Println("---准备台湾餐---")
	d1 := NewTaiwanDinner("台湾")
	DoDinner(d1)
	fmt.Println("---准备杭州餐---")
	d2 := NewHangzhouDinner("杭州")
	DoDinner(d2)
	fmt.Println("---准备北京餐---")
	d3 := NewTaiwanDinner("北京")
	DoDinner(d3)
}
```

输出结果：

```text
---准备台湾餐---
生鲜超市购买，一定要买茶叶蛋
台湾MM 在洗菜切菜
台湾MM 在做台湾菜
台湾MM 让你去品尝
台湾MM说：开吃喽
---准备杭州餐---
门口小贩买菜
杭州MM 在洗菜切菜
杭州MM 在做杭州菜
杭州MM 让你去品尝
杭州MM说：开吃喽
---准备北京餐---
生鲜超市购买，一定要买茶叶蛋
北京MM 在洗菜切菜
北京MM 在做北京菜
北京MM 让你去品尝
北京MM说：开吃喽
```

### 2.4、例子说明



## 三、开源框架使用场景

> 列举某几个框架，供大家参考

### 3.1 JDK AbstractList

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
  	...
    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }  
    
  	...
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);
        boolean modified = false;
        for (E e : c) {
            add(index++, e);
            modified = true;
        }
        return modified;
    }  
  	...
}
```

实现类实现 add 的逻辑，如：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
  	// ...
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
  	// ...
}        

```

### 3.1 spring 中的 事务管理器 

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
 
  // ...
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
		Object transaction = doGetTransaction();

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}

		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException ex) {
				resume(null, suspendedResources);
				throw ex;
			}
			catch (Error err) {
				resume(null, suspendedResources);
				throw err;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
  // ...
    
	protected abstract void doBegin(Object transaction, TransactionDefinition definition)
			throws TransactionException;    
  
  ...
}
```

这里只拿 `getTransaction` 和 `doBegin` 举例，非常标准的写法 `getTransaction` 还用了 final 描述，表示子类不允许改变。

当我自定义事务管理器的时候，比如每次事务开启创建一个 traceId，效果如下：

```java
public class GongDaoDataSourceTransactionManager extends DataSourceTransactionManager implements
    ResourceTransactionManager, InitializingBean, EnvironmentAware {
  
  	// ...
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        String currentXid;
        if (XIDContext.getCurrent() != null && null != XIDContext.getCurrent().getId()) {
            currentXid = xidGenerator.getXID();
            XIDContext.childCurrent(new TransactionContent(currentXid, definition.getName()));
        } else {
            currentXid = xidGenerator.getXID();
            XIDContext.setXid(new TransactionContent(currentXid, definition.getName()));
        }

        if (null == HyjalTransactionFileAppender.contextHolder.get()) {
            String logFileName = HyjalTransactionFileAppender.makeLogFileName(new Date(),
                HyjalTransactionFileAppender.logId.get());
            HyjalTransactionFileAppender.contextHolder.set(logFileName);
        }

        try {
            if (null == XIDContext.getCurrentXid()) {
                XIDContext.setXid(xidGenerator.getXID());
            }
            super.doBegin(transaction, definition);
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("do begin Xid : {}", currentXid);
                HyjalTransactionLogger.log("do begin Xid : {}, obj : {}", currentXid, definition.getName());
            }
        }
    }
  	// ...
}  
```

> 欢迎各位补充更多的例子和场景

## 四、优势和劣势

### 4.1 优势

- 对扩展开放，对修改关闭，符合“开闭原则”
  - 定义标准算法，子类可自定义扩展，把变性和不变性分离
  - 子类的扩展不会导致标准算法结构
- 能够提高代码复用，公共部分易维护

### 4.2 劣势

- 每一个实现类都需要定义自己的行为，如果复杂业务实现类会膨胀的比较多

## 五、参考

[https://www.tutorialspoint.com/design_pattern/template_pattern.htm](https://www.tutorialspoint.com/design_pattern/template_pattern.htm)

[https://zh.wikipedia.org/wiki/%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95](https://zh.wikipedia.org/wiki/%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95)