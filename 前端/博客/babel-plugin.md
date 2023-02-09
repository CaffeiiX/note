[AST与前端编译](https://mp.weixin.qq.com/s/tDIIZUV4sDZ0lkPX0FPAHQ)
## 简介
本文主要是对于ast的介绍,以及通过ast实现相关的babel-plugin的过程。
## ast的生成
[ast在线生成网站](https://astexplorer.net/)
### 词法分析
词法分析的过程是将代码喂给有限状态机，结果是将代码单词转换为令牌(token)，一个token包含的信息包括其的种类、属性值
对于将`const a = 1 + 1`转换为token
```javascript
[  
  {type: 关键字, value: const},   
  {type: 标识符, value: a},  
  {type: 赋值操作符, value: =},  
  {type: 常数, value: 1},  
  {type: 运算符, value: +},   
  {type: 常数, value: 1},  
]
```
### 语法分析
获取第一个`token`，生成第一个`ast`节点，然后移动到下一个`token`，将其转换成`ast`子节点，添加到现有的`ast`中，然后重复这个移动&深沉的递归过程
对于`const a = 1`，生成ast的过程
1.  读取const，生成一个VariableDeclaration节点
2.  读取a，新建VariableDeclarator节点
3.  读取=
4.  读取1，新建NumericLiteral节点
5.  将NumericLiteral赋值给VariableDeclarator的init属性
6.  将VariableDeclarator赋值给VariableDeclaration的declaration属性
```javascript
{
    "type": "VariableDeclaration",
    "start": 533, //源码中的起始结束位置
    "end": 545,
    "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 539,
          "end": 544,
          "id": {
            "type": "Identifier",
            "start": 539,
            "end": 540,
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "start": 543,
            "end": 544,
            "value": 1,
            "raw": "1"
          }
        }
      ],
      "kind": "const"
}
```
## 前端编译
**编译**：将代码转换为可以直接在浏览器中执行的代码，如将`typescript、jsx`等转换为可以直接在浏览器中执行的代码。
**编译工具**
1.  babel：目前最主流的编译工具，使用javascript编写。（访问者模式）
2.  esbuild：使用Go语言开发的打包工具（也包含了编译功能）, 被Vite用于开发环境的编译。（不直接提供操作ast的能力）
3.  swc：使用rust编写的编译工具。（访问者模式）
**编译过程**
- `parse`
将代码从字符串转化为树状结构的ast
- `transform`
对ast节点进行遍历，遍历过程中可以对ast进行修改
- `generate`
根据ast生成代码
## hello plugin
实现的功能：打印函数的执行时间
```javascript
export default function(){
    return{
        visitor: {
	        // 对一种ast节点进行访问  
	        //FunctionDeclaration: {  
	        //    enter: enterFunction,  
	        // 在babel对ast进行深度优先遍历时，  
            // 我们有enter和exit两次机会访问同一个ast节点。   
	        //    exit: exitFunction  
	        //},  
	        // 对多种ast节点进行访问  
	        //"FunctionDeclaration|FunctionExpression|ArrowFunctionExpression": {  
	            //enter: enterFunction  
	        //}  
	        // 使用“别名”进行访问  
	        Function: functionEnter
        }
    }
};
// path代表访问的路径,state代表属性
const functionEnter = (path, state) => {
//获取节点
    const node = path.node;
    const fnName = node.name || node.id?.name || "anonymous";
    const varName = `${fnName}_start_time`;
    const start = template(
        `const ${varName} = Date.now()`
    )();
    if(!node.body){
        return;
    }
    path.get('body').unshiftContainer('body', start);
    //遍历ast
    path.traverse({
        Function(innerPath){
            innerPath.skip();
        },
        //返回状态执行returnEnter函数
        ReturnStatement: returnEnter
    }, {fnName})
}
const returnEnter = (path, state) =>  {
//return的数值
    const {fnName} = state;
    const resultVar = identifier(`${fnName}_result`);
    //生成 const xx = xx的语句
    const returnResult = template(`const RESULT_VAR = RESULT`)({
        RESULT_VAR: resultVar,
        RESULT: path.node.argument || identifier('undefined')
    });
    //在return之前插入
    path.insertBefore(returnResult);
    
    const varName = `${fnName}_start_time`;
    const end = template(
        `console.log('${fnName} cost:', Date.now() - ${varName})`
    )();
    //return console.log(Date.now() - xx), 
    const ast = sequenceExpression([
        end.expression,
        resultVar
    ]);
    path.node.argument = ast;
}
```