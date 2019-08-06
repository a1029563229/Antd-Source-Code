# Types

```tsx
// Props 类型
import * as PropTypes from 'prop-types';

// 返回一个字符串数组的方法，用于定义枚举类型
export const tuple = <T extends string[]>(...args: T) => args;

// 鼠标事件
React.MouseEventHandler<HTMLButtonElement | HTMLAnchorElement>
```