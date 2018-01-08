## 简介

ABP 提供了一个基础设施，可以自动的记录所有实体以及属性的变更历史。

实体更改涉及要保存的字段有：**租户Id (tenant id)**,
**实体变更集Id (entity change set id)**, **实体Id (entity id)**,
**实体类型名称 (entity type name)**, **变更时间 (change time)** 以及 **变更类型 (change type)**.

实体属性更改涉及要保存的字段有： **租户Id (tenant id)**,
**实体变更集Id (entity change set id)**, **属性名称 (property name)**, **属性类型名称 (property type name)**,
**新值 (new value)** 以及 **旧值 (original value)**.

更改的实体被分组在一个每次被调用的SaveChanges变更集合里面。

实体变更集合涉及要保存的字段有：**租户Id (tenant id)**,
变更者的 **用户Id (user id)**, **创建时间 (creation time)**, **原因 (reason)**, 客户端
**IP地址**, 客户端 **计算机名称 (computer name)** 以及 **浏览器信息 (browser info)** (如果实体的变更是来自web端请求).

实体历史跟踪系统使用 [**IAbpSession**](2.2ABP公共结构-会话管理.md) 取得当前 UserId 以及 TenantId。

默认是没有实体会被自动跟踪。应该通过启动配置或使用属性来配置实体。


#### 关于 IEntityHistoryStore

实体历史跟踪系统使用  **IEntityHistoryStore** 保存变更信息。也可以使用你自己的方式来扩展它， **module-zero** 项目默认已经实现了该接口。

### Configuration

实体变更跟踪系统配置，你可以在 [模块](1.3ABP总体介绍-模块系统.md) 的PreInitialize 方法中配置 **Configuration.EntityHistory** 。记录实体历史默认 **开启**。

你可以禁用该功能，如下所示：

```csharp
public class MyModule : AbpModule
{
    public override void PreInitialize()
    {
        Configuration.EntityHistory.IsEnabled = false;
    }

    //...
}
```

下面是配置实体历史的一些属性：

- **IsEnabled ：** 用来 开启/关闭 实体历史跟踪；默认是：**true** 开启。
-  **IsEnabledForAnonymousUsers ：** 如果该属性设置为 **true**，对于没有登录系统的用户所操作的实体变更历史也会保存起来。
-  **Selectors ：** 用来选择那些实体的变更需要保存。
  
**Selectors** 是一个用来添加选择那些实体需要保存历史变更的过滤谓词列表。选择器具有唯一 **name** 和 **predicate** 。例如：选择器可以用来选择 **所有的审计实体(Full Audited Entities)**。

如下所示：

```csharp
Configuration.EntityHistory.Selectors.Add(
    new NamedTypeSelector(
        "Abp.FullAuditedEntities",
        type => typeof (IFullAudited).IsAssignableFrom(type)
    )
);
```

你可以在模块的 **PreInitialize** 方法中添加选择器。


### 通过特性 开启/关闭 Enable/Disable 

你可以通过配置来选择需要跟踪的实体，也可以通过对单一实体或单一实体的某个属性添加 **Audited** 和 **DisableAuditing** 特性。例如：

```csharp
[Audited]
public class MyEntity : Entity
{
    public string MyProperty1 { get; set; }

    [DisableAuditing]
    public int MyProperty2 { get; set; }

    public long MyProperty3 { get; set; }
}
```

除了 MyProperty2属性没有被跟踪外， MyEntity 的所有属性都会被跟踪。你可以对实体的某个属性按需使用 **Audited** 特性，使用了该特性的字段会被跟踪记录更改历史。

**DisableAuditing** 特性可以用来对实体或者实体的某个属性禁用记录历史变更。**因此，你可以用来隐藏敏感数据** 被记录跟踪；例如：密码。


### Reason

实体变更集合有个 **Reason** 属性，可以用来记录是什么导致了这一系列的变更。例如：导致这些变更的用例。

例如：Person A 将钱从A账户转到B账户。两个账户余额都会改变，“汇款” 将被记录为此次变更的原因。由于余额变化可能是其他原因，**Reason** 属性解释了为什么会进行这些更改。

**Abp.AspNetCore** 包，实现了 **HttpRequestEntityChangeSetReasonProvider**,
它会返回 HttpContext.Request 的URL作为产生变更的原因。

#### UseCase 特性

首先方法是使用 **UseCase** 特性。例如：

```csharp
[UseCase(Description = "分配问题给用户")]
public virtual async Task AssignIssueAsync(AssignIssueInput input)
{
    // ...
}
```

##### UseCase 特性限制

你可以使用UseCase特性：

- 所有实现其接口的类的 **public** 或者 **public virtual** 方法，例如：使用了它自己接口的Applicaton Service。
- 所有自注册类的 **public virtual** 方法，例如 **MVC Controllers**。
- 所有的 **protected virtual** 方法。

#### IEntityChangeSetReasonProvider

在某些情况下，你可能需要在某个特定范围内更改/复写 **Reason** 的值。你可以使用 **IEntityChangeSetReasonProvider.Use(...)** 方法，如下所示：

```csharp
public class MyService
{
    private readonly IEntityChangeSetReasonProvider _reasonProvider;

    public MyService(IEntityChangeSetReasonProvider reasonProvider)
    {
        _reasonProvider = reasonProvider;
    }

    public virtual async Task AssignIssueAsync(AssignIssueInput input)
    {
        var reason = "Assign an issue to user: " + input.UserId.ToString();
        using (_reasonProvider.Use(reason))
        {
            ...

            _unitOfWorkManager.Current.SaveChanges();
        }
    }
}
```

Use方法会返回一个 **IDisposable** 并且 **必须被释放**。一旦该值被释放，Reason会自动的还原为以前所使用的值。


### 注意

- 属性必须是 **public** 才能保存在变更日志中。**private，protected** 属性会被忽略。
- **DisableAuditing** 优先于 **Audited** 特性。
- 只适用于实体。
- 只适用于原生属性，例如string，int，bool ...


