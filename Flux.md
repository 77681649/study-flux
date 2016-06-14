

# Flux
---

## Flux 是什么?
Flux是一个架构思想 , 是Facebook提出的用于组织Web应用开发的一种架构思想实现.

**核心思想 -- 单向数据流**

Flux的核心思想就是让数据在应用中**单向的流动**

那么 , 所谓的单向数据流具体是怎么样的呢?

### Flux 单向数据流

为更好的理解 , 先解释一下Flux中用到的术语 :  

**Action**       

对应用中出现的一些操作的统称 , 具体如UI事件,初始化,Ajax响应等等... 

Action 具体的功能 : 
1. 描述发生了什么 ( 既不关注操作是如何实现的 , 也不关注如何处理这个操作 , 仅仅只是描述而已 )
2. 承载变动的数据


**ActionCreater**
Action的构造器

**Dispatcher**     

其实就是已经很司空见惯的事件器 , 在Flux中的也是类似的概念

Dispatcher 具体的功能 : 
1. 注册处理Action的回调函数
2. 分发Action

**Store**

用来存储数据和封装数据操作


**View**

传统的视图层


下面 , 就来具体讲解Flux的单向数据流到底是怎么样一回事.


当应用中出现一个操作时 , 在Flux架构中会经过如下流程 : 

![Flux](http://facebook.github.io/flux/img/flux-simple-f8-diagram-1300w.png)

1. 生成描述这个操作的Action

2. 通过Dispatcher将Action分发出去

3. 对该Action感兴趣的Store做出相应的操作后 , 通知View更新UI

这就是Flux中的一次数据流转 , 数据的流向是单向的 , 不会出现反向的数据流.

那么问题来 , 

**Flux是如何做到数据的单向流动呢?**

Flux是通过**控制数据更新的方式和时机** , 以达到控制数据流动的目的.
具体表现如下 : 

1. **[ 保证  操作 --> Action --> Dispatcher ]** 对数据的更新必须通过action的分发 ( dispatcher(action) )    
2. **[ 保证  Dispatcher --> Store ]** Store通过在Dispatcher注册 , 以便被分发的Action 
3. **[ 保证  Store --> View ]** 数据在Store中完成更新 , 在Store中通知View
4. **[ 保证  数据流的顺序执行]** 数据流中不允许出现异步操作
5. **[ 保存  数据流的串行执行]** 当要dispatch action时 , 必须等到上一次Action分发完成 ; 否则 , Flux会抛出异常 
6. **[ 杜绝  出现 View --> Store , XX --> Store]** Store**不公开能直接修改数据的方法** , 仅提供数据的查询方法 , 要变动Store中的数据必须通过分发Action

这样 , 在Flux的数据流就会单向流动的了



**那么数据的单向流动有什么好处呢?**

**使得应用具有可预测性** -- 这也是Flux的最主要的动机

可预测性的具体表现 : 

1. 一个Action的分发 , 开发者可以清晰的预测他的结果 ( 因为是单向顺序 , 无其他Action干扰执行数据流的 ) 

2. 数据变化点的集中 ,  可以让开发者更容易知道数据的变化是在哪里发生的


说了这么多 , 下面看看具体代码是如何实现单向数据流的




具体的代码实现如下 : 
( 这里只给出最简单的代码结构 )

````

// Dispatcher
// dispatcher.js
// Flux以单例模式实例化dispatcher
// 多Dispatcher容易造成混乱

const Dispatcher = require('flux').Dispatcher;
module.exports = new Dispatcher();



// Store
// listStore.js
// 这是Flux中最重要的地方 
// 他负责存储数据和实现所有业务逻辑 , 并且还负责通知View

const dispatcher = require('./dispatcher');

// mock一个事件器
const events = mock(()=>{
    return {
        on:(evt,cb)=>{},
        trigger:(evt,...args)=>{}
    }
});


const listStore = {
    _list:[],

    // 公开的获得数据的方法
    getAll:function(){
        return this._list;
    },

    // 通知View有数据变化
    // Store --> View
    triggerChangeListener:function(){
        events.trigger('change');  
    },

    onChangeListener:function(callback){
        events.on('change' , callback);
    },

    offChangeListener:function(callback){
        events.off('change' , callback);
    }
};

// 修改数据的方法是私有方法 , 不会对外公开
function setList(list){
    listStore._list = list;
}

// register dispatch callback
// 当有Action分发时 , Dispatcher会依次调用所有注册了的回调函数
// 以便通知对该Action感兴趣的Store
// Dispatcher --> Store
dispatcher.register((action)=>{
    switch(action.type){
        case 'INIT_LIST':
            listStore.init(action.list);
            listStore.triggerChangeListener();
            break;
        
        case 'CLEAR_LIST':
            listStore.clear();
            listStore.triggerChangeListener();
            break;

        default:
            break;
    }
});

module.exports = listStore;



// Action Creaters
// actions.js
// 专门用来创建Action的一些函数

const dispatcher = require('./dispatcher');

module.exports = {

    // Action --> Dispatcher
    initList:function(list){
        dispatcher.dispatch({
            type:'INIT_LIST',
            list:list
        })
    },

    // Action --> Dispatcher
    clearList:function(){
        dispatcher.dispatch({ 
            type:'CLEAR_LIST'
        })
    }
}



// View
// 这里以React为例
const actions = require('./actions');
const listStore = require('./listStore');

var List = React.createClass(
    render:function(){
        var items = this.props.data.map((it)=><Item name={it.name} />)
        
        return (
          <ul>{items}</ul>  
        );
    }
)

var Item = React.createClass(
    render:function(){
        return (
          <li>{this.props.name}</li>
        );
    }
)

var ClearButton = React.createClass(

    render:function(){
        return (
            // UI Event : View --> ActionCreater
            <a href="#" onClick={actions.clearList}>Clear</a>
        )
    }
)



// Controller-view
var App = React.createClass(
    
    getInitialState:function(){
        return {
            list : []
        };
    },

    componentWillMount:function(){

        // 从Server获得初始数据
        // 为什么不把异步操作放在Action , Store , Dispatcher中呢?
        // 因为 , 这样会破坏Flux的单向数据流
        $.ajax({
            url : this.props.url,
            success:function(data){

                // Ajax : View --> ActionCreater
                actions.initList(data);
            },
            error:function(){ 
                // ...
            } 
        }
    },

    // 绑定事件
    componentDidMount:function(){
        listStore.onChangeListener(this._onChange);
    },

    // 卸载事件
    componentWillUnmount:function(){
        listStore.offChangeListener(this._onChange);
    },
    
    // 这里完成最后的UI状态的更新
    _onChange:function(){
        var list = listStore.getAll();

        this.setState({list:list})
    },

    render:function(){
        return (
            <List data={this.state.list} />
            <ClearButton />
        )
    }
)


ReactDOM.render(
    <App url='http://localhost/api/list' />,
    $('body')[0]
)

```


下面的图就是Flux的完整数据流 ( 从数据的更新到View的更新形成了一个完整的闭环 )

![Flux](http://172.18.23.67/images/flux/flux.png)


## Flux vs MVC


看了上面的内容 , 大家应该对Flux有了一定的了解 .那么 , 现在就来看看Flux架构与MVC架构相比到底有哪些优势和局限性.

MVC中最大的问题就是,数据流动的双向性.

当一个数据的变革涉及到多个Model,View时 , 这时要理清数据流的流程将会是一件很困难的事.

具体体现在如下如下几点 : 

1. MVC中Model可能在任意一个地方被更改 

