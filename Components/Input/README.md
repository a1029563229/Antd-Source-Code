# Input

## index.tsx
```tsx
import Input from './Input';
import Group from './Group';
import Search from './Search';
import TextArea from './TextArea';
import Password from './Password';

export { InputProps } from './Input';
export { GroupProps } from './Group';
export { SearchProps } from './Search';
export { TextAreaProps } from './TextArea';
export { PasswordProps } from './Password';

Input.Group = Group;
Input.Search = Search;
Input.TextArea = TextArea;
Input.Password = Password;
export default Input;
```

## Input.tsx
```tsx
import * as React from 'react';
import * as PropTypes from 'prop-types';
import classNames from 'classnames';
import omit from 'omit.js';
import { polyfill } from 'react-lifecycles-compat';
import Group from './Group';
import Search from './Search';
import TextArea from './TextArea';
import { ConfigConsumer, ConfigConsumerProps } from '../config-provider';
import Password from './Password';
import Icon from '../icon';
import { Omit, tuple } from '../_util/type';
import warning from '../_util/warning';

// value 为 undefined 或 null 时，返回一个空值
function fixControlledValue<T>(value: T) {
  if (typeof value === 'undefined' || value === null) {
    return '';
  }
  return value;
}

function hasPrefixSuffix(props: InputProps) {
  return !!('prefix' in props || props.suffix || props.allowClear);
}

const InputSizes = tuple('small', 'default', 'large');

export interface InputProps
  extends Omit<React.InputHTMLAttributes<HTMLInputElement>, 'size' | 'prefix'> {
  prefixCls?: string;
  size?: (typeof InputSizes)[number];
  onPressEnter?: React.KeyboardEventHandler<HTMLInputElement>;
  addonBefore?: React.ReactNode;
  addonAfter?: React.ReactNode;
  prefix?: React.ReactNode;
  suffix?: React.ReactNode;
  allowClear?: boolean;
}

class Input extends React.Component<InputProps, any> {
  static Group: typeof Group;
  static Search: typeof Search;
  static TextArea: typeof TextArea;
  static Password: typeof Password;

  static defaultProps = {
    type: 'text',
  };

  static propTypes = {
    type: PropTypes.string,
    id: PropTypes.string,
    size: PropTypes.oneOf(InputSizes),
    maxLength: PropTypes.number,
    disabled: PropTypes.bool,
    value: PropTypes.any,
    defaultValue: PropTypes.any,
    className: PropTypes.string,
    addonBefore: PropTypes.node,
    addonAfter: PropTypes.node,
    prefixCls: PropTypes.string,
    onPressEnter: PropTypes.func,
    // 比文档多几个支持的事件监听
    onKeyDown: PropTypes.func,
    onKeyUp: PropTypes.func,
    onFocus: PropTypes.func,
    onBlur: PropTypes.func,
    prefix: PropTypes.node,
    suffix: PropTypes.node,
    allowClear: PropTypes.bool,
  };

  // 如果传入了 value，则一直使用传入的 value，这就是传入 value 以后，value 不变，值就不会再变化的原因
  static getDerivedStateFromProps(nextProps: InputProps) {
    if ('value' in nextProps) {
      return {
        value: nextProps.value,
      };
    }
    return null;
  }

  input: HTMLInputElement;

  constructor(props: InputProps) {
    super(props);
    const value = typeof props.value === 'undefined' ? props.defaultValue : props.value;
    this.state = {
      value,
    };
  }

  getSnapshotBeforeUpdate(prevProps: InputProps) {
    if (hasPrefixSuffix(prevProps) !== hasPrefixSuffix(this.props)) {
      warning(
        this.input !== document.activeElement,
        'Input',
        `When Input is focused, dynamic add or remove prefix / suffix will make it lose focus caused by dom structure change. Read more: https://ant.design/components/input/#FAQ`,
      );
    }
    return null;
  }

  // Since polyfill `getSnapshotBeforeUpdate` need work with `componentDidUpdate`.
  // We keep an empty function here.
  componentDidUpdate() {}

  handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    const { onPressEnter, onKeyDown } = this.props;
    // keyCode 13 是回车键
    if (e.keyCode === 13 && onPressEnter) {
      onPressEnter(e);
    }
    if (onKeyDown) {
      onKeyDown(e);
    }
  };

  // 组件自带 focus 和 blur 方法可供使用
  focus() {
    this.input.focus();
  }

  blur() {
    this.input.blur();
  }

  select() {
    this.input.select();
  }

  getInputClassName(prefixCls: string) {
    const { size, disabled } = this.props;
    return classNames(prefixCls, {
      [`${prefixCls}-sm`]: size === 'small',
      [`${prefixCls}-lg`]: size === 'large',
      [`${prefixCls}-disabled`]: disabled,
    });
  }

  saveInput = (node: HTMLInputElement) => {
    this.input = node;
  };

  setValue(
    value: string,
    e: React.ChangeEvent<HTMLInputElement> | React.MouseEvent<HTMLElement, MouseEvent>,
    callback?: () => void,
  ) {
    // 当组件非完全受控组件时，取传入的 value 值
    if (!('value' in this.props)) {
      this.setState({ value }, callback);
    }
    // 组件内部处理了值以后，再通知给外部
    // 通过 getDerivedStateFromProps 又可以让组件变成完全受控组件
    const { onChange } = this.props;
    if (onChange) {
      let event = e;
      // 做了个 click 事件的判断，很奇怪
      if (e.type === 'click') {
        // click clear icon
        event = Object.create(e);
        event.target = this.input;
        event.currentTarget = this.input;
        const originalInputValue = this.input.value;
        // change input value cause e.target.value should be '' when clear input
        this.input.value = '';
        onChange(event as React.ChangeEvent<HTMLInputElement>);
        // reset input value
        this.input.value = originalInputValue;
        return;
      }
      onChange(event as React.ChangeEvent<HTMLInputElement>);
    }
  }

  // 重置组件的值，并且获取焦点
  handleReset = (e: React.MouseEvent<HTMLElement, MouseEvent>) => {
    this.setValue('', e, () => {
      this.focus();
    });
  };

  handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    this.setValue(e.target.value, e);
  };

  // 清除按钮，功能是清除 value
  renderClearIcon(prefixCls: string) {
    const { allowClear } = this.props;
    const { value } = this.state;
    if (!allowClear || value === undefined || value === null || value === '') {
      return null;
    }
    return (
      <Icon
        type="close-circle"
        theme="filled"
        onClick={this.handleReset}
        className={`${prefixCls}-clear-icon`}
        role="button"
      />
    );
  }

  // 后缀
  renderSuffix(prefixCls: string) {
    const { suffix, allowClear } = this.props;
    if (suffix || allowClear) {
      return (
        <span className={`${prefixCls}-suffix`}>
          {this.renderClearIcon(prefixCls)}
          {suffix}
        </span>
      );
    }
    return null;
  }

  // 前后标签的 Input
  renderLabeledInput(prefixCls: string, children: React.ReactElement<any>) {
    const { addonBefore, addonAfter, style, size, className } = this.props;
    // Not wrap when there is not addons
    if (!addonBefore && !addonAfter) {
      return children;
    }

    const wrapperClassName = `${prefixCls}-group`;
    const addonClassName = `${wrapperClassName}-addon`;
    const addonBeforeNode = addonBefore ? (
      <span className={addonClassName}>{addonBefore}</span>
    ) : null;
    const addonAfterNode = addonAfter ? <span className={addonClassName}>{addonAfter}</span> : null;

    const mergedWrapperClassName = classNames(`${prefixCls}-wrapper`, {
      [wrapperClassName]: addonBefore || addonAfter,
    });

    const mergedGroupClassName = classNames(className, `${prefixCls}-group-wrapper`, {
      [`${prefixCls}-group-wrapper-sm`]: size === 'small',
      [`${prefixCls}-group-wrapper-lg`]: size === 'large',
    });

    // Need another wrapper for changing display:table to display:inline-block
    // and put style prop in wrapper
    return (
      <span className={mergedGroupClassName} style={style}>
        <span className={mergedWrapperClassName}>
          {addonBeforeNode}
          {/* 如果使用了前后标签，则 input 上的样式会被清除 */}
          {React.cloneElement(children, { style: null })}
          {addonAfterNode}
        </span>
      </span>
    );
  }

  renderLabeledIcon(prefixCls: string, children: React.ReactElement<any>) {
    const { props } = this;
    const suffix = this.renderSuffix(prefixCls);

    if (!hasPrefixSuffix(props)) {
      return children;
    }

    const prefix = props.prefix ? (
      <span className={`${prefixCls}-prefix`}>{props.prefix}</span>
    ) : null;

    const affixWrapperCls = classNames(props.className, `${prefixCls}-affix-wrapper`, {
      [`${prefixCls}-affix-wrapper-sm`]: props.size === 'small',
      [`${prefixCls}-affix-wrapper-lg`]: props.size === 'large',
      [`${prefixCls}-affix-wrapper-with-clear-btn`]:
        props.suffix && props.allowClear && this.state.value,
    });
    return (
      <span className={affixWrapperCls} style={props.style}>
        {prefix}
        {React.cloneElement(children, {
          style: null,
          className: this.getInputClassName(prefixCls),
        })}
        {suffix}
      </span>
    );
  }

  renderInput(prefixCls: string) {
    const { className, addonBefore, addonAfter } = this.props;
    const { value } = this.state;
    // Fix https://fb.me/react-unknown-prop
    const otherProps = omit(this.props, [
      'prefixCls',
      'onPressEnter',
      'addonBefore',
      'addonAfter',
      'prefix',
      'suffix',
      'allowClear',
      // Input elements must be either controlled or uncontrolled,
      // specify either the value prop, or the defaultValue prop, but not both.
      'defaultValue',
    ]);

    return this.renderLabeledIcon(
      prefixCls,
      <input
        {...otherProps}
        value={fixControlledValue(value)}
        onChange={this.handleChange}
        className={classNames(this.getInputClassName(prefixCls), {
          [className!]: className && !addonBefore && !addonAfter,
        })}
        onKeyDown={this.handleKeyDown}
        ref={this.saveInput}
      />,
    );
  }

  renderComponent = ({ getPrefixCls }: ConfigConsumerProps) => {
    const { prefixCls: customizePrefixCls } = this.props;
    const prefixCls = getPrefixCls('input', customizePrefixCls);
    return this.renderLabeledInput(prefixCls, this.renderInput(prefixCls));
  };

  render() {
    return <ConfigConsumer>{this.renderComponent}</ConfigConsumer>;
  }
}

polyfill(Input);

export default Input;
```

## calculateNodeHeight.tsx
```tsx
// Thanks to https://github.com/andreypopp/react-textarea-autosize/

/**
 * calculateNodeHeight(uiTextNode, useCache = false)
 */

const HIDDEN_TEXTAREA_STYLE = `
  min-height:0 !important;
  max-height:none !important;
  height:0 !important;
  visibility:hidden !important;
  overflow:hidden !important;
  position:absolute !important;
  z-index:-1000 !important;
  top:0 !important;
  right:0 !important
`;

const SIZING_STYLE = [
  'letter-spacing',
  'line-height',
  'padding-top',
  'padding-bottom',
  'font-family',
  'font-weight',
  'font-size',
  'font-variant',
  'text-rendering',
  'text-transform',
  'width',
  'text-indent',
  'padding-left',
  'padding-right',
  'border-width',
  'box-sizing',
];

export interface NodeType {
  sizingStyle: string;
  paddingSize: number;
  borderSize: number;
  boxSizing: string;
}

const computedStyleCache: { [key: string]: NodeType } = {};
let hiddenTextarea: HTMLTextAreaElement;

export function calculateNodeStyling(node: HTMLElement, useCache = false) {
  const nodeRef = (node.getAttribute('id') ||
    node.getAttribute('data-reactid') ||
    node.getAttribute('name')) as string;

  if (useCache && computedStyleCache[nodeRef]) {
    return computedStyleCache[nodeRef];
  }

  const style = window.getComputedStyle(node);

  const boxSizing =
    style.getPropertyValue('box-sizing') ||
    style.getPropertyValue('-moz-box-sizing') ||
    style.getPropertyValue('-webkit-box-sizing');

  const paddingSize =
    parseFloat(style.getPropertyValue('padding-bottom')) +
    parseFloat(style.getPropertyValue('padding-top'));

  const borderSize =
    parseFloat(style.getPropertyValue('border-bottom-width')) +
    parseFloat(style.getPropertyValue('border-top-width'));

  const sizingStyle = SIZING_STYLE.map(name => `${name}:${style.getPropertyValue(name)}`).join(';');

  const nodeInfo: NodeType = {
    sizingStyle,
    paddingSize,
    borderSize,
    boxSizing,
  };

  if (useCache && nodeRef) {
    computedStyleCache[nodeRef] = nodeInfo;
  }

  return nodeInfo;
}

// 计算节点的高度
export default function calculateNodeHeight(
  uiTextNode: HTMLTextAreaElement,
  useCache = false,
  minRows: number | null = null,
  maxRows: number | null = null,
) {
  if (!hiddenTextarea) {
    hiddenTextarea = document.createElement('textarea');
    document.body.appendChild(hiddenTextarea);
  }

  // Fix wrap="off" issue
  // https://github.com/ant-design/ant-design/issues/6577
  if (uiTextNode.getAttribute('wrap')) {
    hiddenTextarea.setAttribute('wrap', uiTextNode.getAttribute('wrap') as string);
  } else {
    hiddenTextarea.removeAttribute('wrap');
  }

  // Copy all CSS properties that have an impact on the height of the content in
  // the textbox
  const { paddingSize, borderSize, boxSizing, sizingStyle } = calculateNodeStyling(
    uiTextNode,
    useCache,
  );

  // Need to have the overflow attribute to hide the scrollbar otherwise
  // text-lines will not calculated properly as the shadow will technically be
  // narrower for content
  hiddenTextarea.setAttribute('style', `${sizingStyle};${HIDDEN_TEXTAREA_STYLE}`);
  hiddenTextarea.value = uiTextNode.value || uiTextNode.placeholder || '';

  let minHeight = Number.MIN_SAFE_INTEGER;
  let maxHeight = Number.MAX_SAFE_INTEGER;
  // 精髓就是这一个 scrollHeight
  // scrollHeight 是这个内容包括看不见、溢出、被窗口遮挡的部分；
  let height = hiddenTextarea.scrollHeight;
  let overflowY: any;

  if (boxSizing === 'border-box') {
    // border-box: add border, since height = content + padding + border
    height = height + borderSize;
  } else if (boxSizing === 'content-box') {
    // remove padding, since height = content
    height = height - paddingSize;
  }

  if (minRows !== null || maxRows !== null) {
    // measure height of a textarea with a single row
    hiddenTextarea.value = ' ';
    const singleRowHeight = hiddenTextarea.scrollHeight - paddingSize;
    if (minRows !== null) {
      minHeight = singleRowHeight * minRows;
      if (boxSizing === 'border-box') {
        minHeight = minHeight + paddingSize + borderSize;
      }
      height = Math.max(minHeight, height);
    }
    if (maxRows !== null) {
      maxHeight = singleRowHeight * maxRows;
      if (boxSizing === 'border-box') {
        maxHeight = maxHeight + paddingSize + borderSize;
      }
      overflowY = height > maxHeight ? '' : 'hidden';
      height = Math.min(maxHeight, height);
    }
  }
  return { height, minHeight, maxHeight, overflowY };
}
```

## Input.Search
```tsx
// 内部使用 Input + addonAfter + ... 拼装得到一个内嵌了搜索功能的组件
```