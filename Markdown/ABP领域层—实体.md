3.1 ABP 领域层—实体
--------------------------
> **实体是 DDD（领域驱动设计）的核心概念之一。Eric Evans 是这样描述的“很多对象不是通过它们的属性定义的，而是通过一连串的连续性事件和标识定义的”（引用领域驱动设计一书）。**

> 对象不是通过它们的属性来下根本性的定义，而应该是通过它的线性连续性和标识性定义的。所以，实体是具有唯一标识的ID且存储在数据库中。实体通常被映射成数据库中的一个表。 
 
### 3.1.1 实体类 
在 ABP 中，实体继承自 Entity 类，请看下面示例： 

``` csharp
public class Person : Entity 
{     
    public virtual string Name { get; set; } 
    public virtual DateTime CreationTime { get; set; } 
	public Task() 
    { 
        CreationTime = DateTime.Now; 
    } 
}
```
Person 类被定义为一个实体。它具有两个属性，它的父类( Entity )中有 Id 属性, Id 是该实体的主键。所以，Id 是所有继承自 Entity 类的实体的主键（所有实体的主键都是 Id 字段）。 

Id(主键)数据类型可以被更改。默认是 int类型。如果你想给 Id 定义其它类型，你应该像下面示例一样来声明 Id 的类型。 
``` csharp
public class Person : Entity<long> 
{     
	public virtual string Name { get; set; } 
    public virtual DateTime CreationTime { get; set; } 
    public Task() 
    { 
        CreationTime = DateTime.Now; 
    } 
} 
```
你可以设置为 string，Guid 或者其它数据类型。 
实体类重写了 equality  (==) 操作符用来判断两个实体对象是否相等（两个实体的 Id 是否相等）。
还定义了一个 IsTransient()方法来检测当前 Id 的值是否与指定的类型的缺省值相等。 
 
###3.1.2 接口约定 
在很多应用程序中，很多实体具有像 CreationTime 的属性（数据库表也有该字段）用来指示该实体是什么时候被创建的。APB 提供了一些有用的接口来实现这些类似的功能。也就是说，为实现这些接口的实体提供了一个通用的编码方式（通俗的说只要实现指定的接口就能实现指定的功能）。 
#### 1.  审计（Auditing） 
实体类实现 IHasCreationTime  接口就可以具有 CreationTime 的属性。当该实体被插入到数据库时， ABP 会自动设置该属性的值为当前时间。 
``` csharp
public interface IHasCreationTime 
{ 
    DateTime CreationTime { get; set; } 
} 
```
Person 类可以被重写像下面示例一样实现 IHasCreationTime 接口：

``` csharp 
public class Person : Entity<long>, IHasCreationTime 
{     
	public virtual string Name { get; set; } 
    public virtual DateTime CreationTime { get; set; } 
    public Task() 
    { 
        CreationTime = DateTime.Now; 
    } 
} 
```
ICreationAudited 扩展自 IHasCreationTime 并且该接口具有属性 CreatorUserId ： 
``` csharp
public interface ICreationAudited : IHasCreationTime 
{     
	long? CreatorUserId { get; set; } 
} 
```
当保存一个新的实体时，ABP 会自动设置 CreatorUserId 的属性值为当前用户的 Id 。
你可以轻松的实现 ICreationAudited 接口，通过派生自实体类 CreationAuditedEntity  (因为该类已经实现了 ICreationAudited 接口，我们可以直接继承 CreationAuditedEntity 类就实现了上述功能)。它有一个实现不同 ID 数据类型的泛型版本(默认是 int)，可以为 ID（Entity 类中的 ID）赋予不同的数据类型。 

下面是一个为实现类似修改功能的接口 
``` csharp
public interface IModificationAudited 
{ 
    DateTime? LastModificationTime { get; set; }     
    long? LastModifierUserId { get; set; } 
} 
```
当更新一个实体时，APB 会自动设置这些属性的值。你只需要在你的实体类里面实现这些属性。 
如果你想实现所有的审计属性，你可以直接扩展 IAudited 接口；示例如下： 
``` csharp
public interface IAudited : ICreationAudited, IModificationAudited 
{ 
         
} 
```
作为一个快速开发方式，你可以直接派生自 AuditedEntity 类，不需要再去实现 IAudited 接口（AuditedEntity 类已经实现了该功能，直接继承该类就可以实现上述功能），AuditedEntity 类有一个实现不同 ID 数据类型的泛型版本(默认是 int)，可以为 ID（Entity 类中的 ID）赋予不同的数据类型。 
#### 2. 软删除(Soft delete) 
软删除是一个通用的模式，它标记一个实体已经被删除了，而不是实际从数据库中删除记录。
例如：你可能不想从数据库中硬删除一条用户记录，因为它被许多其它的表所关联。
为了实现软删除的目的我们可以实现该接口 ISoftDelete： 
``` csharp
public interface ISoftDelete
{     
	bool IsDeleted { get; set; } 
} 
```
ABP 实现了开箱即用的软删除模式。当一个实现了软删除的实体正在被删除， ABP 会察觉到这个动作，并且阻止其删除，设置 IsDeleted 属性值为 true 并且更新数据库中的实体。也就是说，被软删除的记录不可以从数据库中检索出，ABP 会为我们自动过滤软删除的记录。（例如：Select 查询，这里指通过 ABP 查询，不是通过数据库中的查询分析器查询。） 

如果你用了软删除，你有可能也想实现这个功能，就是记录谁删除了这个实体。要实现该功能你可以实现 IDeletionAudited 接口，请看下面示例： 
``` csharp
public interface IDeletionAudited : ISoftDelete 
{     
	long? DeleterUserId { get; set; } 
    DateTime? DeletionTime { get; set; } 
} 
```
正如你所看到的 IDeletionAudited 扩展自 ISoftDelete 接口。当一个实体被删除的时候 ABP 会自动的为这些属性设置值。
如果你想为实体类扩展所有的审计接口（例如：创建（creation），修改（modification）和删除（deletion）），你可以直接实现 IFullAudited 接口，因为该接口已经继承了这些接口。
请看下面示例： 
``` csharp
public interface IFullAudited : IAudited, IDeletionAudited 
{ 
         
} 
```
作为一个快速开发方式，你可以直接从 FullAuditedEntity 类派生你的实体类，因为该类已经实现了 IFullAudited 接口。 
 
为了导航定义属性到你的 User 实体，所有的审计接口和类都有一个泛型模板（例如： ICreationAudited<TUser>和FullAuditedEntity<TPrimaryKey, TUser>），这里的TUser指的进行创建，修改和删除的用户的实体类的类型，
详细请看源代码（Abp.Domain.Entities.Auditing 空间下的FullAuditedEntity<TPrimaryKey, TUser>类），TprimaryKey 指的是Entity基类Id 类型，默认是int。 
 
#### 3. 激活状态/闲置状态(Active/Passive) 
有些实体需要被标记为激活状态或者闲置状态。那么你可以为实体采取 active/passive 状态的方式来实现。
基于这个原因而创建的实体，你可以扩展IPassivable 接口来实现该功能。该接口定义了 IsActive 的属性。 

如果你首次创建的实体被标记为激活状态，你可以在构造函数设置 IsActive 属性值为 true。这不同于软删除（IsDeleted）。
如果实体被软删除，它不能从数据库中被检索到（ABP 已经过滤了软删除记录）。但是对于激活状态/闲置状态的实体，这完全取决于你怎样去获取这些被标记了的实体。 

###3.1.3 IEntity 接口 
事实上 Entity 实现了 IEntity 接口（ Entity<TPrimaryKey> 实现了 IEntity<TPrimaryKey>接口）。如果你不想从 Entity 类派生，你能直接的实现这些接口。
其他实体类也可以实现相应的接口。但是不建议你用这种方式。除非你有一个很好的理由不从 Entity 类派生。
