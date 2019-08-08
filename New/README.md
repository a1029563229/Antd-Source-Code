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

export default class Row extends React.Component<RowProps, RowState> {
  // ...
  renderRow = ({ getPrefixCls }: ConfigConsumerProps) => {
    // ...
    // 取 gutter 赋值给 style 中的 marginLeft 和 marginRight
    // 这里是赋值负数，暂时不太清楚用途是什么，效果是可以让父级的宽度更宽
    // 感叹号是非null和非undefined的类型断言
    const rowStyle =
      gutter! > 0
        ? {
            marginLeft: gutter! / -2,
            marginRight: gutter! / -2,
            ...style,
          }
        : style;
    const otherProps = { ...others };
    delete otherProps.gutter;
    return (
      // 通过 RowContext.Provider（通过 React.createContext 创建） 组件的 value 属性将 gutter 传递给子组件（col）
      // React.createContext：创建一个 Context 对象。当 React 渲染一个订阅了这个 Context 对象的组件，
      // 这个组件会从组件树中离自身最近的那个匹配的 Provider 中读取到当前的 context 值
      // Context.Provider：每个 Context 对象都会返回一个 Provider React 组件，它允许消费组件订阅 context 的变化。
      // Provider 接收一个 value 属性，传递给消费组件。一个 Provider 可以和多个消费组件有对应关系。多个 Provider 也可以嵌套使用，里层的会覆盖外层的数据。
      // 当 Provider 的 value 值发生变化时，它内部的所有消费组件都会重新渲染。
      // Provider 及其内部 consumer（消费者） 组件都不受制于 shouldComponentUpdate 函数，因此当 consumer 组件在其祖先组件退出更新的情况下也能更新。
      // 通过新旧值检测来确定变化，使用了与 Object.is 相同的算法。
      // Context.Consumer：这里，React 组件也可以订阅到 context 变更。这能让你在函数式组件中完成订阅 context。
      // 这需要函数作为子元素（function as a child）这种做法。这个函数接收当前的 context 值返回一个 React 节点。
      // 传递给函数的 value 值等同于往上组件树这个 context 最近的 Provider 提供的 value 值。
      // 如果没有对应的 Provider，value 参数等同于传递给 createContext() 的 defaultValue。
      <RowContext.Provider value={{ gutter }}>
        <div {...otherProps} className={classes} style={rowStyle}>
          {children}
        </div>
      </RowContext.Provider>
    );
  };

  render() {
    return <ConfigConsumer>{this.renderRow}</ConfigConsumer>;
  }
}

// 利用 generator 进行参数控制
// 在返回的函数中传入主体
// 属于函数柯里化和装饰器模式
function generator({ suffixCls, tagName }: GeneratorProps) {
  return (BasicComponent: React.ComponentClass<BasicPropsWithTagName>): any => {
    return class Adapter extends React.Component<BasicProps, any> {
      static Header: any;
      static Footer: any;
      static Content: any;
      static Sider: any;

      renderComponent = ({ getPrefixCls }: ConfigConsumerProps) => {
        const { prefixCls: customizePrefixCls } = this.props;
        const prefixCls = getPrefixCls(suffixCls, customizePrefixCls);

        return <BasicComponent prefixCls={prefixCls} tagName={tagName} {...this.props} />;
      };

      render() {
        return <ConfigConsumer>{this.renderComponent}</ConfigConsumer>;
      }
    };
  };
}

// React.cloneElement(element, [props], [...children])
// 以 element 元素为样板克隆并返回新的 React 元素。返回元素的 props 是将新的 props 和原始元素的 props 浅层合并后的结果。
// 新的子元素将取代现有的子元素，来自原始元素的 key 和 ref 将被保留
// React.cloneElement() 几乎等同于：<element.type {...element.props} {...props}>{children}</element.type>
import { cloneElement } from 'react';

// 通过使用 onFieldsChange 与 mapPropsToFields，可以把表单的数据存储到上层组件或者 Redux、dva 中，更多可参考 rc-form 示例。
// 注意：mapPropsToFields 里面返回的表单域数据必须使用 Form.createFormField 包装。
const CustomizedForm = Form.create({
  name: 'global_state',
  onFieldsChange(props, changedFields) {
    props.onChange(changedFields);
  },
  mapPropsToFields(props) {
    return {
      username: Form.createFormField({
        ...props.username,
        value: props.username.value,
      }),
    };
  },
  onValuesChange(_, values) {
    console.log(values);
  },
})(props => {
  const { getFieldDecorator } = props.form;
  return (
    <Form layout="inline">
      <Form.Item label="Username">
        {getFieldDecorator('username', {
          rules: [{ required: true, message: 'Username is required!' }],
        })(<Input />)}
      </Form.Item>
    </Form>
  );
});

class Demo extends React.Component {
  state = {
    fields: {
      username: {
        value: 'benjycui',
      },
    },
  };

  handleFormChange = changedFields => {
    this.setState(({ fields }) => ({
      fields: { ...fields, ...changedFields },
    }));
  };

  render() {
    const { fields } = this.state;
    return (
      <div>
        <CustomizedForm {...fields} onChange={this.handleFormChange} />
        <pre className="language-bash">{JSON.stringify(fields, null, 2)}</pre>
      </div>
    );
  }
}
```
