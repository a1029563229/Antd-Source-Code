# 新知识（个人）

```tsx
// Props 类型
import * as PropTypes from "prop-types";

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
    block: PropTypes.bool
  };

  // getDerivedStateFromProps 会在调用 render 方法之前调用，并且在初始挂载及后续更新时都会被调用。
  // 它应该返回一个对象来更新 state，如果返回 null 则不更新任何内容
  // 此方法适用于罕见的用例，即 state 的值在任何时候都取决于 props
  static getDerivedStateFromProps(
    nextProps: ButtonProps,
    prevState: ButtonState
  ) {
    // 如果 loading 发生改变并且类型为 Boolean 值，则更新 state
    if (nextProps.loading instanceof Boolean) {
      return {
        ...prevState,
        loading: nextProps.loading
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

// ReactDOM.findDOMNode(component) 如果组件已经被挂载到 DOM 上，此方法会返回浏览器中相应的原生 DOM 元素。
// 此方法对于从 DOM 中读取值很有用，例如获取表单字段的值或者执行 DOM 检测。
// 大多数情况下，你可以绑定一个 ref 到 DOM 节点上，可以完全避免使用 findDOMNode
// 注意：findDOMNode 是一个访问底层 DOM 节点的应急方案（escape hatch）。在大多数情况下，不推荐使用该方法，因为它会破坏组件的抽象结构。
import { findDOMNode } from "react-dom";

// anim(el,animationName,function(){}); 使用这个方法可以让 el 节点播放指定 animationName 动画
import TransitionEvents from "css-animation/lib/Event";

// Where el is the DOM element you'd like to test for visibility
function isHidden(element: HTMLElement) {
  if (process.env.NODE_ENV === "test") {
    return false;
  }

  // 元素是否存在
  // offsetParent：在 Webkit 中，如果元素为隐藏的（该元素或其祖先元素的 style.display 为 "none"），或者该元素的 style.position 被设为 "fixed"，则该属性返回 null。
  return !element || element.offsetParent === null;
}

// Window.getComputedStyle()方法返回一个对象，该对象在应用活动样式表并解析这些值可能包含的任何基本计算后报告元素的所有CSS属性的值。
// 私有的CSS属性值可以通过对象提供的API或通过简单地使用CSS属性名称进行索引来访问。
// 这里是按照优先级获取 node 的几个主色调
const waveColor =
  getComputedStyle(node).getPropertyValue("border-top-color") || // Firefox Compatible
  getComputedStyle(node).getPropertyValue("border-color") ||
  getComputedStyle(node).getPropertyValue("background-color");
this.clickWaveTimeoutId = window.setTimeout(
  () => this.onClick(node, waveColor),
  0
);

// 响应css媒体查询的轻量级javascript库
const enquire = require("enquire.js");
```
