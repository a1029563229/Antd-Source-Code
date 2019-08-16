# 新知识（个人）

```tsx
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
            ...style
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

        return (
          <BasicComponent
            prefixCls={prefixCls}
            tagName={tagName}
            {...this.props}
          />
        );
      };

      render() {
        return <ConfigConsumer>{this.renderComponent}</ConfigConsumer>;
      }
    };
  };
}

// 通过使用 onFieldsChange 与 mapPropsToFields，可以把表单的数据存储到上层组件或者 Redux、dva 中，更多可参考 rc-form 示例。
// 注意：mapPropsToFields 里面返回的表单域数据必须使用 Form.createFormField 包装。
const CustomizedForm = Form.create({
  name: "global_state",
  onFieldsChange(props, changedFields) {
    props.onChange(changedFields);
  },
  mapPropsToFields(props) {
    return {
      username: Form.createFormField({
        ...props.username,
        value: props.username.value
      })
    };
  },
  onValuesChange(_, values) {
    console.log(values);
  }
})(props => {
  const { getFieldDecorator } = props.form;
  return (
    <Form layout="inline">
      <Form.Item label="Username">
        {getFieldDecorator("username", {
          rules: [{ required: true, message: "Username is required!" }]
        })(<Input />)}
      </Form.Item>
    </Form>
  );
});

class Demo extends React.Component {
  state = {
    fields: {
      username: {
        value: "benjycui"
      }
    }
  };

  handleFormChange = changedFields => {
    this.setState(({ fields }) => ({
      fields: { ...fields, ...changedFields }
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

class Field {
  constructor(fields) {
    // 将 fields 属性赋值给当前类
    // 批量赋值
    Object.assign(this, fields);
  }
}

// 依次降序判断入参，智能判断
// 猜测是用于 form.validateFields([fieldNames: string[]], [options: object], callback(errors, values))
export function getParams(ns, opt, cb) {
  let names = ns;
  let options = opt;
  let callback = cb;
  if (cb === undefined) {
    if (typeof names === "function") {
      callback = names;
      options = {};
      names = undefined;
    } else if (Array.isArray(names)) {
      if (typeof options === "function") {
        callback = options;
        options = {};
      } else {
        options = options || {};
      }
    } else {
      callback = options;
      options = names || {};
      names = undefined;
    }
  }
  return {
    names,
    options,
    callback
  };
}

// 利用 lastIndexOf 来判断是否是以 prefix 为前缀的字符串
export function startsWith(str, prefix) {
  return str.lastIndexOf(prefix, 0) === 0;
}

// 利用 reduce 重置组件（对象）的值，并返回改变后的对象
return names.reduce((acc, name) => {
  const field = fields[name];
  if (field && "value" in field) {
    acc[name] = {};
  }
  return acc;
}, {});

getNestedFields(names, getter) {
    const fields = names || this.getValidFieldsName();
    // 利用 reduce + lodash 的 set 方法，由数组返回了一个新的对象
    return fields.reduce((acc, f) => set(acc, f, getter(f)), {});
}

// 装饰器模式的另一个作用是创建一个属于自己的作用域，在作用域范围内可以访问一些变量
componentWillReceiveProps(nextProps) {
  if (mapPropsToFields) {
    this.fieldsStore.updateFields(mapPropsToFields(nextProps));
  }
}

// 支持 callback 和 Promise 的两种调用方法
const oldCb = callback;
callback = (errors, values) => {
  if (oldCb) {
    oldCb(errors, values);
  } else if (errors) {
    reject({ errors, values });
  } else {
    resolve(values);
  }
};

// PropTypes.shape 可以指定一个对象由特定的类型值组成
const formShape = PropTypes.shape({
  getFieldsValue: PropTypes.func,
  getFieldValue: PropTypes.func,
  getFieldInstance: PropTypes.func,
  setFieldsValue: PropTypes.func,
  setFields: PropTypes.func,
  setFieldsInitialValue: PropTypes.func,
  getFieldDecorator: PropTypes.func,
  getFieldProps: PropTypes.func,
  getFieldsError: PropTypes.func,
  getFieldError: PropTypes.func,
  isFieldValidating: PropTypes.func,
  isFieldsValidating: PropTypes.func,
  isFieldsTouched: PropTypes.func,
  isFieldTouched: PropTypes.func,
  isSubmitting: PropTypes.func,
  submit: PropTypes.func,
  validateFields: PropTypes.func,
  resetFields: PropTypes.func,
});

// 获取指定样式，这个样式是渲染之后的样式
function computedStyle(el, prop) {
  const getComputedStyle = window.getComputedStyle;
  const style =
    // If we have getComputedStyle
    getComputedStyle ?
      // Query it
      // TODO: From CSS-Query notes, we might need (node, null) for FF
      getComputedStyle(el) :

      // Otherwise, we are in IE and use currentStyle
      el.currentStyle;
  if (style) {
    return style
      [
      // Switch to camelCase for CSSOM
      // DEV: Grabbed from jQuery
      // https://github.com/jquery/jquery/blob/1.9-stable/src/css.js#L191-L194
      // https://github.com/jquery/jquery/blob/1.9-stable/src/core.js#L593-L597
        prop.replace(/-(\w)/gi, (word, letter) => {
          return letter.toUpperCase();
        })
      ];
  }
  return undefined;
}

// 父子关系可以使用 while 实现递归查询
while ((nodeName = node.nodeName.toLowerCase()) !== 'body') {
  const overflowY = computedStyle(node, 'overflowY');
  // https://stackoverflow.com/a/36900407/3040605
  if (
    node !== n &&
      (overflowY === 'auto' || overflowY === 'scroll') &&
      node.scrollHeight > node.clientHeight
  ) {
    return node;
  }
  node = node.parentNode;
}

const node = ReactDOM.findDOMNode(instance);
// Element.getBoundingClientRect()方法返回元素的大小及其相对于视口的位置。
// rect 是一个具有四个属性left、top、right、bottom 的 DOMRect 对象
// rect 的值是相对视口的，而不是绝对的，所以可能会因为随着滚动位置的变化而出现负数
const top = node.getBoundingClientRect().top;
// 获取节点 DOM，通过比较 top 的大小来判断距离顶部最近的一个节点
if (node.type !== 'hidden' && (firstTop === undefined || firstTop > top)) {
  firstTop = top;
  firstNode = node;
}

// React.isValidElement(object) 验证对象是否为 React 元素，返回值为 true 或 false
if (React.isValidElement(e)) {
  node = e;
} else if (React.isValidElement(e.message)) {
  node = e.message;
}

// 这里没有 children ，是如何把子节点显示出来的？
// 原生 DOM 就有 children 属性，children 属性就是子节点
return <form {...formProps} className={formClassName} />;

// keyCode 13 是回车键
if (e.keyCode === 13 && onPressEnter) {
  onPressEnter(e);
}

// 当组件非完全受控组件时，取传入的 value 值
if (!('value' in this.props)) {
  this.setState({ value }, callback);
}
// 组件内部处理了值以后，再通知给外部
// 通过 getDerivedStateFromProps 又可以让组件变成完全受控组件
const { onChange } = this.props;
if (onChange) {
  // ...
}

// 很多地方都调用一个显式方法来获取 props 的值
export function getPropValue(child: Option, prop?: any) {
  if (prop === 'value') {
    return getValuePropValue(child);
  }
  return child.props[prop];
}

// 使用方法来做判断，与 vue 同出一辙，这应该是一种比较流行的编程思想
export function isMultiple(props: Partial<ISelectProps>) {
  return props.multiple;
}

// 生成 uuid 的方法
export function generateUUID(): string {
  if (process.env.NODE_ENV === 'test') {
    return 'test-uuid';
  }
  let d = new Date().getTime();
  const uuid = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
    // tslint:disable-next-line:no-bitwise
    const r = (d + Math.random() * 16) % 16 | 0;
    d = Math.floor(d / 16);
    // tslint:disable-next-line:no-bitwise
    return (c === 'x' ? r : (r & 0x7) | 0x8).toString(16);
  });
  return uuid;
}

// 使用一个 noop 函数作为函数类 props 的默认值，可以在其他地方减少一些判断的逻辑
const noop = () => null;

class Select extends React.Component<Partial<ISelectProps>, ISelectState> {
  // ...
  
  // 在类中可以挂载一些静态方法，用于在类的内部使用（示意与类关联性较高）
  public static getOptionsFromChildren = (
    children: Array<React.ReactElement<any>>,
    options: any[] = [],
  ) => {
    React.Children.forEach(children, child => {
      if (!child) {
        return;
      }
      const type = (child as React.ReactElement<any>).type as any;
      if (type.isSelectOptGroup) {
        Select.getOptionsFromChildren((child as React.ReactElement<any>).props.children, options);
      } else {
        options.push(child);
      }
    });
    return options;
  };

  // ...

  {/* React.cloneElement 被大量使用，应该显式使用该函数，而不是 <Component {...oldProps} {...newProps} /> */}
  {React.cloneElement(inputElement, {
    ref: this.saveInputRef,
    onChange: this.onInputChange,
    onKeyDown: chaining(
      this.onInputKeyDown,
      inputElement.props.onKeyDown,
      this.props.onInputKeyDown,
    ),
    value: this.state.inputValue,
    disabled: props.disabled,
    className: inputCls,
  })}
}

// 获取浏览器支持的 style 属性
window.document.documentElement.style

// createEvent 创建一个指定类型的事件。其返回的对象必须先初始化并可以被传递给 element.dispatchEvent
// 创建事件
var event = document.createEvent('Event');

// 定义事件名为'build'.
event.initEvent('build', true, true);

// 监听事件
elem.addEventListener('build', function (e) {
  // e.target matches elem
}, false);

// 触发对象可以是任何元素或其他事件目标
elem.dispatchEvent(event);

// 获取元素的上滚距离
getCurrentScrollTop = () => {
  const getTarget = this.props.target || getDefaultTarget;
  const targetNode = getTarget();
  if (targetNode === window) {
    return window.pageYOffset || document.body.scrollTop || document.documentElement!.scrollTop;
  }
  return (targetNode as HTMLElement).scrollTop;
};
```
