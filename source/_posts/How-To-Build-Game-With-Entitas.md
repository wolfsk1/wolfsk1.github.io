---
title: 如何使用Entitas来构建游戏
date: 2018-06-16 17:18:32
tags:
- 游戏架构
- ECS
- Entitas
- 框架
description: 这篇文章主要用来解释如何使用Entitas来组织你的项目结构。本文假定你已经对Entitas的工作方式有足够的了解。因此你这不是一篇教你Entitas各种特性的文章。这篇文章的主要目的是使用Entitas框架使你的项目代码更加的简洁明了。
categories: 
- 技术
---
## 序言
本文意译自[《How I build games with Entitas (FNGGames)》](https://github.com/sschmid/Entitas-CSharp/wiki/How-I-build-games-with-Entitas-%28FNGGames%29)这篇文章，是作者在阅读时对其文章内容的一个思想记录。以下正文。

这篇文章主要用来解释如何使用Entitas来组织你的项目结构。本文假定你已经对Entitas的工作方式有足够的了解。因此你可以明白这不是一篇教你Entitas各种特性的文章。这篇文章的主要目的在于如何通过Entitas使你的项目代码可以更加的简洁明了。

这篇文章将围绕下面列出的话题来展开。针对Views和view controllers的概念，我会使用Unity来举例说明，但这种思路可以应用到你使用的任何引擎中。
- 定义 Data，Logic，View层
- 解耦这些层
- 使用接口来抽象化设计
- 控制反转
- 抽象化视图层（Views 和 view controllers）
## 定义
- **Data：**游戏中的一些状态，如血量背包经验敌人类型ai状态移动方向和速度等等。在Entitas中，这些数据存在于Components中。
- **Login：**数据如何进行运算的一些规则。比如放入背包，建造物品，武器开火等，在Entitas中，Logic将被实现为Systems。
- **View：**用来向玩家展示游戏状态的代码，如渲染，动画，音频，ui组件等。在我的案例中，他们实际上就是存在于GameObject上的MonoBehaviours。
- **Services：**外部的信息模块，信息源。如，寻路，排行榜，反作弊，社交，物理系统，甚至包含游戏引擎本身。
- **Input：**外部的输入系统。通常可以有限的访问游戏逻辑的一部分。如控制器，键盘鼠标网络输入。
## 架构
任何游戏的核心都是通过CPU在做数值模拟，每一个游戏本质上都是一组组的数据（游戏状态）产生一些有规则的周期性的变化（游戏的逻辑）。游戏状态包括得分，血量，经验，NPC对话，库存等东西。游戏逻辑规定了这些数据通过哪种规则进行转化（如调用PutItemInBag方法）来改变游戏的状态。理论上整个这个过程或者说模拟都可以在不存在任何附加层的状态下进行。

通常来说，游戏与纯模拟的区别在于，游戏有一个外部用户可以使用模拟逻辑来从外界改变游戏的状态。随着用户的加入，游戏也应该将一些游戏状态用适当的方式转达给用户。这就是视图层（View Layer）。因此有一部分代码会用来展现游戏的状态，如渲染，播放音频，更新UI组件等。没有这层，用户便无法理解与游戏中模拟系统的交互的过程。

大多数游戏架构的目的是为了使视图，逻辑，数据层更方便的分离。数据模拟层不应该关心甚至不需要去知道他们如何对应屏幕的图像。而当玩家受伤时，进行伤害计算的代码也不应该在屏幕渲染血液或者播放疼痛声音效果的代码中出现。

我在我的项目中使用如下图所示的架构来保持上文所述的这种分离。我将试着描述图中的每一个部分的含义。通过接口的方式有助于这些东西的抽象化。
![Alt text](./68747470733a2f2f692e696d6775722e636f6d2f5244576b6150582e706e67.png)


## 抽象化的作用
抽象化是解决**你想做什么**和**你如何去做这件事儿**之间强耦合的一种方式。举例来说，假设我们有一个函数是用来输出调试信息给用户的。
### 笨办法
Debug.Log就是一个广为人知的笨办法。你需要把这个方法放在各个需要输出调试信息的地方。这样在你的所有代码中都立刻产生了非常强的耦合，成为了代码维护的噩梦。
### 笨办法有哪些问题
试想一下，如果你想将调试信息记录在某个日志文件中而不是直接输出在控制台时，你打算在游戏内实现一个调试控制台时，你必须在你所有涉及到Debug.Log代码的地方添加新的代码或者替换掉旧的代码调用。甚至对于中型项目来说这都是一个噩梦。

你也将拥有一些令人困惑的，可读性较差的代码，因为这些代码没有将责任分离。如果你完成了一个Unity示例项目，你会觉得这些还行，没那么不堪。那是因为Unity帮你做了一些事儿，实际上你的CharacterController在处理移动时，根本不应该存在对原始的用户输入的处理，管理背包，播放脚步声或者发facebook的这些代码。你应该只让你的角色移动，从而分隔焦点。
### 解耦的代码 - Entitas的实现
那让我们看看如果用Entitas来做的话，会比笨方法好多少。

在Entitas中，我们会先创建一个LogMessageComponent组件，并设计一个用来收集信息同时进行实际操作的ReactiveSystem系统。我们可以很轻松的创建一个Entity，然后挂上对应的组件并且将消息输出在控制台上。
``` Csharp
using UnityEngine;
using Entitias; 
using System.Collections.Generic;

// debug message component
[Debug] 
public sealed class DebugLogComponent : IComponent {
    public string message;
}

// reactive system to handle messages
public class HandleDebugLogMessageSystem : ReactiveSystem<DebugEntity> {

    // collector: Debug.Matcher.DebugLog

    // filter: entity.hasDebugLog

    public void Execute (List<DebugEntity> entities) {
        foreach (var e in entities) {
            Debug.Log(e.debugLog.message);
            e.isDestroyed = true;
        }
    }
}
```
现在无论什么东西想去创建一条日志信息，他只需要创建一个Entity，然后这个系统就会关注这些事儿。我们可以随意的变更我们的实现，只需要修改这个系统中的某处代码。
### Entitas官方这种实现的一些问题
一般来说这种方法已经足够了，特别是对一些比较小的项目来说。但这种方式也不是完全没有问题的，我们直接使用了Unity的API，将我们的系统的实现与Unity产生了强依赖。

如果你的调试信息需要记录在一个json文件中或者通过网络发送你的调试信息，你必须做一些补丁式的特殊实现。同时这个系统也将同时饮用很多不必要的库文件（如UnityEngine，json库，网络库等等）。同时代码的可读性会变差，当你的工具库发生修改的时候，代码出错的可能性也会提升。更别说如果你要更换你的引擎的话，你就得完完全全重写你的游戏代码。

## 接口示例
在C#中可以使用接口来解决这些依赖问题。

当我在填写接口的时候，我只需要关注我实际上想要干什么，然后写下一些带有自述性的简单API就可以。那LogMessage系统来举例：
``` Csharp
// the interface 
public interface ILogService {
    void LogMessage(string message);
}

// a class that implements the interface
using UnityEngine;
public class UnityDebugLogService : ILogService {
    public void LogMessage(string message) {
        Debug.Log(message);
    }
}

// another class that does things differently but still implements the interface
using SomeJsonLib;
public class JsonLogService : ILogService {
    string filepath;
    string filename;
    bool prettyPrint;
    // etc...
    public void LogMessage(string message) {
        // open file
        // parse contents
        // write new contents
        // close file
    }
}
```

然后我们在系统中只需要直接使用ILogService接口就可以了，如果我们需要一个JsonLogService，我们只需要实现一个并将其填充到接口中去。System甚至可以在你实现之前就使用这个接口。
``` Csharp
// the previous reactive system becomes 
public class HandleDebugLogMessageSystem : ReactiveSystem<DebugEntity> {

    ILogService _logService;
    
    // contructor needs a new argument to get a reference to the log service
    public HandleDebugLogMessageSystem(Contexts contexts, ILogService logService) {
         // could be a UnityDebugLogService or a JsonLogService
        _logService = logService; 
    }
    
    // collector: Debug.Matcher.DebugLog
    // filter: entity.hasDebugLog

    public void Execute (List<DebugEntity> entities) {
        foreach (var e in entities) {
            _logService.LogMessage(e.DebugLog.message); // using the interface to call the method
            e.isDestroyed = true;
        }
    }
}
```

IInputService是我项目中一个相对更复杂一些的示例。我需要去思考哪些输入输出才是我真正需要的，我如何定义一个关于输入输出的集合。
``` Csharp
// the previous reactive system becomes 
public class HandleDebugLogMessageSystem : ReactiveSystem<DebugEntity> {

    ILogService _logService;
    
    // contructor needs a new argument to get a reference to the log service
    public HandleDebugLogMessageSystem(Contexts contexts, ILogService logService) {
         // could be a UnityDebugLogService or a JsonLogService
        _logService = logService; 
    }
    
    // collector: Debug.Matcher.DebugLog
    // filter: entity.hasDebugLog

    public void Execute (List<DebugEntity> entities) {
        foreach (var e in entities) {
            _logService.LogMessage(e.DebugLog.message); // using the interface to call the method
            e.isDestroyed = true;
        }
    }
}
```

现在我就可以具体去实现EmitInputSystem来将实际的操作数据放入Entitas中，然后这些数据就可以直接被拿去操作了。这样我便可以直接切换我的实现使其脱离Unity，而System也只关心从接口中可以查询到什么。

``` Csharp
public class EmitInputSystem : IInitalizeSystem, IExecuteSystem {    
    Contexts _contexts;
    IInputService _inputService; 
    InputEntity _inputEntity;
    
    // contructor needs a new argument to get a reference to the log service
    public EmitInputSystem (Contexts contexts, IInputService inputService) {
        _contexts = contexts;
        _inputService= inputService;
    }

    public void Initialize() {
        // use unique flag component to create an entity to store input components        
        _contexts.input.isInputManger = true;
        _inputEntity = _contexts.input.inputEntity;
    }

    // clean simple api, 
    // descriptive, 
    // obvious what it does
    // resistant to change
    // no using statements
    public void Execute () {
        inputEntity.isButtonAInput = _inputService.button1Pressed;
        inputEntity.ReplaceLeftStickInput(_inputService.leftStick);
        // ... lots more queries
    }
}
```

现在我希望你明白了我所说的抽象化，我将逻辑从具体视线中抽离出来。比如在输入系统中我只关系用户是否按下了按钮A，我不关心它是使用键盘还是鼠标还是手柄来按下的，对于我的系统来说这根本无所谓。

再比如TimeService，我只需要说，给我一个TimeDelta。然后我也根本不需要去关心这个TimeDelta是由Unity还是XNA还是Unreal给出的。我只需要知道那是多少就行。

## 控制反转
由于我们使用了接口，目前我们面临着一个新的问题，那就是我们需要以某种形式将接口的具体实例注入到接口中去。在上一个例子中，我通过构造函数来传递这个实例。但这也会导致系统将有很多的构造函数。我们希望让我们的Service实例成为可全局访问的，也希望我们的代码中有且仅有一处初始化方法使得我们可以确定我们的这些接口使用哪种实现。然后我们在哪里创建他们的实现并使其可以全局访问。这样我们就可以直接使用而不用通过构造函数传来传去。幸好这在Entitas中很容易实现。
 
``` Csharp
public class Services
{
    public readonly IViewService View;
    public readonly IApplicationService Application;
    public readonly ITimeService Time;
    public readonly IInputService Input;
    public readonly IAiService Ai;
    public readonly IConfigurationService Config;
    public readonly ICameraService Camera;
    public readonly IPhysicsService Physics;

    public Services(IViewService view, IApplicationService application, ITimeService time, IInputService input, IAiService ai, IConfigurationService config, ICameraService camera, IPhysicsService physics)
    {
        View = view;
        Application = application;
        Time = time;
        Input = input;
        Ai = ai;
        Config = config;
        Camera = camera;
        Physics = physics;
    }
}

// 然后我就可以这么注入他们
var _services = new Services(
    new UnityViewService(), // responsible for creating gameobjects for views
    new UnityApplicationService(), // gives app functionality like .Quit()
    new UnityTimeService(), // gives .deltaTime, .fixedDeltaTime etc
    new InControlInputService(), // provides user input
    // next two are monobehaviours attached to gamecontroller
    GetComponent<UnityAiService>(), // async steering calculations on MB
    GetComponent<UnityConfigurationService>(), // editor accessable global config
    new UnityCameraService(), // camera bounds, zoom, fov, orthsize etc
    new UnityPhysicsService() // raycast, checkcircle, checksphere etc.
);
```

在我的MetaContext中，我有一个唯一组件集合来持有这些系统的接口。
``` Csharp
[Meta, Unique]
public sealed class TimeServiceComponent : IComponent {
    public ITimeService instance;
}
```
然后我增加了一个注册用的Feature组件并在其构造方法中增加了一个Services参数。通过传递Services来初始化若干Service的实现。
``` Csharp
public class ServiceRegistrationSystems : Feature
{
    public ServiceRegistrationSystems(Contexts contexts, Services services)
    {
        Add(new RegisterViewServiceSystem(contexts, services.View));
        Add(new RegisterTimeServiceSystem(contexts, services.Time));
        Add(new RegisterApplicationServiceSystem(contexts, services.Application));
        Add(new RegisterInputServiceSystem(contexts, services.Input));
        Add(new RegisterAiServiceSystem(contexts, services.Ai));
        Add(new RegisterConfigurationServiceSystem(contexts, services.Config));
        Add(new RegisterCameraServiceSystem(contexts, services.Camera));
        Add(new RegisterPhysicsServiceSystem(contexts, services.Physics));
        Add(new ServiceRegistrationCompleteSystem(contexts));
    }
}

// 比如一个注册时间系统的例子
public class RegisterTimeServiceSystem : IInitializeSystem
{
    private readonly MetaContext _metaContext;
    private readonly ITimeService _timeService;

    public RegisterTimeServiceSystem(Contexts contexts, ITimeService timeService)
    {
        _metaContext = contexts.meta;
        _timeService = timeService;
    }

    public void Initialize()
    {
        _metaContext.ReplaceTimeService(_timeService);
    }
}
```
接下来我们就可以通过_contexts.meta.timeService.instance类似这样的调用来全局访问我们的Service。而我们将这种实现都放在了一起。这就使得我们不论是更换实现方式还是假造系统来进行测试都非常方便。我们通过**控制反转**这种方式解决了依赖性。

## 视图层抽象化
到目前为止，我们描述了上图左侧的Service接口部分，现在让我们看看右侧的View接口部分，原理基本一样。像我们之前说的，视图层就是用来展现游戏状态给玩家的，如动画音频渲染等。这一步的目标和之前一样，移除与特定游戏引擎或者第三方库的依赖，同时使代码变得整洁且具有可读性，让我们的系统不关心实际的实现细节。

一个简单的办法是使用一个保有GameObject引用的ViewComponent组件，然后可能再来一个简单的AssignViewComponent组件作为标签，标示我们需要将当前Entity作为一个GameObject，使其成为一个View。为了这么做你需要去写一个Reactive System来处里AssignView同时去选出没有添加View组件的entity（!entity.hasView）来确保我们不会重复添加View。同时在这个System中甚至Component中，你必须直接使用Unity的API。这对解耦没有任何好处。

我们可以使用上文描述的Service那样的方法来对View进行更进一步的抽象。思考下你的视图实际上需要哪些数据和方法然后写个接口出来。你的系统将使用这些接口来get/set这些数据。我管这个叫“ViewControllers”，这一层用来直接控制视图对象。因此他们将包含一些tranform相关数据，如（postion/scale/toration)以及一些其他的像tags，layers，names，enable/disable的数据。

本质上来说，View应该会和一个entity绑定，然后我的System可能需要从Entity和其他部分获取一些游戏数据。比如我需要一个entity和Contexts instance的引用当我开始构建view的时候。同时我也需要能够在Entitas层面销毁我的View。这里有一个简单的示例

``` Csharp
public interface IViewController {
    Vector2D Position {get; set;}
    Vector2D Scale {get; set;}
    bool Active {get; set;}
    void InitializeView(Contexts contexts, IEntity Entity);
    void DestroyView();
}
```
这是一个基于Unity的实现
``` Csharp
public class UnityGameView : Monobehaviour, IViewController {

    protected Contexts _contexts;
    protected GameEntity _entity;

    public Vector2D Position {
        get {return transform.position.ToVector2D();} 
        set {transform.position = value.ToVector2();}
    }

    public Vector2D Scale // as above but with tranform.localScale

    public bool Active {get {return gameObject.activeSelf;} set {gameObject.SetActive(value);} }

    public void InitializeView(Contexts contexts, IEntity Entity) {
        _contexts = contexts;
        _entity = (GameEntity)entity;
    }

    public void DestroyView() {
        Object.Destroy(this);
    }
}
```
我们需要一个Service去创建这些视图并将它们绑定在Entity上。以下给出一个IViewService接口和对应的Unity实现。
``` Csharp
// 一个Component用来保有ViewController引用
[Game]
public sealed class ViewComponent : IComponent {
    public IViewController instance;
}
// 一个接口定义了我需要的一些数据
public interface IViewService {   
    // create a view from a premade asset (e.g. a prefab)
    IViewController LoadAsset(Contexts contexts, IEntity entity, string assetName);
}
// 对应的Unity实现
public class UnityViewService : IViewService {
    public IViewController LoadAsset(Contexts contexts, IEntity entity, string assetName) {
        var viewGo = GameObject.Instantiate(Resources.Load<GameObject>("Prefabs/" + assetName));
        if (viewGo == null) return null;
        var viewController = viewGo.GetComponent<IViewController>();
        if (viewController != null) viewController.InitializeView(contexts, entity);
        return viewController;
    }
}
// 对应的资源加载System可以将资源绑定到View上
public class LoadAssetSystem : ReactiveSystem<GameEntity>, IInitializeSystem {
    readonly Contexts _contexts;
    readonly IViewService _viewService;

    // collector: GameMatcher.Asset
    // filter: entity.hasAsset && !entity.hasView

    public void Initialize() {    
        // grab the view service instance from the meta context
        _viewService = _contexts.meta.viewService.instance;
    }

    public void Execute(List<GameEntity> entities) {
        foreach (var e in entities) {
            // call the view service to make a new view
            var view = _viewService.LoadAsset(_contexts, e, e.asset.name); 
            if (view != null) e.ReplaceView(view);
        }
    }
}
// 一个抽象于View的postion System
public class SetViewPositionSystem : ReactiveSystem<GameEntity> {
    // collector: GameMatcher.Position;
    // filter: entity.hasPosition && entity.hasView
    public void Execute(List<GameEntity> entities) {
        foreach (var e in entities) {
            e.view.instance.Position = e.position.value;
        }
    }
}
```
这样就解耦了对Unity引擎的依懒性。

### 这样做的问题
一个明显的缺陷就是你在写一个和视图层交互的System，这对于我们之前说的逻辑层不需要知道其他资源绘制过程的原则是一种破坏。因此我们将使用“Event”来实现完全的视图层解耦。

## Event
下面我将使用Event来简化刚才的项目，来使视图层和逻辑层充分分离。


首先我们给PostionComponent加上一个Event属性，让代码生成器生成对应的监听器和事件系统。
``` Csharp
[Game, Event(true)] // generates events that are bound to the entities that raise them
public sealed class PositionComponent : IComponent {
    public Vector2D value;
}
```

代码生成器会自动生成**PositionListererComponent**组件和**IPositionListener**接口。我还会写一个**IEventListener**接口，来让所有的事件监听器都可以正确的在他们启动的时候初始化。
``` Csharp
public interface IEventListener {
    void RegisterListeners(IEntity entity);
}
```
现在我们不需要让ViewService中的LoadAsset方法返回视图组件或其他返回值，同时我们还要添加一些代码来初始化事件的监听。
``` Csharp
public UnityViewService : IViewService {

    // now returns void instead of IViewController
    public void LoadAsset(Contexts contexts, IEntity entity, string assetName) {

        // as before 
        var viewGo = GameObject.Instantiate(Resources.Load<GameObject>("Prefabs/" + name));
        if (viewGo == null) return null;
        var viewController = viewGo.GetComponent<IViewController>();
        if (viewController != null) viewController.InitializeView(contexts, entity);      
  
        // except we add some lines to find and initialize any event listeners
        var eventListeners = viewGo.GetComponents<IEventListener>();
        foreach (var listener in eventListeners) listener.RegisterListeners(entity); 
    }
}
```
现在我们可以摆脱所有的SetViewXXXSystem，因为我们不会告诉视图任何事情，我们会让Monobehaviours来监听所有的变化，比如position：
``` Csharp
public class PositionListener : Monobehaviour, IEventListener, IPositionListener {
    
    GameEntity _entity;
 
    public void RegisterEventListeners(IEntity entity) {
        _entity = (GameEntity)entity;
        _entity.AddPositionListener(this);
    }

    public void OnPosition(GameEntity e, Vector2D newPosition) {
        transform.position = newPosition.ToVector2();
    }
}
```
只要将这个脚本添加到对应的Prefab上，GameObject就可以与Entity的Position变化保持同步。这样就完成了逻辑层与视图层的彻底解耦。

这种通过Services来加载资源的方式也使得我们对初始化过程有了更强的控制能力，同时我们也可以通过添加IAudioPlayer，IAnimator等接口的方式来实现其他资源的加载，只要调整初始化顺序便可以避免以前类似Unity中的Start或者Update方法中类似的顺序问题。

现在视图层变成了一个简单的监听器，只响应他感兴趣的组件变化。同时不需要发送任何信息到逻辑层。