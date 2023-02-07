>博客地址 [# Build your own React](https://pomb.us/build-your-own-react/)
### 0. 概述
```Js
const element = <h1 title = "foo">Hello </h1>
const container = document.getElementById('root');
ReactDOM.render(element, container)
```
首先`element`是`JSX`元素，需要通过类似`Babel`的转换工具，将其转化为`JS`代码，对生成的`JS`带代码采用**createElement**进行处理。
```js
const element = React.createElement(
	"h1",
	{title: 'foo'},
	"Hello"
)
```
其中`React.createElement`根据输入生成对象，对象如下，其中着重关注`type`以及`props`属性，其中`type`定义了DOM类型，`props`是另一个对象，一部分是`JSX`的属性，另一部分`children`着代表了嵌套的DOM元素。
```js
const element = {
	type: 'h1',
	props: {
		title: 'foo',
		children: 'Hello'
	}
}
```
通过`render`函数对`element`进行渲染，先对于根元素进行渲染，然后对于`children`进行渲染，由于子节点为文本节点，最后将子节点添加入父结点中
```js
const node = document.createElement(elemetn.type);
node['title'] = element.props.title;

const text = document.createTextNode("");
text['nodeValue'] = element.props.children;

node.appendChild(text);
container.appendChild(node);
```
### 1. `createElement`函数
```js
const element = (
	<div id = "foo">
		<a>bar</a>
		<b/>
	</div>
)
```
该函数的主要作用是将`JSX`转化得到的`JS`转化为实际渲染所需要的`object`，如
```js
createElement('div', null, a, b)
# ->
{
	"type": "div",
	"props": {"children": [a,b]}
}
```
其主要作用就是通过`types props children`生成一个对象，如果`children`节点类型位`object`表明其位同`node`类型一致的节点，如果不是则代表了该节点为文本，需要通过函数生成文本节点。
```js
function createElement(type, props, ...children){
	reutrn {
		type,
		props: {
			...props,
			children: children.map(child => 
				typeof child === "object" ? child: createTextElement(child))
		}
	}
}
function createTextElement(text){
	return {
		type: "TEXT_ELEMENT",
		props: {
			nodeValue: text,
			children: [],
		}
	}
}
```
### 2. `render`函数
`render`函数主要是对传回的`element object`进行渲染，首先根据`element`的`type`决定生成的是文本节点还是元素节点，然后对`element`的`props`中的非`children`属性对生成的节点进行属性赋值，对`children`属性中的子节点进行递归渲染，最后将节点挂载到容器中。
```js
function render(element, container){
	const dom = element.type == "TEXT_ELEMENT" 
	? document.createTextNode('')
	: document.createElement(dom);
	const isProperty = key => key !== "children"
	Object.keys(element.props)
		.filter(isProperty)
		.forEach(name => {
		dom[name] = element.props[name]})
	element.props.children.forEach(child => render(child, dom))
	container.appendChild(dom);
}
```
### 3. 并发模式（Concurrent Mode）
上述的渲染模式存在一个问题，即开始进行渲染`element`树的时候，直至渲染完成，否则无法终止，这对于过大的`element`树会阻塞主进程很长的时间，导致浏览器无法处理用户的其它操作，以及保证动画的渲染稳定。
因此，我们需要将工作拆分成小单元结构，当其中结束其中一个单元时，且需要执行其它人物的时候，会中断渲染流程。
