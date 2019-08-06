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
      // 合并为一个字符串
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

// Insert one space between two chinese characters automatically.
function insertSpace(child: React.ReactChild, needInserted: boolean) {
  // Check the child if is undefined or null.
  if (child == null) {
    return;
  }
  const SPACE = needInserted ? ' ' : '';
  // strictNullChecks oops.
  if (
    typeof child !== 'string' &&
    typeof child !== 'number' &&
    isString(child.type) &&
    isTwoCNChar(child.props.children)
  ) {
    // 返回一个用空格隔开的 children 字符串的节点
    return React.cloneElement(child, {}, child.props.children.split('').join(SPACE));
  }
  if (typeof child === 'string') {
    if (isTwoCNChar(child)) {
      child = child.split('').join(SPACE);
    }
    return <span>{child}</span>;
  }
  return child;
}

// 使用 tuple 生成一个 ["default", "primary", "ghost", "dashed", "danger", "link"] 的数组作为枚举类型入参，约定入参为数组中的某项
const ButtonTypes = tuple('default', 'primary', 'ghost', 'dashed', 'danger', 'link');
export type ButtonType = (typeof ButtonTypes)[number];
// 约定了其他的一些属性
const ButtonShapes = tuple('circle', 'circle-outline', 'round');
export type ButtonShape = (typeof ButtonShapes)[number];
const ButtonSizes = tuple('large', 'default', 'small');
export type ButtonSize = (typeof ButtonSizes)[number];
const ButtonHTMLTypes = tuple('submit', 'button', 'reset');
export type ButtonHTMLType = (typeof ButtonHTMLTypes)[number];

// 约定了按钮的 props
export interface BaseButtonProps {
  type?: ButtonType;
  icon?: string;
  shape?: ButtonShape;
  size?: ButtonSize;
  loading?: boolean | { delay?: number };
  prefixCls?: string;
  className?: string;
  ghost?: boolean;
  block?: boolean;
  children?: React.ReactNode;
}

export type AnchorButtonProps = {
  href: string;
  target?: string;
  onClick?: React.MouseEventHandler<HTMLElement>;
} & BaseButtonProps &
  Omit<React.AnchorHTMLAttributes<any>, 'type' | 'onClick'>;

export type NativeButtonProps = {
  htmlType?: ButtonHTMLType;
  onClick?: React.MouseEventHandler<HTMLElement>;
} & BaseButtonProps &
  Omit<React.ButtonHTMLAttributes<any>, 'type' | 'onClick'>;

interface ButtonState {
  loading?: boolean | { delay?: number };
  hasTwoCNChar: boolean;
}

// 定义 Button 子类，传入 ButtonProps 作为 Button 的 props 类型，传入 ButtonState 作为 Button 的 state 类型
class Button extends React.Component<ButtonProps, ButtonState> {
  //...
  // 定义静态属性
  static Group: typeof Group;
  static __ANT_BUTTON = true;

  // 默认的 props
  static defaultProps = {
    loading: false,
    ghost: false,
    block: false,
    htmlType: 'button',
  };

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

  private delayTimeout: number;
  private buttonNode: HTMLElement | null;

  constructor(props: ButtonProps) {
    super(props);
    // 初始化的时候就读取 props 的 loading 状态
    this.state = {
      loading: props.loading,
      hasTwoCNChar: false,
    };
  }

  componentDidMount() {
    this.fixTwoCNChar();
  }

  componentDidUpdate(prevProps: ButtonProps) {
    this.fixTwoCNChar();

    // 在更新 loading 属性且之前的状态是非 Boolean 值时，清除延时定时器
    if (prevProps.loading && typeof prevProps.loading !== 'boolean') {
      clearTimeout(this.delayTimeout);
    }

    const { loading } = this.props;
    // loading 是一个对象的话，设置一个延时定时器在指定时间后进入 loading 状态
    if (loading && typeof loading !== 'boolean' && loading.delay) {
      this.delayTimeout = window.setTimeout(() => this.setState({ loading }), loading.delay);
    } else if (prevProps.loading === this.props.loading) {
      return;
    } else {
      this.setState({ loading });
    }
  }

  componentWillUnmount() {
    if (this.delayTimeout) {
      clearTimeout(this.delayTimeout);
    }
  }

  // 缓存一份 Button Ref
  saveButtonRef = (node: HTMLElement | null) => {
    this.buttonNode = node;
  };

  fixTwoCNChar() {
    // Fix for HOC usage like <FormatMessage />
    if (!this.buttonNode) {
      return;
    }
    // 获取 Button 中的 text
    const buttonText = this.buttonNode.textContent || this.buttonNode.innerText;
    // 判断是否需要插入和是否为两个中文字符
    if (this.isNeedInserted() && isTwoCNChar(buttonText)) {
      if (!this.state.hasTwoCNChar) {
        this.setState({
          hasTwoCNChar: true,
        });
      }
    } else if (this.state.hasTwoCNChar) {
      this.setState({
        hasTwoCNChar: false,
      });
    }
  }

  handleClick: React.MouseEventHandler<HTMLButtonElement | HTMLAnchorElement> = e => {
    const { loading } = this.state;
    const { onClick } = this.props;
    // 如果处于 Loading 状态则静默处理鼠标点击事件
    if (!!loading) {
      return;
    }
    // 如果 onClick 属性存在，则执行该方法并将 e 作为参数传入
    if (onClick) {
      (onClick as React.MouseEventHandler<HTMLButtonElement | HTMLAnchorElement>)(e);
    }
  };

  isNeedInserted() {
    const { icon, children } = this.props;
    // React.Children.count 返回 children 中的组件总数量，等同于通过 map 或 forEach 调用回调函数的次数
    // 子组件数量等于1，并且没有 icon 时，需要插入空格
    return React.Children.count(children) === 1 && !icon;
  }

  renderButton = ({ getPrefixCls, autoInsertSpaceInButton }: ConfigConsumerProps) => {
    // 解构 props 属性
    const {
      prefixCls: customizePrefixCls,
      type,
      shape,
      size,
      className,
      children,
      icon,
      ghost,
      loading: _loadingProp,
      block,
      ...rest
    } = this.props;
    const { loading, hasTwoCNChar } = this.state;

    // 组装组件的前缀 customizePrefixCls 的值应该是 ant，组装后的值应该是 ant-btn
    const prefixCls = getPrefixCls('btn', customizePrefixCls);
    // 是否自动插入空格
    const autoInsertSpace = autoInsertSpaceInButton !== false;

    // large => lg
    // small => sm
    let sizeCls = '';
    switch (size) {
      case 'large':
        sizeCls = 'lg';
        break;
      case 'small':
        sizeCls = 'sm';
        break;
      default:
        break;
    }

    // 组装 classList 首先是 ant-btn，其次是 props 传入的 className，最后是由 props 定义的一些样式
    const classes = classNames(prefixCls, className, {
      // 按钮类型，如 ant-btn-primary
      [`${prefixCls}-${type}`]: type,
      [`${prefixCls}-${shape}`]: shape,
      // 按钮的尺寸，如 ant-btn-sm
      [`${prefixCls}-${sizeCls}`]: sizeCls,
      // 按钮只包含 icon，组件没有 children 且 icon 属性存在
      [`${prefixCls}-icon-only`]: !children && children !== 0 && icon,
      [`${prefixCls}-loading`]: loading,
      [`${prefixCls}-background-ghost`]: ghost,
      // 是否为两个中文字符，autoInsertSpace 是全局配置参数
      [`${prefixCls}-two-chinese-chars`]: hasTwoCNChar && autoInsertSpace,
      [`${prefixCls}-block`]: block,
    });

    // 组件为 loading 状态的话，创建一个 iconNode
    const iconType = loading ? 'loading' : icon;
    const iconNode = iconType ? <Icon type={iconType} /> : null;
    // 判断存在 children 且只有一个 child 节点，并且满足其他条件的话，在两个中文字符间插入空格
    const kids =
      children || children === 0
        ? spaceChildren(children, this.isNeedInserted() && autoInsertSpace)
        : null;

    // 取出 rest 中除了 htmlType 以外的所有属性
    const linkButtonRestProps = omit(rest as AnchorButtonProps, ['htmlType']);
    // 如果 linkButtonRestProps 中包含 href 属性，则返回一个由 a 标签包裹的元素
    if (linkButtonRestProps.href !== undefined) {
      return (
        <a
          {...linkButtonRestProps}
          className={classes}
          onClick={this.handleClick}
          ref={this.saveButtonRef}
        >
          {iconNode}
          {kids}
        </a>
      );
    }

    // React does not recognize the `htmlType` prop on a DOM element. Here we pick it out of `rest`.
    const { htmlType, ...otherProps } = rest as NativeButtonProps;

    const buttonNode = (
      <button
        {...(otherProps as NativeButtonProps)}
        type={htmlType}
        className={classes}
        onClick={this.handleClick}
        ref={this.saveButtonRef}
      >
        {iconNode}
        {kids}
      </button>
    );

    // 如果类型为 link 则返回 buttonNode
    if (type === 'link') {
      return buttonNode;
    }

    // 返回由 Wave 组件包裹的 Button 组件
    return <Wave>{buttonNode}</Wave>;
  };

  render() {
    return <ConfigConsumer>{this.renderButton}</ConfigConsumer>;
  }
  //...
}
// ...
```