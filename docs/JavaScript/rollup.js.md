## 基本用法

命令行的基本用法

```bash
$ rollup main.js --file bundle.js
```

如果希望打包后代码最小化，使用参数`--compact`。

```bash
$ rollup main.js --compact
```

配置可以写进配置文件`rollup.config.js`。

```javascript
// rollup.config.js
export default {
  input: 'main.js',
  output: {
    file: 'bundle.js',
    format: 'es'
  }
};
```

参数`-c`或`--config`启用配置文件。

```bash
$ rollup -c
```

## 模块名解析插件

Rollup 不认识 CommonJS 的模块名加载语法，需要使用插件`@rollup/plugin-node-resolve`。

```bash
$ npm install @rollup/plugin-node-resolve --save-dev
```

然后，在配置文件里面使用该插件。

```javascript
import { nodeResolve } from "@rollup/plugin-node-resolve";
...

export default {

  external: [/node_modules/], // <-- Add `node_modules` as regex
  plugins: [
    nodeResolve(),  // <-- Resolver to handle node modules
  ],
};
```

## rollup.config.js 示例

```javascript
import { nodeResolve } from "@rollup/plugin-node-resolve";

export default {
  input: 'src/index.js',
  output: {
    file: 'build/bundle.js',
    format: 'es',
    compact: true,
  },
  external: [/node_modules/], // <-- Add `node_modules` as regex
  plugins: [
    nodeResolve(),  // <-- Resolver to handle node modules
  ],
};
```

HTML 网页的加载代码如下。

```html
    <script type="importmap">
  {
    "imports": {
      "uppy": "https://releases.transloadit.com/uppy/v4.17.0/uppy.min.mjs"
    }
  }
    </script>
    <script type="module" src="build/bundle.js"></script>  
```