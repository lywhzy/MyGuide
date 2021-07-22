## 虚拟DOM

- DOM 浏览器的概念， 用JS对象来表示页面上的元素，并提供了操作DOM对象的API
- 虚拟DOM 是框架中的概念，用JS对象来模拟页面上的DOM和DOM嵌套
- 虚拟DOM的目的 为了实现页面上,DOM元素的高效更新  

- 本质： 用JS对象来模拟DOM元素和嵌套关系

## 在项目中使用react

### 运行cnpm i react react-dom -S 安装包

- react 专门用于创建组件和虚拟dom的，同时组件的生命周期都在这个包中
- react-dom 专门进行dom操作的，最主要的应用场景，就是render（）

### 创建虚拟dom元素

- react.createElement()
    - param1: 创建元素的类型
    - param2： 是一个对象或null 表示当前dom元素的属性
    - param3： 子节点
    - param4： 其它子节点
    
- ReactDOM.render()
    - param1: 虚拟dom对象
    - param2： 指定页面的dom元素当作容器
    
    
### JSX

#### 样式处理

````

<div style={ {color : 'red'} }/>

<div className="title"/> // 需要在其它地方定义title样式

<div className="css`.title{ color:red }`"> // 需要引入import { css } from 'emotion';

````

#### 列表渲染

````

const songs = [{ id : "1", name: "lyw"}]

const list = (
    <ul>
        {songs.map(item => <li key={item.id}>{item.name}</li>)}
    <ul>
)

````
    
## 创建组件的方式

### 通过function创建组件

````
// App即为一个新组件
function App() {
    return (
        //返回一个合法的JSX虚拟DOM元素
        <div></div>
    )
}

ReactDOM.render(<App/>,document.getElementById("root"));
 
````

### 通过类创建组件

````
class MyComp extends React.Component {
    constructor(props) {
        super(props);

    }

    render() {

        return (
            <div>
                123
            </div>
        )
    }

}

//导出组件 以供其它文件调用该组件
export default MyComp;

````

## state

- 函数组件没有自己的状态，是无状态组件
- 类组件有自己的状态，是有状态组件

### 从JSX中抽离事件处理程序

````
//第一种方法 箭头函数 因为箭头函数自身不绑定this，箭头函数this指向外部环境，也就是render方法
<div onclick = {() => {}} />

//第二种方法 在constructor中用bind方法将this与方法绑定在一起

constructor(props) {
    super(props);
    this.click = this.click.bind(this)        
}

//第三种方法 class实例方法
click = () => {

}

````

### 受控组件

````

类似于input/select这种表单组件，react不允许其自己更改value，而是通过在change事件中更改state来更改value

<input key={1} value={this.state.count} onChange={ (e)=> {this.setState({count : e.target.value})}}/>

````

### 非受控组件

````
constructor(props) {
    super(props);
    this.textRef = React.createRef()
}

<input type="1" ref={this.textRef}/>
````

## props 组件间通讯

- children属性 表示组件标签的子节点 

### 父组件给子组件传数据

````

<father name={this.state.name} />

class father extends React.Compoment{
    constructor(props) {
        super(props);
        console.log(props.name)
    }
}
< 

````

### 子组件给父组件传数据

````
//利用回调函数，父组件提供回调，子组件调用，将要传递的数据作为回调函数的参数

class father extends React.Compoment{
    constructor(props) {
        super(props);
    }

    change = (data) => {
        console.log(data)
    }

    render() {
        return(
            <child onchange={this.change}>
        )
    }
}

class child extends React.Compoment{
    constructor(props) {
        super(props);
    }

    render() {
        return(
            <div onClick={this.props.onchange(this.state.name)}>
        )
    }
}

````

### 兄弟组件间通讯

````

//将共享状态提升到最近的公共父组件中，由公共父组件管理这个状态

````

## props校验

- 在创建组件的时候 就可以指定props的类型，格式
- 作用 捕获使用组件时因为props导致的错误，给出明确的错误提示，增加组件的健壮性

## Context

    实现跨组件传递数据
    提供了两个组件：Provider和Consumer
    Provider：用来提供数据
    Consumer：用来消费数据
     
## 组件声明周期

