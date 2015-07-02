# ABP表现层
## ABP展现层—动态WebApi层
### 建立动态WebApi控制器

Abp框架能够通过应用层自动生成web api：
``` csharp
    public interface ITaskAppService : IApplicationService
    {
        GetTasksOutput GetTasks(GetTasksInput input);
        void UpdateTask(UpdateTaskInput input);
        void CreateTask(CreateTaskInput input);
    }
```

Abp框架通过一行关键代码的配置就可以自动、动态的为应用层建立一个web api 控制器:

    DynamicApiControllerBuilder.For<ITaskAppService>("tasksystem/task").Build();
这样就OK了！建好的webapi控制器(/api/services/tasksystem/task)所有的方法都能够在客户端调用。webapi控制器通常是在模块初始化的时候完成配置。
ITaskAppService是应用层服务（application service)接口，我们通过封装让接口实现一个api控制器。ITaskAppService不仅限于在应用层服务使用，这仅仅是我们习惯和推荐的使用方法。
tasksystem/task是api 控制器的命名空间。一般来说，应当最少定义一层的命名空间，如：公司名称/应用程序/命名空间/命名空间1/服务名称。
‘api/services/’是所有动态web api的前缀。所以api控制器的地址一般是这样滴：‘/api/services/tasksystem/task’，GetTasks 方法的地址一般是这样滴：
‘/api/services/tasksystem/task/getTasks’。因为在传统的js中都是使用驼峰式命名方法，这里也不一样。
你也可以删除一个api方法，如下：

    DynamicApiControllerBuilder
    
    .For<ITaskAppService>("tasksystem/taskService")
    
    .ForMethod("CreateTask").DontCreateAction()
    
    .Build();
ForAll方法
在程序的应用服务层建立多个api控制器可能让人觉得比较枯燥，DynamicApiControllerBuilper提供了建立所有应用层服务的方法，如下所示：

    DynamicApiControllerBuilder
    
    .ForAll<IApplicationService>(Assembly.GetAssembly(typeof(SimpleTaskSystemApplicationModule)), "tasksystem")
    
    .Build();
    
ForAll方法是一个泛型接口，第一个参数是从给定接口中派生的集合，最后一个参数则是services命名空间的前缀。ForAll集合有ITaskAppService和 IpersonAppService接口。根据如上配置，服务层的路由是这样的：'/api/services/tasksystem/task'和'/api/services/tasksystem/person'。

服务命名约定：服务名+AppService(在本例中是person+AppService) 的后缀会自动删除，生成的webapi控制器名为“person”。同时，服务名称将采用峰驼命名法。如果你不喜欢这种约定，你也可以通过“WithServiceName”方法来自定义名称。如果你不想创建所有的应用服务层，可以使用where来过滤部分服务。

### 使用动态JavaScript代理

你可以通过ajax来动态创建web api控制器。Abp框架对通过动态js代理建立web api 控制器做了些简化，你可以通过js来动态调用web api控制器

    abp.services.tasksystem.task.getTasks({
    
    state: 1
    
    }).done(function (data) {
    
    //use data.tasks here..
    .
    });

js代理是动态创建的，页面中需要添加引用:

    <script src="/api/abp.ServiceProxies/GetAll" type="text/javascript"></script>
    
服务方法(service methods)返回约定（可参见JQ的Deferred)，服务方法使用Abp框架.ajax代替，可以处理、显示错误。
#### Ajax参数
自定义ajax代理方法的参数：

    Abp.services.tasksystem.task.createTask({
    
        assignedPersonId: 3,
        
        description: 'a new task description...'
        
    },{ //override jQuery's ajax parameters
    
        async: false,
        
        timeout: 30000
        
    }).done(function () {
    
        Abp.notify.success('successfully created a task!');
        
    });
    
所有的jq.ajax参数都是有效的。
#### 单一服务脚本
'/api/abpServiceProxies/GetAll'将在一个文件中生成所有的代理，通过 '/api/abpServiceProxies/Get?name=serviceName' 你也可以生成单一服务代理，在页面中添加：

    <script src="/api/abpServiceProxies/Get?name=tasksystem/task" type="text/javascript"></script>
#### Augular框架支持
Abp框架能够公开动态的api控制器作为angularjs服务，如下所示：

    (function() {
    
        angular.module('app').controller('TaskListController', [
        
            '$scope', 'abp.services.tasksystem.task',
            
            function($scope, taskService) {
            
                var vm = this;
                
                vm.tasks = [];
                
                taskService.getTasks({
                
                    state: 0
                    
                }).success(function(data) {
                
                    vm.tasks = data.tasks;
                    
                });
                
            }
            
        ]);
        
    })();
我们可以将名称注入服务，然后调用此服务，跟调用一般的js函数一样。注意：我们成功注册处理程序后，他就像一个augular的$http服务。ABP框架使用angular框架的$http服务，如果你想通过$http来配置，你可以设置一个配置对象作为服务方法的一个参数。

要使用自动生成的服务，需要添加:

    <script src="~/abp Framework/Framework/scripts/libs/angularjs/Abp Framework.ng.js"></script>
    
    <script src="~/api/abp Framework/ServiceProxies/GetAll?type=angular"></script>
#### Durandal支持
ABP框架可以注入服务到Durandal框架，如下：

    define(['service!tasksystem/task'],
    
    function (taskService) {
    
        //taskService can be used here
        
    });
ABP框架配置Durandal（实际上是Require.js）来解析服务代理并注入合适的js到服务代理。





