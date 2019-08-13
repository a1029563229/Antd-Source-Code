# Form

## constants.ts
```ts
// 定义两个常量
export const FIELD_META_PROP = 'data-__meta';
export const FIELD_DATA_PROP = 'data-__field';
```

## context.ts
```tsx
import createReactContext from '@ant-design/create-react-context';
import { ColProps } from '../grid/col';
import { FormLabelAlign } from './FormItem';

export interface FormContextProps {
  vertical: boolean;
  colon?: boolean;
  labelAlign?: FormLabelAlign;
  labelCol?: ColProps;
  wrapperCol?: ColProps;
}

// 创建了一个 Context，设置了一些默认参数
export const FormContext = createReactContext<FormContextProps>({
  labelAlign: 'right',
  vertical: false,
});
```

## FormItem.tsx
```tsx
import * as React from 'react';
import * as ReactDOM from 'react-dom';
import * as PropTypes from 'prop-types';
import classNames from 'classnames';
// react 的动画组件，通过一些设置可以更方便的执行动画
import Animate from 'rc-animate';
import Row from '../grid/row';
import Col, { ColProps } from '../grid/col';
import Icon from '../icon';
import { ConfigConsumer, ConfigConsumerProps } from '../config-provider';
// warning (valid: boolean, component: string, message: string) 当 valid 为 false 时，控制台输出警告；条件成立则控制台不会出现警告。
import warning from '../_util/warning';
import { tuple } from '../_util/type';
import { FIELD_META_PROP, FIELD_DATA_PROP } from './constants';
import { FormContext, FormContextProps } from './context';

const ValidateStatuses = tuple('success', 'warning', 'error', 'validating', '');

export type FormLabelAlign = 'left' | 'right';

export interface FormItemProps {
  prefixCls?: string;
  className?: string;
  id?: string;
  htmlFor?: string;
  label?: React.ReactNode;
  labelAlign?: FormLabelAlign;
  labelCol?: ColProps;
  wrapperCol?: ColProps;
  help?: React.ReactNode;
  extra?: React.ReactNode;
  validateStatus?: (typeof ValidateStatuses)[number];
  hasFeedback?: boolean;
  required?: boolean;
  style?: React.CSSProperties;
  colon?: boolean;
}

// 每个数组元素之间插入一个空格
function intersperseSpace<T>(list: Array<T>): Array<T | string> {
  return list.reduce((current, item) => [...current, ' ', item], []).slice(1);
}

// props 类型为 FormItemProps
// state 类型为 any
export default class FormItem extends React.Component<FormItemProps, any> {
  static defaultProps = {
    hasFeedback: false,
  };

  // 静态属性 propTypes 用于定义 props 的类型
  static propTypes = {
    prefixCls: PropTypes.string,
    label: PropTypes.oneOfType([PropTypes.string, PropTypes.node]),
    labelAlign: PropTypes.string,
    labelCol: PropTypes.object,
    help: PropTypes.oneOfType([PropTypes.node, PropTypes.bool]),
    validateStatus: PropTypes.oneOf(ValidateStatuses),
    hasFeedback: PropTypes.bool,
    wrapperCol: PropTypes.object,
    className: PropTypes.string,
    id: PropTypes.string,
    children: PropTypes.node,
    colon: PropTypes.bool,
  };

  helpShow = false;

  componentDidMount() {
    const { children, help, validateStatus, id } = this.props;
    warning(
      this.getControls(children, true).length <= 1 ||
        (help !== undefined || validateStatus !== undefined),
      'Form.Item',
      'Cannot generate `validateStatus` and `help` automatically, ' +
        'while there are more than one `getFieldDecorator` in it.',
    );

    warning(
      !id,
      'Form.Item',
      '`id` is deprecated for its label `htmlFor`. Please use `htmlFor` directly.',
    );
  }

  getHelpMessage() {
    const { help } = this.props;
    if (help === undefined && this.getOnlyControl()) {
      const { errors } = this.getField();
      if (errors) {
        return intersperseSpace(
          errors.map((e: any, index: number) => {
            let node: React.ReactElement<any> | null = null;

            // React.isValidElement(object) 验证对象是否为 React 元素，返回值为 true 或 false
            if (React.isValidElement(e)) {
              node = e;
            } else if (React.isValidElement(e.message)) {
              node = e.message;
            }

            return node ? React.cloneElement(node, { key: index }) : e.message;
          }),
        );
      }
      return '';
    }
    return help;
  }

  // 递归查询子节点中包含几个由 getFiledDecorator 生成的节点
  getControls(children: React.ReactNode, recursively: boolean) {
    let controls: React.ReactElement<any>[] = [];
    const childrenArray = React.Children.toArray(children);
    for (let i = 0; i < childrenArray.length; i++) {
      if (!recursively && controls.length > 0) {
        break;
      }

      const child = childrenArray[i] as React.ReactElement<any>;
      if (
        child.type &&
        ((child.type as any) === FormItem || (child.type as any).displayName === 'FormItem')
      ) {
        continue;
      }
      if (!child.props) {
        continue;
      }
      // 如果子节点中包含 data-__field 属性，说明该子节点也是由 getFieldDecorator 生成的
      // 超过两个此类子节点则输出错误提示
      // 'Cannot generate `validateStatus` and `help` automatically, while there are more than one `getFieldDecorator` in it.'
      if (FIELD_META_PROP in child.props) {
        // And means FIELD_DATA_PROP in child.props, too.
        controls.push(child);
      } else if (child.props.children) {
        controls = controls.concat(this.getControls(child.props.children, recursively));
      }
    }
    return controls;
  }

  // 寻找第一个由 getFiledDecorator 生成的节点
  getOnlyControl() {
    const child = this.getControls(this.props.children, false)[0];
    return child !== undefined ? child : null;
  }

  getChildProp(prop: string) {
    const child = this.getOnlyControl() as React.ReactElement<any>;
    return child && child.props && child.props[prop];
  }

  getId() {
    return this.getChildProp('id');
  }

  getMeta() {
    return this.getChildProp(FIELD_META_PROP);
  }

  getField() {
    return this.getChildProp(FIELD_DATA_PROP);
  }

  onHelpAnimEnd = (_key: string, helpShow: boolean) => {
    this.helpShow = helpShow;
    if (!helpShow) {
      this.setState({});
    }
  };

  // help 就是错误提示，`show-help` 是动画名
  renderHelp(prefixCls: string) {
    const help = this.getHelpMessage();
    const children = help ? (
      <div className={`${prefixCls}-explain`} key="help">
        {help}
      </div>
    ) : null;
    if (children) {
      this.helpShow = !!children;
    }
    return (
      <Animate
        transitionName="show-help"
        component=""
        transitionAppear
        key="help"
        onEnd={this.onHelpAnimEnd}
      >
        {children}
      </Animate>
    );
  }

  renderExtra(prefixCls: string) {
    const { extra } = this.props;
    return extra ? <div className={`${prefixCls}-extra`}>{extra}</div> : null;
  }

  // 每当 onCollect 触发时，会触发 render, 也会重新获取验证状态
  getValidateStatus() {
    const onlyControl = this.getOnlyControl();
    if (!onlyControl) {
      return '';
    }
    const field = this.getField();
    // validating 是正在校验的状态，通常只有异步校验才会保持在此状态
    if (field.validating) {
      return 'validating';
    }
    if (field.errors) {
      return 'error';
    }
    const fieldValue = 'value' in field ? field.value : this.getMeta().initialValue;
    if (fieldValue !== undefined && fieldValue !== null && fieldValue !== '') {
      return 'success';
    }
    return '';
  }

  // 包含验证器的组件
  renderValidateWrapper(
    prefixCls: string,
    c1: React.ReactNode,
    c2: React.ReactNode,
    c3: React.ReactNode,
  ) {
    const { props } = this;
    const onlyControl = this.getOnlyControl;
    const validateStatus =
      props.validateStatus === undefined && onlyControl
        ? this.getValidateStatus()
        : props.validateStatus;

    let classes = `${prefixCls}-item-control`;
    // 不同的状态有不同的样式
    if (validateStatus) {
      classes = classNames(`${prefixCls}-item-control`, {
        'has-feedback': props.hasFeedback || validateStatus === 'validating',
        'has-success': validateStatus === 'success',
        'has-warning': validateStatus === 'warning',
        'has-error': validateStatus === 'error',
        'is-validating': validateStatus === 'validating',
      });
    }

    let iconType = '';
    switch (validateStatus) {
      case 'success':
        iconType = 'check-circle';
        break;
      case 'warning':
        iconType = 'exclamation-circle';
        break;
      case 'error':
        iconType = 'close-circle';
        break;
      case 'validating':
        iconType = 'loading';
        break;
      default:
        iconType = '';
        break;
    }

    // hasFeedback 配合 validateStatus 属性使用，展示校验状态图标，建议只配合 Input 组件使用
    const icon =
      props.hasFeedback && iconType ? (
        <span className={`${prefixCls}-item-children-icon`}>
          <Icon type={iconType} theme={iconType === 'loading' ? 'outlined' : 'filled'} />
        </span>
      ) : null;

    return (
      <div className={classes}>
        <span className={`${prefixCls}-item-children`}>
          {c1}
          {icon}
        </span>
        {c2}
        {c3}
      </div>
    );
  }

  renderWrapper(prefixCls: string, children: React.ReactNode) {
    return (
      // 注入 Consumer
      <FormContext.Consumer key="wrapper">
        {({ wrapperCol: contextWrapperCol, vertical }: FormContextProps) => {
          const { wrapperCol } = this.props;
          const mergedWrapperCol: ColProps =
            ('wrapperCol' in this.props ? wrapperCol : contextWrapperCol) || {};

          const className = classNames(
            `${prefixCls}-item-control-wrapper`,
            mergedWrapperCol.className,
          );

          // No pass FormContext since it's useless
          return (
            <FormContext.Provider value={{ vertical }}>
              <Col {...mergedWrapperCol} className={className}>
                {children}
              </Col>
            </FormContext.Provider>
          );
        }}
      </FormContext.Consumer>
    );
  }

  // 这里应该是用于 UI 展示
  isRequired() {
    const { required } = this.props;
    if (required !== undefined) {
      return required;
    }
    if (this.getOnlyControl()) {
      const meta = this.getMeta() || {};
      const validate = meta.validate || [];

      return validate
        .filter((item: any) => !!item.rules)
        .some((item: any) => {
          return item.rules.some((rule: any) => rule.required);
        });
    }
    return false;
  }

  // Resolve duplicated ids bug between different forms
  // https://github.com/ant-design/ant-design/issues/7351
  onLabelClick = () => {
    const id = this.props.id || this.getId();
    if (!id) {
      return;
    }

    // 点击 Label 也可获取焦点
    const formItemNode = ReactDOM.findDOMNode(this) as Element;
    const control = formItemNode.querySelector(`[id="${id}"]`) as HTMLElement;
    if (control && control.focus) {
      control.focus();
    }
  };

  renderLabel(prefixCls: string) {
    return (
      // 用 Consumer 共享状态来达到控制样式的效果
      <FormContext.Consumer key="label">
        {({
          vertical,
          labelAlign: contextLabelAlign,
          labelCol: contextLabelCol,
          colon: contextColon,
        }: FormContextProps) => {
          const { label, labelCol, labelAlign, colon, id, htmlFor } = this.props;
          const required = this.isRequired();

          const mergedLabelCol: ColProps =
            ('labelCol' in this.props ? labelCol : contextLabelCol) || {};

          const mergedLabelAlign: FormLabelAlign | undefined =
            'labelAlign' in this.props ? labelAlign : contextLabelAlign;

          const labelClsBasic = `${prefixCls}-item-label`;
          const labelColClassName = classNames(
            labelClsBasic,
            mergedLabelAlign === 'left' && `${labelClsBasic}-left`,
            mergedLabelCol.className,
          );

          let labelChildren = label;
          // Keep label is original where there should have no colon
          const computedColon = colon === true || (contextColon !== false && colon !== false);
          const haveColon = computedColon && !vertical;
          // Remove duplicated user input colon
          if (haveColon && typeof label === 'string' && (label as string).trim() !== '') {
            labelChildren = (label as string).replace(/[：:]\s*$/, '');
          }

          const labelClassName = classNames({
            [`${prefixCls}-item-required`]: required,
            [`${prefixCls}-item-no-colon`]: !computedColon,
          });

          return label ? (
            <Col {...mergedLabelCol} className={labelColClassName}>
              <label
                htmlFor={htmlFor || id || this.getId()}
                className={labelClassName}
                title={typeof label === 'string' ? label : ''}
                onClick={this.onLabelClick}
              >
                {labelChildren}
              </label>
            </Col>
          ) : null;
        }}
      </FormContext.Consumer>
    );
  }

  renderChildren(prefixCls: string) {
    const { children } = this.props;
    return [
      // 注入 Label
      this.renderLabel(prefixCls),
      // 注入 Col
      this.renderWrapper(
        prefixCls,
        // 注入验证状态
        this.renderValidateWrapper(
          prefixCls,
          children,
          this.renderHelp(prefixCls),
          this.renderExtra(prefixCls),
        ),
      ),
    ];
  }

  renderFormItem = ({ getPrefixCls }: ConfigConsumerProps) => {
    const { prefixCls: customizePrefixCls, style, className } = this.props;
    const prefixCls = getPrefixCls('form', customizePrefixCls);
    const children = this.renderChildren(prefixCls);
    const itemClassName = {
      [`${prefixCls}-item`]: true,
      [`${prefixCls}-item-with-help`]: this.helpShow,
      [`${className}`]: !!className,
    };

    return (
      // 注入 Row，与 Col 完成配合
      <Row className={classNames(itemClassName)} style={style} key="row">
        {children}
      </Row>
    );
  };

  render() {
    return <ConfigConsumer>{this.renderFormItem}</ConfigConsumer>;
  }
}
```

## Form.tsx
```tsx
import * as React from 'react';
import * as PropTypes from 'prop-types';
import classNames from 'classnames';

// rc-form：react 的高阶表单组件
import createDOMForm from 'rc-form/lib/createDOMForm';
import createFormField from 'rc-form/lib/createFormField';
import omit from 'omit.js';
import { ConfigConsumer, ConfigConsumerProps } from '../config-provider';
import { ColProps } from '../grid/col';
import { tuple } from '../_util/type';
import warning from '../_util/warning';
import FormItem, { FormLabelAlign } from './FormItem';
import { FIELD_META_PROP, FIELD_DATA_PROP } from './constants';
import { FormContext } from './context';
import { FormWrappedProps } from './interface';

type FormCreateOptionMessagesCallback = (...args: any[]) => string;

interface FormCreateOptionMessages {
  [messageId: string]: string | FormCreateOptionMessagesCallback | FormCreateOptionMessages;
}

export interface FormCreateOption<T> {
  onFieldsChange?: (props: T, fields: any, allFields: any) => void;
  onValuesChange?: (props: T, changedValues: any, allValues: any) => void;
  mapPropsToFields?: (props: T) => void;
  validateMessages?: FormCreateOptionMessages;
  withRef?: boolean;
  name?: string;
}

const FormLayouts = tuple('horizontal', 'inline', 'vertical');
export type FormLayout = (typeof FormLayouts)[number];

export interface FormProps extends React.FormHTMLAttributes<HTMLFormElement> {
  layout?: FormLayout;
  form?: WrappedFormUtils;
  onSubmit?: React.FormEventHandler<HTMLFormElement>;
  style?: React.CSSProperties;
  className?: string;
  prefixCls?: string;
  hideRequiredMark?: boolean;
  /**
   * @since 3.14.0
   */
  wrapperCol?: ColProps;
  labelCol?: ColProps;
  /**
   * @since 3.15.0
   */
  colon?: boolean;
  labelAlign?: FormLabelAlign;
}

export type ValidationRule = {
  /** validation error message */
  message?: React.ReactNode;
  /** built-in validation type, available options: https://github.com/yiminghe/async-validator#type */
  type?: string;
  /** indicates whether field is required */
  required?: boolean;
  /** treat required fields that only contain whitespace as errors */
  whitespace?: boolean;
  /** validate the exact length of a field */
  len?: number;
  /** validate the min length of a field */
  min?: number;
  /** validate the max length of a field */
  max?: number;
  /** validate the value from a list of possible values */
  enum?: string | string[];
  /** validate from a regular expression */
  pattern?: RegExp;
  /** transform a value before validation */
  transform?: (value: any) => any;
  /** custom validate function (Note: callback must be called) */
  validator?: (rule: any, value: any, callback: any, source?: any, options?: any) => any;
};

export type ValidateCallback<V> = (errors: any, values: V) => void;

export type GetFieldDecoratorOptions = {
  /** 子节点的值的属性，如 Checkbox 的是 'checked' */
  valuePropName?: string;
  /** 子节点的初始值，类型、可选值均由子节点决定 */
  initialValue?: any;
  /** 收集子节点的值的时机 */
  trigger?: string;
  /** 可以把 onChange 的参数转化为控件的值，例如 DatePicker 可设为：(date, dateString) => dateString */
  // 默认返回 e.target.value 或 e.target.checkbox
  getValueFromEvent?: (...args: any[]) => any;
  /** Get the component props according to field value. */
  getValueProps?: (value: any) => any;
  /** 校验子节点值的时机 */
  validateTrigger?: string | string[];
  /** 校验规则，参见 [async-validator](https://github.com/yiminghe/async-validator) */
  rules?: ValidationRule[];
  /** 是否和其他控件互斥，特别用于 Radio 单选控件 */
  exclusive?: boolean;
  /** Normalize value to form component */
  normalize?: (value: any, prevValue: any, allValues: any) => any;
  /** Whether stop validate on first rule of error for this field.  */
  validateFirst?: boolean;
  /** 是否一直保留子节点的信息 */
  preserve?: boolean;
};

/** dom-scroll-into-view 组件配置参数 */
export type DomScrollIntoViewConfig = {
  /** 是否和左边界对齐 */
  alignWithLeft?: boolean;
  /** 是否和上边界对齐  */
  alignWithTop?: boolean;
  /** 顶部偏移量 */
  offsetTop?: number;
  /** 左侧偏移量 */
  offsetLeft?: number;
  /** 底部偏移量 */
  offsetBottom?: number;
  /** 右侧偏移量 */
  offsetRight?: number;
  /** 是否允许容器水平滚动 */
  allowHorizontalScroll?: boolean;
  /** 当内容可见时是否允许滚动容器 */
  onlyScrollIfNeeded?: boolean;
};

export type ValidateFieldsOptions = {
  /** 所有表单域是否在第一个校验规则失败后停止继续校验 */
  first?: boolean;
  /** 指定哪些表单域在第一个校验规则失败后停止继续校验 */
  firstFields?: string[];
  /** 已经校验过的表单域，在 validateTrigger 再次被触发时是否再次校验 */
  force?: boolean;
  /** 定义 validateFieldsAndScroll 的滚动行为 */
  scroll?: DomScrollIntoViewConfig;
};

// function create
export type WrappedFormUtils<V = any> = {
  /** 获取一组输入控件的值，如不传入参数，则获取全部组件的值 */
  getFieldsValue(fieldNames?: Array<string>): { [field: string]: any };
  /** 获取一个输入控件的值 */
  getFieldValue(fieldName: string): any;
  /** 设置一组输入控件的值 */
  setFieldsValue(obj: Object, callback?: Function): void;
  /** 设置一组输入控件的值 */
  setFields(obj: Object): void;
  /** 校验并获取一组输入域的值与 Error */
  validateFields(
    fieldNames: Array<string>,
    options: ValidateFieldsOptions,
    callback: ValidateCallback<V>,
  ): void;
  validateFields(options: ValidateFieldsOptions, callback: ValidateCallback<V>): void;
  validateFields(fieldNames: Array<string>, callback: ValidateCallback<V>): void;
  validateFields(fieldNames: Array<string>, options: ValidateFieldsOptions): void;
  validateFields(fieldNames: Array<string>): void;
  validateFields(callback: ValidateCallback<V>): void;
  validateFields(options: ValidateFieldsOptions): void;
  validateFields(): void;
  /** 与 `validateFields` 相似，但校验完后，如果校验不通过的菜单域不在可见范围内，则自动滚动进可见范围 */
  validateFieldsAndScroll(
    fieldNames: Array<string>,
    options: ValidateFieldsOptions,
    callback: ValidateCallback<V>,
  ): void;
  validateFieldsAndScroll(options: ValidateFieldsOptions, callback: ValidateCallback<V>): void;
  validateFieldsAndScroll(fieldNames: Array<string>, callback: ValidateCallback<V>): void;
  validateFieldsAndScroll(fieldNames: Array<string>, options: ValidateFieldsOptions): void;
  validateFieldsAndScroll(fieldNames: Array<string>): void;
  validateFieldsAndScroll(callback: ValidateCallback<V>): void;
  validateFieldsAndScroll(options: ValidateFieldsOptions): void;
  validateFieldsAndScroll(): void;
  /** 获取某个输入控件的 Error */
  getFieldError(name: string): string[] | undefined;
  getFieldsError(names?: Array<string>): Record<string, string[] | undefined>;
  /** 判断一个输入控件是否在校验状态 */
  isFieldValidating(name: string): boolean;
  isFieldTouched(name: string): boolean;
  isFieldsTouched(names?: Array<string>): boolean;
  /** 重置一组输入控件的值与状态，如不传入参数，则重置所有组件 */
  resetFields(names?: Array<string>): void;
  // tslint:disable-next-line:max-line-length
  getFieldDecorator<T extends Object = {}>(
    id: keyof T,
    options?: GetFieldDecoratorOptions,
  ): (node: React.ReactNode) => React.ReactNode;
};

export interface WrappedFormInternalProps<V = any> {
  form: WrappedFormUtils<V>;
}

export interface RcBaseFormProps {
  wrappedComponentRef?: any;
}

export interface FormComponentProps<V = any> extends WrappedFormInternalProps<V>, RcBaseFormProps {
  form: WrappedFormUtils<V>;
}

export default class Form extends React.Component<FormProps, any> {
  static defaultProps = {
    colon: true,
    layout: 'horizontal' as FormLayout,
    hideRequiredMark: false,
    onSubmit(e: React.FormEvent<HTMLFormElement>) {
      e.preventDefault();
    },
  };

  static propTypes = {
    prefixCls: PropTypes.string,
    layout: PropTypes.oneOf(FormLayouts),
    children: PropTypes.any,
    onSubmit: PropTypes.func,
    hideRequiredMark: PropTypes.bool,
    colon: PropTypes.bool,
  };

  static Item = FormItem;

  static createFormField = createFormField;

  // 关键函数 create
  static create = function<TOwnProps extends FormComponentProps>(
    options: FormCreateOption<TOwnProps> = {},
  ): FormWrappedProps<TOwnProps> {
    // createDOMForm 是一个装饰器函数，会返回入参为 WrappedComponent 的函数，该函数可以构建一个 rc-form
    // formPropName 默认参数为 form，rc-form 的主体会挂载到 [formPropName] 中，不传该参数则默认挂载到 props.form 属性中
    return createDOMForm({
      fieldNameProp: 'id',
      ...options,
      // 定义 fieldMetaProp 和 fieldDataProp，在 rc-form 底层，会将 field 和 fieldMeta 信息挂载在与之同名的 props 下
      fieldMetaProp: FIELD_META_PROP,
      fieldDataProp: FIELD_DATA_PROP,
    });
  };

  constructor(props: FormProps) {
    super(props);

    warning(!props.form, 'Form', 'It is unnecessary to pass `form` to `Form` after antd@1.7.0.');
  }

  renderForm = ({ getPrefixCls }: ConfigConsumerProps) => {
    const { prefixCls: customizePrefixCls, hideRequiredMark, className = '', layout } = this.props;
    const prefixCls = getPrefixCls('form', customizePrefixCls);
    const formClassName = classNames(
      prefixCls,
      {
        [`${prefixCls}-horizontal`]: layout === 'horizontal',
        [`${prefixCls}-vertical`]: layout === 'vertical',
        [`${prefixCls}-inline`]: layout === 'inline',
        [`${prefixCls}-hide-required-mark`]: hideRequiredMark,
      },
      className,
    );

    const formProps = omit(this.props, [
      'prefixCls',
      'className',
      'layout',
      'form',
      'hideRequiredMark',
      'wrapperCol',
      'labelAlign',
      'labelCol',
      'colon',
    ]);

    // 这里没有 children ，是如何把子节点显示出来的？
    // 原生 DOM 就有 children 属性，children 属性就是子节点
    return <form {...formProps} className={formClassName} />;
  };

  render() {
    const { wrapperCol, labelAlign, labelCol, layout, colon } = this.props;
    return (
      <FormContext.Provider
        value={{ wrapperCol, labelAlign, labelCol, vertical: layout === 'vertical', colon }}
      >
        <ConfigConsumer>{this.renderForm}</ConfigConsumer>
      </FormContext.Provider>
    );
  }
}
```