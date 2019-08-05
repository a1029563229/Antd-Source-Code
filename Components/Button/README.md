# Button

## index.tsx
```tsx
import Button from './button';
import ButtonGroup from './button-group';

// 从 button 中导出一些 Interface 供开发者使用
export { ButtonProps, ButtonShape, ButtonSize, ButtonType } from './button';
export { ButtonGroupProps } from './button-group';

// 赋值
Button.Group = ButtonGroup;
export default Button;
```

## button.tsx
```tsx
// ...
// 让程序可以使用旧版本 react 的兼容新版本生命周期，而不需要更新版本
import { polyfill } from 'react-lifecycles-compat';
// 在对象中排除被忽略的字段，返回一个新的对象 omit(obj: Object, fields: string[]): Object
import omit from 'omit.js';

const rxTwoCNChar = /^[\u4e00-\u9fa5]{2}$/;
// 是否为两个中文字符（用于 两字符则自动填充空格）
const isTwoCNChar = rxTwoCNChar.test.bind(rxTwoCNChar);

// 给子节点添加空格
function spaceChildren(children: React.ReactNode, needInserted: boolean) {
  let isPrevChildPure: boolean = false;
  const childList: React.ReactNode[] = [];
  // React 的 API，可以更方便的遍历子节点
  React.Children.forEach(children, child => {
    const type = typeof child;
    const isCurrentChildPure = type === 'string' || type === 'number';
    // 上一个节点和当前节点均为 Pure 节点
    if (isPrevChildPure && isCurrentChildPure) {
      const lastIndex = childList.length - 1;
      const lastChild = childList[lastIndex];
      childList[lastIndex] = `${lastChild}${child}`;
    } else {
      childList.push(child);
    }

    isPrevChildPure = isCurrentChildPure;
  });

  // Pass to React.Children.map to auto fill key
  return React.Children.map(childList, child =>
    insertSpace(child as React.ReactChild, needInserted),
  );
}

// ...
```