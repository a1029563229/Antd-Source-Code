# 新知识（个人）

```tsx
// Props 类型
import * as PropTypes from 'prop-types';

// 返回一个字符串数组的方法，用于定义枚举类型
export const tuple = <T extends string[]>(...args: T) => args;

class Button extends React.Component<ButtonProps, ButtonState> {
  //...
  // 定义了 Props 类型
  static propTypes = {
    type: PropTypes.string,
    // 指定 prop 只能是特定的值，指定它为枚举类型
    shape: PropTypes.oneOf(ButtonShapes),
    size: PropTypes.oneOf(ButtonSizes),
    htmlType: PropTypes.oneOf(ButtonHTMLTypes),
    onClick: PropTypes.func,
    // 一个对象可以是几种几种类型中的任意一个类型
    loading: PropTypes.oneOfType([PropTypes.bool, PropTypes.object]),
    className: PropTypes.string,
    icon: PropTypes.string,
    block: PropTypes.bool,
  };

  // getDerivedStateFromProps 会在调用 render 方法之前调用，并且在初始挂载及后续更新时都会被调用。
  // 它应该返回一个对象来更新 state，如果返回 null 则不更新任何内容
  // 此方法适用于罕见的用例，即 state 的值在任何时候都取决于 props
  static getDerivedStateFromProps(nextProps: ButtonProps, prevState: ButtonState) {
    // 如果 loading 发生改变并且类型为 Boolean 值，则更新 state
    if (nextProps.loading instanceof Boolean) {
      return {
        ...prevState,
        loading: nextProps.loading,
      };
    }
    return null;
  }

  isNeedInserted() {
    const { icon, children } = this.props;
    // React.Children.count 返回 children 中的组件总数量，等同于通过 map 或 forEach 调用回调函数的次数
    return React.Children.count(children) === 1 && !icon;
  }
  //...
}
```