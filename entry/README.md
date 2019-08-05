# 入口文件

## index.js
```js
// 驼峰式写法
function camelCase(name) {
  return name.charAt(0).toUpperCase() + name.slice(1).replace(/-(\w)/g, (m, n) => n.toUpperCase());
}

// 使用 require.context 导出 components 下的所有 style 目录尾缀为 index.jsx 的文件
const req = require.context('./components', true, /^\.\/[^_][\w-]+\/style\/index\.tsx?$/);

req.keys().forEach(mod => {
  // 获取模块
  let v = req(mod);
  if (v && v.default) {
    v = v.default;
  }

  // 判断文件是否为 index.tsx 结尾
  const match = mod.match(/^\.\/([^_][\w-]+)\/index\.tsx?$/);
  if (match && match[1]) {
    if (match[1] === 'message' || match[1] === 'notification') {
      // message & notification should not be capitalized
      exports[match[1]] = v;
    } else {
      exports[camelCase(match[1])] = v;
    }
  }
});

// 导出 components 目录
module.exports = require('./components');
```