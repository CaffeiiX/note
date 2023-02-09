[从零实现一个迷你webpacck](https://mp.weixin.qq.com/s/e0ibggrhNdr_ZAGvxxok2Q)
## 前置知识
对于下列的代码，通过webpack最简单的配置，将index.js文件作为入口进行打包，具体如下：
```javascript
// index.js  
require('./a.js');  
console.log('entry load');  
  
// a.js  
require("./b.js");  
const a = 1;  
console.log("a load");  
module.exports = a;  
  
// b.js  
console.log("b load");  
const b = 1;  
module.exports = b;
```
![[Pasted image 20230113092334.png]]
可见打包结果是一个立即执行函数，首先初始化了一个`__webpack_modules`，其中的每个`module`都是文件内的具体代码，同时由于浏览器不支持`require`方法，所以内部实现了一个`__webpack__require__`方法，在定义`module`后，就开始执行入口函数，weppack的打包过程就是通过入口文件，将直接依赖和间接依赖以`module`的方法聚合起来，并通过`__webpack__require__`实现模块的同步加载。

`__webpack__require__`：这部分的实现其实是通过key来查找具体函数，然后执行函数的方法。由于`__webpack_modules_`的存储结构是`key: function`的形式，在执行`module`的函数时候，通过`__webpack__require__(key)`获取具体的执行函数。

## 初始化参数
```javascript
// 正常的webpack流程
const webpack = require('webpack'); // 引用 webpack  

options = {
	entry: './index.js',
	output: {
		path: path.resolve(__dirname, "dist"),
		filename: "[name].js"
	},
	module: {
		rules: [
			{
				test: /\.css$/i,
				use: ['style-loader', 'css-loader']
			}
		] //根据规则使用对应的loader对文件进行处理
	}
}

const compiler = webpack(options); // 传入配置生成 compiler 对象  

compiler.run((err, stats) => {  // 执行编译, 传入回调  
  
});
```
## 编译
根据上述对于通过编译文件得到的文件，可知主要需要解决的是如下几个问题
1. 读取得到入口文件，根据rule的配置，讲入口文件交给loader进行处理，返回处理后的代码
2. 编译loader处理完后的代码
3. 分析代码中的依赖，并进行如下操作：
	- 一个依赖文件对应一个`moduleId`，得到`webpack_modules = {moduleId: ()=>{}}`
	- 对于依赖文件中还有`require`的，将其转化为`webpack_require`
	- 实现`webpack_require`，`(moduleId) => {webpack_modules[moduleId]()}`
4. 对每个文件的处理结果，将文件的编译结果从初始的入口文件组织成chunk
执行流程如：[[webpack执行流程]]
### 整体代码
```javascript
const fs = require('fs');
const path = require('path');
const parser = require('@babel/parser');
const { default: traverse } = require('@babel/traverse');
const { default: generate } = require('@babel/generator');
const t = require('@babel/types')
require("@babel/generator");
require('@babel/traverse');
/**
 * @type {import('@babel/core').TransformOptions;}
 */
class Compiler {
    constructor(option){
        this.options = option || {};
        this.modules = new Set();
    }
    run(callback){
        //根据entry路径处理得到模块
        const entryModule = this.build(path.join(process.cwd(), this.options.entry));
        const entryChunk = this.buildChunk('entry', entryModule);
        this.generateFile(entryChunk);
    }
    build(modulePath){
        //根据module路径读取代码
        let originCode = fs.readFileSync(modulePath);
        //处理loader部分
        originCode = this.dealWidthLoader(modulePath, originCode.toString());
        //处理文件依赖
        return this.dealDependencies(originCode, modulePath);
    }
    //根据rule,找到对应的文件，对其进行loader的处理
    dealWidthLoader(modulePath, originCode){
        [...this.options.module.rules].reverse().forEach(item => {
            if(item.test(modulePath)){
                const loaders = [...item.use].reverse();
                loaders.forEach(loader => originCode = loader(originCode))
            }
        })
        return originCode;
    }
    //处理依赖
    dealDependencies(code, modulePath){
        const fullPath = path.relative(process.cwd(), modulePath);
        const module = {
            id: fullPath,
            dependencies: []
        };
        //对代码进行ast的处理
        const ast = parser.parse(code, {
            sourceType: 'module',
            ast: true
        });
        //遍历ast，找到require的节点，对其修改成__webpack_rquire__，并存储到模块的依赖数组中
        traverse(ast, {
            CallExpression: (nodePath) => {
                const node = nodePath.node;
                if (node.callee.name == "require") {
                    const requirePath = node.arguments[0].value;

                    const moduleDirName = path.dirname(modulePath);
                    const fullPath = path.join(moduleDirName, requirePath);
                    const argPath = path.relative(moduleDirName,path.join(moduleDirName, requirePath));

                    node.callee = t.identifier('__webpack_require__');
                    node.arguments = [t.stringLiteral(argPath)];
                    
                    const exitModule = [...this.modules].find(item => item.id === fullPath);
                    if (!exitModule) {
                        module.dependencies.push(fullPath);
                    }
                }
            }
        });
        //根据ast生成代码
        const {code: compilerCode} = generate(ast);
        module._source = compilerCode;
        //对于文件中获取的每个依赖路径，对其文件进行重新遍历，实现一个递归的处理
        module.dependencies.forEach((dependency) => {
            const depModule = this.build(dependency);
            this.modules.add(depModule);
        })
        return module;
    }
    //生成chunk，根据module生成chunk
    buildChunk(entryName, entryModule){
        return{
            name: entryName,
            entryModule: entryModule,
            modules: this.modules
        }
    }
    //根据chunk生成文件，并写入文件
    generateFile(entryChunk){
        const code = this.getCode(entryChunk);
        if(!fs.existsSync(this.options.output.path)){
            fs.mkdirSync(this.options.output.path);
        }
        fs.writeFileSync(
            path.join(
                this.options.output.path,
                this.options.output.filename.replace('[name]', entryChunk.name)
            ),
            code
        );
    }
    //根据chunk内的具体参数生成代码
    getCode(entryChunk){
        return `
        (() => {
            var __webpack_modules__ = {
                ${[...entryChunk.modules].map(module => `
                "${module.id}": (module, __unused_webpack_exports, __webpack_require__) => {
                    ${module._source}
                }
                `).join(',')}
            };
            var __webpack_module_cache__ = {};
            function __webpack_require__(moduleId) {
                // Check if module is in cache
                var cachedModule = __webpack_module_cache__[moduleId];
                if (cachedModule !== undefined) {
                  return cachedModule.exports;
                }
                // Create a new module (and put it into the cache)
                var module = (__webpack_module_cache__[moduleId] = {
                  exports: {},
                });
        
                // Execute the module function
                __webpack_modules__[moduleId](
                  module,
                  module.exports,
                  __webpack_require__
                );
        
                // Return the exports of the module
                return module.exports;
            }
            var __webpack_exports__ = {};
            (() => {
                ${entryChunk.entryModule._source};
            })();
        })();
        `
    }
}
module.exports = Compiler;
```