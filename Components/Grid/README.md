# Grid

## row.tsx
```tsx
import { ConfigConsumer, ConfigConsumerProps } from '../config-provider';
import * as React from 'react';
import classNames from 'classnames';
import * as PropTypes from 'prop-types';
import RowContext from './RowContext';
import { tuple } from '../_util/type';
import ResponsiveObserve, {
  Breakpoint,
  BreakpointMap,
  responsiveArray,
} from '../_util/responsiveObserve';

const RowAligns = tuple('top', 'middle', 'bottom');
const RowJustify = tuple('start', 'end', 'center', 'space-around', 'space-between');
export interface RowProps extends React.HTMLAttributes<HTMLDivElement> {
  gutter?: number | Partial<Record<Breakpoint, number>>;
  type?: 'flex';
  align?: (typeof RowAligns)[number];
  justify?: (typeof RowJustify)[number];
  prefixCls?: string;
}

export interface RowState {
  screens: BreakpointMap;
}

export default class Row extends React.Component<RowProps, RowState> {
  static defaultProps = {
    gutter: 0,
  };

  static propTypes = {
    type: PropTypes.oneOf<'flex'>(['flex']),
    align: PropTypes.oneOf(RowAligns),
    justify: PropTypes.oneOf(RowJustify),
    className: PropTypes.string,
    children: PropTypes.node,
    gutter: PropTypes.oneOfType([PropTypes.object, PropTypes.number]),
    prefixCls: PropTypes.string,
  };

  state: RowState = {
    screens: {},
  };
  token: string;
  componentDidMount() {
    this.token = ResponsiveObserve.subscribe(screens => {
      if (typeof this.props.gutter === 'object') {
        this.setState({ screens });
      }
    });
  }
  componentWillUnmount() {
    ResponsiveObserve.unsubscribe(this.token);
  }
  getGutter(): number | undefined {
    const { gutter } = this.props;
    if (typeof gutter === 'object') {
      for (let i = 0; i < responsiveArray.length; i++) {
        const breakpoint: Breakpoint = responsiveArray[i];
        // 如果 gutter 为响应式对象（例 { xs: 8, sm: 16, md: 24} ），返回对应尺寸的 gutter
        if (this.state.screens[breakpoint] && gutter[breakpoint] !== undefined) {
          return gutter[breakpoint];
        }
      }
    }
    return gutter as number;
  }
  renderRow = ({ getPrefixCls }: ConfigConsumerProps) => {
    const {
      prefixCls: customizePrefixCls,
      type,
      justify,
      align,
      className,
      style,
      children,
      ...others
    } = this.props;
    const prefixCls = getPrefixCls('row', customizePrefixCls);
    const gutter = this.getGutter();
    const classes = classNames(
      {
        [prefixCls]: !type,
        [`${prefixCls}-${type}`]: type,
        [`${prefixCls}-${type}-${justify}`]: type && justify,
        [`${prefixCls}-${type}-${align}`]: type && align,
      },
      className,
    );
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
```

## col.tsx
```tsx
import * as React from 'react';
import * as PropTypes from 'prop-types';
import classNames from 'classnames';
import RowContext from './RowContext';
import { ConfigConsumer, ConfigConsumerProps } from '../config-provider';

const objectOrNumber = PropTypes.oneOfType([PropTypes.object, PropTypes.number]);

// https://github.com/ant-design/ant-design/issues/14324
type ColSpanType = number | string;

export interface ColSize {
  span?: ColSpanType;
  order?: ColSpanType;
  offset?: ColSpanType;
  push?: ColSpanType;
  pull?: ColSpanType;
}

export interface ColProps extends React.HTMLAttributes<HTMLDivElement> {
  span?: ColSpanType;
  order?: ColSpanType;
  offset?: ColSpanType;
  push?: ColSpanType;
  pull?: ColSpanType;
  xs?: ColSpanType | ColSize;
  sm?: ColSpanType | ColSize;
  md?: ColSpanType | ColSize;
  lg?: ColSpanType | ColSize;
  xl?: ColSpanType | ColSize;
  xxl?: ColSpanType | ColSize;
  prefixCls?: string;
}

export default class Col extends React.Component<ColProps, {}> {
  static propTypes = {
    span: PropTypes.number,
    order: PropTypes.number,
    offset: PropTypes.number,
    push: PropTypes.number,
    pull: PropTypes.number,
    className: PropTypes.string,
    children: PropTypes.node,
    xs: objectOrNumber,
    sm: objectOrNumber,
    md: objectOrNumber,
    lg: objectOrNumber,
    xl: objectOrNumber,
    xxl: objectOrNumber,
  };

  renderCol = ({ getPrefixCls }: ConfigConsumerProps) => {
    const props: any = this.props;
    const {
      prefixCls: customizePrefixCls,
      span,
      order,
      offset,
      push,
      pull,
      className,
      children,
      ...others
    } = props;
    const prefixCls = getPrefixCls('col', customizePrefixCls);
    let sizeClassObj = {};
    ['xs', 'sm', 'md', 'lg', 'xl', 'xxl'].forEach(size => {
      let sizeProps: ColSize = {};
      if (typeof props[size] === 'number') {
        sizeProps.span = props[size];
        // Col 组件同样支持响应式布局
      } else if (typeof props[size] === 'object') {
        sizeProps = props[size] || {};
      }

      delete others[size];

      sizeClassObj = {
        ...sizeClassObj,
        [`${prefixCls}-${size}-${sizeProps.span}`]: sizeProps.span !== undefined,
        [`${prefixCls}-${size}-order-${sizeProps.order}`]: sizeProps.order || sizeProps.order === 0,
        [`${prefixCls}-${size}-offset-${sizeProps.offset}`]:
          sizeProps.offset || sizeProps.offset === 0,
        [`${prefixCls}-${size}-push-${sizeProps.push}`]: sizeProps.push || sizeProps.push === 0,
        [`${prefixCls}-${size}-pull-${sizeProps.pull}`]: sizeProps.pull || sizeProps.pull === 0,
      };
    });
    const classes = classNames(
      prefixCls,
      {
        [`${prefixCls}-${span}`]: span !== undefined,
        [`${prefixCls}-order-${order}`]: order,
        [`${prefixCls}-offset-${offset}`]: offset,
        [`${prefixCls}-push-${push}`]: push,
        [`${prefixCls}-pull-${pull}`]: pull,
      },
      className,
      sizeClassObj,
    );

    // 利用 RowContext.Consumer 提供的 gutter 属性对组件的 style 进行赋值
    return (
      <RowContext.Consumer>
        {({ gutter }) => {
          let style = others.style;
          if (gutter! > 0) {
            style = {
              paddingLeft: gutter! / 2,
              paddingRight: gutter! / 2,
              ...style,
            };
          }
          return (
            <div {...others} style={style} className={classes}>
              {children}
            </div>
          );
        }}
      </RowContext.Consumer>
    );
  };

  render() {
    return <ConfigConsumer>{this.renderCol}</ConfigConsumer>;
  }
}
```