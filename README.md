
肉夹馍([https://github.com/inversionhourglass/Rougamo](https://github.com))，一款编译时AOP组件。相比动态代理AOP需要在应用启动时进行初始化，编译时完成代码编织的肉夹馍减少了应用启动初始化的时间，同时肉夹馍还支持所有种类的方法，无论方法是同步还是异步、静态还是实例、构造方法还是属性都是支持的。肉夹馍无需初始化，编写好切面类型后直接应用到对应方法上即可，同时肉夹馍还提供了方法特征匹配和类AspectJ表达式匹配的批量应用规则。


# 前言


肉夹馍已经步入第五个大版本了，主要功能及优化已基本补全，在该版本之后，将在很长一段时间里不再发布新的大版本。如果你还在考虑肉夹馍是否值得一试，不妨看看 PostSharp 精选的“2024主要AOP框架”([https://www.postsharp.net/solutions/aspect\-oriented\-programming\#list\-of\-aop\-frameworks\-for.net](https://github.com))


是的，肉夹馍也在其列。对于这一结果，我个人是非常惊讶的，其他入选框架在 nuget.org 上的下载量都是肉夹馍的几十上百倍，而且还有很多高下载量的框架没有入选（[MethodBoundaryAspect.Fody](https://github.com), [MethodDecorator.Fody](https://github.com),
[MrAdvice](https://github.com) 等）。另外，对于 PostSharp 能够发现肉夹馍这个项目，我也是感觉非常意外的，肉夹馍从未在外网做过宣发，同时由于初建项目时的灵机一动，给项目取了"Rougamo"这么个怪名字，导致在各种搜索引擎上不论搜索"Rougamo"还是"AOP"，都很难见到我这个肉夹馍。


如果你认可 PostSharp，那么你也可以选择尝试肉夹馍。另外，肉夹馍在2024年连续发布了三个大版本，现在肉夹馍的实际表现要远远超出 PostSharp 撰写 [aspect\-oriented\-programming](https://github.com) 的时候。随着 5\.0 版本的发布，大版本或将长期稳定在 5\.x，现在便是入手的最佳时机。


# 5\.0


好了，王婆卖瓜环节结束，现在进入正文。


5\.0 的主要内容是优化，由于本次优化对切面类型和`MethodContext`的基本结构都有改动，无法兼容 4\.0 及之前的版本，所以作为一个大版本发布。当然，除了优化还提供了一些新的功能，欢迎阅读全文了解更多。


## 性能优化


### 切面类型属性成员缩减


目前的切面类型包含众多属性，这些属性均为配置项，基本只在编译时供肉夹馍使用，在运行时并不需要，而切面类型在实例化时这些属性都会占用内存空间，所以 5\.0 版本将删除切面类型中的所有属性成员，并提供对应的 Attribute 和接口，用于实现相同的功能。


在介绍具体改动之前，先再次回顾一下切面类型的基本情况。所有切面类型均实现`IMo`接口，所以删除切面类型中的所有属性成员，也就是删除`IMo`中定义的所有属性成员，以下实现`IMo`接口的类型都将受到影响：



```
IMo                        # 切面类型基础接口，所有切面类型都需要实现该接口
├── RawMoAttribute         # 继承Attribute，可自定义同步和异步切面方法
│   ├── MoAttribute        # 仅可自定义同步切面方法，异步切面方法使用调用同步切面方法的默认实现
│   └── AsyncMoAttribute   # 仅可自定义异步切面方法，同步切面方法使用调用异步切面方法的默认实现
└── RawMo                  # 与RawMoAttribute功能相同，唯一差别是RawMo未继承Attribute
    ├── Mo                 # 与MoAttribute功能相同，唯一差别是Mo未继承Attribute
    └── AsyncMo            # 与AsyncMoAttribute功能相同，唯一差别是AsyncMo未继承Attribute

```

升级前的属性与升级后的 Attribute 及接口的具体对应关系如下：




| 5\.0 之前切面类型属性 | 5\.0 对应的Attribute | 5\.0 对应的Interface |
| --- | --- | --- |
| `Flags` | `PointcutAttribute` | `IFlexibleModifierPointcut` |
| `Pattern` | `PointcutAttribute` | `IFlexiblePatternPointcut` |
| `Features` | `AdviceAttribute` | / |
| `MethodContextOmits` | `OptimizationAttribute` | / |
| `ForceSync` | `OptimizationAttribute` | / |
| `Order` | / | `IFlexibleOrderable` |


使用代码展示升级前后的差异：



```
// 5.0之前的切面类型定义
public class TestAttribute : MoAttribute
{
    public override AccessFlags Flags => AccessFlags.All | AccessFlags.Method;

    public override string Pattern => "method(* *(..))";

    public override Feature Features => Feature.OnEntry;

    public override ForceSync ForceSync => ForceSync.All;

    public override Omit MethodContextOmits => Omit.None;

    public override double Order => 2;
}

// 5.0的切面类型定义
[Pointcut("method(* *(..))")] // Pattern 属性和 Flags 属性合并为该属性，由于 Pattern 优先级高于 Flags，在 Pattern 有值的情况下忽略 Flags 配置
[Advice(Feature.OnEntry)]     // Features 属性迁移为该属性
[Optimization(ForceSync = ForceSync.All, MethodContext = Omit.None)] // ForceSync 和 MethodContextOmits 合并为该属性
public class T1Attribute : MoAttribute, IFlexibleOrderable           // 如果需要定义 Order，需要实现 IFlexibleOrderable 接口
{
    public double Order { get; set; } = 2;
}

```

看完上面的列表和代码后，你或许有个疑问“为什么升级后有的只有 Attribute，有的只有接口，而有的两个都有”。


这是结合用途综合考虑的。前面介绍到，删减属性成员是为了优化切面类型实例化后的内存占用。那么 Attribute 作为元数据，并不会在切面类型实例化时为每个实例单独创建，所以基本所有属性都可以使用 Attribute 代替。那么什么样的配置需要提供接口呢？在 5\.0 之前的版本可以通过`new`关键字为属性增加 `setter`，然后在应用切面类型时通过属性动态配置，如下代码所示：



```
// 5.0之前的用法
public class TestAttribute : MoAttribute
{
    // 默认Pattern只有getter，通过new关键字为Pattern增加setter
    public new Pattern { get; set; }
}

[Test(Pattern = "method(* *.Try*(..))")] // 应用Attribute动态指定Pattern
public class Cls { }

```

这种方式提供了一定的灵活性，在 5\.0 版本中，对于需要继续保持这种灵活性的配置提供了对应的接口。对于没有提供对应接口的配置，表示该配置不会在应用 Attribute 动态配置（如果你有这种使用场景，可以新建 [issue](https://github.com) 具体聊聊）。


#### Roslyn代码分析器


本次的属性成员变动较大，升级后手动修改会比较繁琐，同时还可能出现遗漏。虽然肉夹馍在编译时会对切面类型进行检查，并在发现不符合规范的切面类型时产生一个编译错误。但 5\.0 提供了更好的升级体验，新增 Roslyn 代码分析器和代码修复程序，可以在编写代码时直接发现问题并提供快捷修复。


![](https://raw.githubusercontent.com/inversionhourglass/Rougamo/82840776d1bc1fab5576c261899458b95cf22630/member_migration.gif)


### ref struct参数及返回值处理


在 5\.0 版本中，`MethodContextOmits`属性被迁移到`OptimizationAttribute`中。`MethodContextOmits`除了可以用来[瘦身`MethodContext`](https://github.com)，还可以用来处理`ref struct`文件，详见 [\#61](https://github.com). 虽然可以通过`OptimizationAttribute`设置`Omit`，但由于只提供了 Attribute 没有提供接口，所以无法在应用 Attribute 时动态配置。不过 5\.0 版本提供了更加方便的配置方式。


#### SkipRefStructAttribute


在 5\.0 版本中新增`SkipRefStructAttribute`用于处理`ref struct`：



```
public class TestAttribute : MoAttribute { }

[SkipRefStruct]
[Test]
public ReadOnlySpan<char> M(ReadOnlySpan<char> value) => default;

```

这种方式更加合理，如果方法上应用了多个切面类型，不再需要为每个切面类型指定`MethodContextOmits`，同时`SkipRefStructAttribute`还可以应用在类和程序集上，可以在更大范围上声明忽略`ref struct`。


#### 配置项skip\-ref\-struct


除了`SkipRefStructAttribute`的方式，在确定当前程序集默认忽略`ref struct`的情况下，还可以通过配置项`skip-ref-struct`为当前程序集应用这个设定，配合 [Cli4Fody](https://github.com):[MeoMiao 萌喵加速](https://biqumo.org) 可以实现非侵入式配置，`skip-ref-struct`设置为`true`的效果等同于`[assembly: SkipRefStruct]`。



```
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="FodyWeavers.xsd">
	<Rougamo skip-ref-struct="true" />
Weavers>

```

### 自定义切面类型生命周期


在 2\.2 版本中，肉夹馍新增了 [.NET无侵入式对象池解决方案](https://github.com)），原本计划使用 [Pooling](https://github.com) 完成对肉夹馍 GC 的整体优化，但思来想去，还是决定将对象池功能内置，同时新增单例模式。


在 5\.0 版本中，可以通过`LifetimeAttribute`指定切面类型的生命周期：



```
[Lifetime(Lifetime.Transient)] // 临时，每次创建都是直接new。在没有应用LifetimeAttribute时，默认为Transient
public class Test1Attribute : MoAttribute { }

[Lifetime(Lifetime.Pooled)] // 对象池，每次创建都从对象池中获取
public class Test2Attribute : MoAttribute { }

[Lifetime(Lifetime.Singleton)] // 单例
public class Test3Attribute : MoAttribute { }

```

需要注意的是，不是所有切面类型无脑设置为对象池或单例模式即可完成优化。选择生命周期时需要注意以下几点：


1. `Singleton`要求切面类型必须是无状态的，必须包含无参构造方法，且应用切面类型时不可调用有参构造方法或设置属性



```
[Lifetime(Lifetime.Singleton)]
public class SingletonAttribute : MoAttribute
{
    public SingletonAttribute() { }

    public SingletonAttribute(int value) { }

    public int X { get; set; }
}

[Singleton(1)]     // 编译时报错，不可调用有参构造方法
[Singleton(X = 2)] // 编译时报错，不可设置属性

```
2. `Pooled`要求切面类型必须包含无参构造方法，且应用切面类型时不可调用有参构造方法。如果切面类型有状态可实现`Rougamo.IResettable`接口，并在`TryReset`方法中完成状态重置，或在`OnExit` / `OnExitAsync`中完成状态重置



```
[Lifetime(Lifetime.Pooled)]
public class PooledAttribute : MoAttribute, IResettable
{
    public SingletonAttribute() { }

    public SingletonAttribute(int value) { }

    public int X { get; set; }

    public override void OnExit(MethodContext context)
    {
        // 可以在OnExit中状态重置，比如将X重置为0
        X = 0;
    }

    public bool TryReset()
    {
        // 也可以实现IResettable接口，在该方法中完成状态重置
        X = 0;

        // 返回true表示重置成功，返回false，当前对象将会直接抛弃，不会返回到对象池中
        return true;
    }
}

[Pooled(1)]     // 编译时报错，不可调用有参构造方法
[Pooled(X = 2)] // 支持的操作，所以如果需要在应用时设置一些状态，可以用属性的方式而不要用构造方法参数的方式

```
3. 结构体切面类型无法定义生命周期


总结来说，如果可以将切面类型设计为无状态的，推荐使用`Singleton`；如果无法保证无状态，但可以管理好状态的重置，推荐使用`Pooled`；如果无法很好的管理状态，可以使用结构体（但结构体无法继承 Attribute，所以无法在应用时像 Attribute 那样通过构造参数和属性动态配置）；最后，如果对 GC 优化要求没那么严格，使用默认的无拘无束的`Transient`即可。


### MethodContext对象池化


在 5\.0 版本中，`MethodContext`将默认从对象池中获取，这一默认行为将在较大程度上优化Rougamo产生的GC。


`MethodContext`的对象池和切面类型的对象池用的是同一个，可以通过环境变量设置对象池最大持有数量，默认为`CPU逻辑核心数 * 2`（不同类型分开）。




| 环境变量 | 说明 |
| --- | --- |
| `NET_ROUGAMO_POOL_MAX_RETAIN` | 对象池最大持有对象数量，对所有类型生效 |
| `NET_ROUGAMO_POOL_MAX_RETAIN_` | 指定类型对象池最大持有对象数量，覆盖`NET_ROUGAMO_POOL_MAX_RETAIN`配置，为指定类型的全名称，命名空间分隔符`.`替换为`_` |


## 异常堆栈信息优化


关联 \[[\#82](https://github.com)]


Rougamo 自 4\.0 版本开始全面使用代理织入的方式，由于该方式会为被拦截方法额外生成一个代理方法，所以在堆栈信息中会额外产生一层调用堆栈，在程序抛出异常时，调用堆栈会显得复杂且冗余：



```
// 测试代码
public class TestAttribute : MoAttribute { }

try
{
    await M1();
}
catch (Exception e)
{
    Console.WriteLine(e);
}

[Test]
public static async Task M1() => await M2();

[Test]
public static async ValueTask M2()
{
    await Task.Yield();
    M3();
}

[Test]
public static void M3() => throw new NotImplementedException();

```

在 5\.0 之前，上面代码在 .NET 6\.0 中运行的结果为（不同.NET版本堆栈信息可能有些差异，早期的Framework版本将更加冗长）：



```
System.NotImplementedException: The method or operation is not implemented.
   at X.Program.$Rougamo_M3() in D:\X\Y\Z\Program.cs:line 49
   at X.Program.M3()
   at X.Program.$Rougamo_M2() in D:\X\Y\Z\Program.cs:line 43
   at X.Program.M2()
   at X.Program.M2()
   at X.Program.$Rougamo_M1() in D:\X\Y\Z\Program.cs:line 36
   at X.Program.M1()
   at X.Program.M1()
   at X.Program.Main(String[] args) in D:\X\Y\Z\Program.cs:line 13

```

5\.0 版本之后，运行结果为：



```
System.NotImplementedException: The method or operation is not implemented.
   at X.Program.$Rougamo_M3() in D:\X\Y\Z\Program.cs:line 49
   at X.Program.$Rougamo_M2() in D:\X\Y\Z\Program.cs:line 43
   at X.Program.$Rougamo_M1() in D:\X\Y\Z\Program.cs:line 36
   at X.Program.Main(String[] args) in D:\X\Y\Z\Program.cs:line 13

```

这种异常堆栈优化在 .NET 6\.0 及之后的 .NET 版本中是默认的，不需要任何操作，但对于 .NET 6\.0 之前的版本，需要调用`Exception`的扩展方法`ToNonRougamoString`来获取优化后的ToString字符串，或者调用`Exception`的扩展方法`GetNonRougamoStackTrace`获取优化后的调用堆栈。


之所以 .NET 6\.0 之后默认支持异常堆栈优化，是因为 .NET 6\.0 之后调用堆栈会默认排除应用了`StackTraceHiddenAttribute`的方法。


**关于优化后堆栈信息默认方法名自带`$Rougamo_`前缀的说明**


代理织入使得实际方法全部增加`$Rougamo_`前缀，所以只有`$Rougamo_`前缀的方法菜与 PDB 信息对应，可以获取行号等信息。不做额外处理去除前缀是因为没有必要，前缀固定不会影响阅读分析，额外操作去除前缀影响性能，同时也会导致 .NET 6\.0 及以上版本无法无感知完成优化。如果确实想要去除该前缀，请自行处理。


另外，如果你有特殊需求不需要这种堆栈信息优化，可以将配置项`pure-stacktrace`设置为`false`。



```
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="FodyWeavers.xsd">
	<Rougamo pure-stacktrace="false" />
Weavers>

```

## AspectN新语法


新增`ctor`和`cctor`，用于用于快速匹配构造方法和静态构造方法。


* `ctor(([..]))`



```
// 匹配所有构造方法
[Pointcut("ctor(*(..))")]

// 匹配所有非泛型类型的构造方法
[Pointcut("ctor(*(..))")]

// 匹配IService子类的构造方法
[Pointcut("ctor(IService+(..))")]

// 匹配所有无参构造方法
[Pointcut("ctor(*())")]

// 匹配所有包含三个参数（任意类型）的构造方法
[Pointcut("ctor(*(,,))")]

// 匹配两个参数分别为int和Guid的构造方法
[Pointcut("ctor(*(int,System.Guid))")]

```
* `cctor()`



```
// 匹配所有静态构造方法
[Pointcut("cctor(*)")]

// 匹配类名包含Singleton的静态构造方法
[Pointcut("cctor(*Singleton*)")]

```


## 配置化非侵入式织入


5\.0 版本可以通过配置`FodyWeavers.xml`完成切面类型应用，而不必再添加/修改C\#代码。



```
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="FodyWeavers.xsd">
  <Rougamo>
    <Mos>
      <Mo assembly="Rougamo.OpenTelemetry" type="Rougamo.OpenTelemetry.OtelAttribute" pattern="method(* *Service.*(..))"/>
    Mos>
  Rougamo>
Weavers>

```

上面的配置中，每一个`Mo`节点为一条应用规则，`Mos`节点下可以定义多个`Mo`节点，下面是`Mo`节点的属性说明：


* `type`，切面类型全名称
* `assembly`，切面类型所在程序集名称（不要.dll后缀）
* `pattern`，AspectN 表达式，切面类型`type`将应用到该表达式匹配的方法上。该配置可选，未设置时将采用切面类型`type`自身的匹配规则


配置结合 [.NET无侵入式对象池解决方案——零侵入式池化操作](https://github.com)。


## 其他更新


* **删除配置项`moarray-threshold`**


该配置项是用数组优化大量切面类型应用到方法时，用遍历数组执行切面方法的方式代替每个切面类型单独执行切面方法，以达到精简MSIL的目的。


但随着Rougamo的功能完善，在 4\.0 版本中因异步切面的加入，使得异步方法无法使用数组达到预期优化。在 5\.0 版本中，随着对象池的加入，同步方法也难以使用数组完成预期优化。


综合复杂度和实际优化效果考虑，最终决定在 5\.0 版本中移除配置项`moarray-threshold`。
* **新增`ISyncMo`和`IAsyncMo`接口**


由于结构体无法继承父类/父结构体，所以在定义结构体切面类型时只能直接实现`IMo`接口，但该接口包含全部同步/异步切面方法，全部实现比较繁琐。


肉夹馍在 5\.0 版本中新增`ISyncMo`和`IAsyncMo`，通过 [默认接口方法](https://github.com) 对部分方法提供默认实现。


默认接口方法需要 SDK 最低 .NET Core 3\.0 的版本，所以只有 .NET Core 3\.0 及以上版本才有`ISyncMo`和`IAsyncMo`两个接口。



```
// 实现ISyncMo接口可以不用实现异步切面方法
[Pointcut("method(* *(..))")]
public struct SyncMo : ISyncMo
{
    public void OnEntry(MethodContext context) { }

    public void OnException(MethodContext context) { }

    public void OnExit(MethodContext context) { }

    public void OnSuccess(MethodContext context) { }
}

// 实现IAsyncMo接口可以不用实现同步切面方法
[Pointcut("method(* *(..))")]
public struct AsyncMo : IAsyncMo
{
    public ValueTask OnEntryAsync(MethodContext context) => default;

    public ValueTask OnExceptionAsync(MethodContext context) => default;

    public ValueTask OnExitAsync(MethodContext context) => default;

    public ValueTask OnSuccessAsync(MethodContext context) => default;
}

// 应用切面类型
[assembly: Rougamo]
[assembly: Rougamo(typeof(AsyncMo))]

```


# Unity相关


如果要拿 Rougamo 与 PostSharp 进行对比并讨论其优势，那么第一个鲜为人知的优势就是肉夹馍开源免费，而另一个比较大的优势可能就是 Unity 了。


根据我查询的资料显示，PostSharp 以及新推出的 Metalama 并不支持 Unity.


* [https://stackoverflow.com/a/20647529/3614672](https://github.com)
* [https://support.postsharp.net/request/20888\-support\-for\-unity3d](https://github.com)


此前曾有朋友到 GitHub 中询问如何在 Unity 中使用肉夹馍，但很可惜，我并不了解 Unity，所以当时并给有给出解决方案。后来直到 [@gaozhou](https://github.com) 带着他的解决方案出现了。现在，我可以掷地有声的回答——是的，肉夹馍支持 Unity.


具体解决方案，请查看 [https://github.com/inversionhourglass/Rougamo/issues/86\#issuecomment\-2378505655](https://github.com) 。由于本人对 Unity 一窍不通，所以无法提供一个开箱即用的 Package，有兴趣的朋友可以参考解决方案制作一个开箱即用的 Package 分享出来。


# 鸣谢


每次大版本发布的时候，就是 Rougamo 发博客刷存在的时候，但随着 5\.0 的发布，大版本的发布将陷入停滞（小版本还会发，但一般不会特意发博客通告），肉夹馍的宣发也将随之减少。感谢各位朋友长期以来的关注，感谢提供使用反馈的朋友们，感谢选择使用肉夹馍的各开源项目。


