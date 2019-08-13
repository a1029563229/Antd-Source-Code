# createBaseForm

```es6
/* eslint-disable react/prefer-es6-class */
/* eslint-disable prefer-promise-reject-errors */

import React from "react";
// 不通过 es6，通过 create-react-class 来创建 React 组件
import createReactClass from "create-react-class";
// 一个异步验证插件，用于验证值
import AsyncValidator from "async-validator";
// 在满足条件时弹出提示
import warning from "warning";
import get from "lodash/get";
import set from "lodash/set";
// _.eq(value, other) 比较两者的值是否相等
import eq from "lodash/eq";
// 创建 fields 集合的组件
import createFieldsStore from "./createFieldsStore";
import {
  argumentContainer,
  identity,
  normalizeValidateRules,
  getValidateTriggers,
  getValueFromEvent,
  hasRules,
  getParams,
  isEmptyObject,
  flattenArray
} from "./utils";

const DEFAULT_TRIGGER = "onChange";

// 柯里化函数，在一个函数传入配置项，可以批量生产一系列的实例
function createBaseForm(option = {}, mixins = []) {
  const {
    validateMessages,
    onFieldsChange,
    onValuesChange,
    mapProps = identity,
    mapPropsToFields,
    fieldNameProp,
    fieldMetaProp,
    fieldDataProp,
    formPropName = "form",
    name: formName,
    // @deprecated
    withRef
  } = option;

  return function decorate(WrappedComponent) {
    const Form = createReactClass({
      mixins,

      getInitialState() {
        const fields = mapPropsToFields && mapPropsToFields(this.props);
        this.fieldsStore = createFieldsStore(fields || {});

        this.instances = {};
        this.cachedBind = {};
        this.clearedFieldMetaCache = {};

        this.renderFields = {};
        this.domFields = {};

        // HACK: https://github.com/ant-design/ant-design/issues/6406
        // 批量赋值方法
        [
          "getFieldsValue",
          "getFieldValue",
          "setFieldsInitialValue",
          "getFieldsError",
          "getFieldError",
          "isFieldValidating",
          "isFieldsValidating",
          "isFieldsTouched",
          "isFieldTouched"
        ].forEach(key => {
          this[key] = (...args) => {
            if (process.env.NODE_ENV !== "production") {
              warning(
                false,
                "you should not use `ref` on enhanced form, please use `wrappedComponentRef`. " +
                  "See: https://github.com/react-component/form#note-use-wrappedcomponentref-instead-of-withref-after-rc-form140"
              );
            }
            return this.fieldsStore[key](...args);
          };
        });

        return {
          submitting: false
        };
      },

      componentDidMount() {
        this.cleanUpUselessFields();
      },

      // 装饰器模式的另一个作用是创建一个属于自己的作用域，在作用域范围内可以访问一些变量
      componentWillReceiveProps(nextProps) {
        if (mapPropsToFields) {
          this.fieldsStore.updateFields(mapPropsToFields(nextProps));
        }
      },

      componentDidUpdate() {
        this.cleanUpUselessFields();
      },

      onCollectCommon(name, action, args) {
        const fieldMeta = this.fieldsStore.getFieldMeta(name);
        // [action] 可能是 normalize 这类的格式化函数，进入值收集之前，可能会先进行格式化处理
        if (fieldMeta[action]) {
          fieldMeta[action](...args);
        } else if (fieldMeta.originalProps && fieldMeta.originalProps[action]) {
          fieldMeta.originalProps[action](...args);
        }

        // 可以把 onChange 的参数（如 event）转化为控件的值，通过 getValueFromEvent 来获取值
        // getValueFromEvent 的默认第一个入参是 event
        const value = fieldMeta.getValueFromEvent
          ? fieldMeta.getValueFromEvent(...args)
          : getValueFromEvent(...args);
        // 当值发生改变时，将会触发 onValuesChange 函数
        if (onValuesChange && value !== this.fieldsStore.getFieldValue(name)) {
          const valuesAll = this.fieldsStore.getAllValues();
          const valuesAllSet = {};
          valuesAll[name] = value;
          Object.keys(valuesAll).forEach(key =>
            set(valuesAllSet, key, valuesAll[key])
          );
          // onValuesChange(props, changedValues, allValues) => void
          onValuesChange(
            {
              [formPropName]: this.getForm(),
              ...this.props
            },
            set({}, name, value),
            valuesAllSet
          );
        }
        const field = this.fieldsStore.getField(name);
        // touched 属性用于 isFieldsTouched
        // 判断是否任一输入控件经历过 getFieldDecorator 的值收集时机 options.trigger
        // option.trigger 默认为 onChange，反推可得出，默认在 Input 触发 onChange 事件时，则触发 onCollectCommon（收集组件值事件）
        return { name, field: { ...field, value, touched: true }, fieldMeta };
      },

      // 收集新的 field
      onCollect(name_, action, ...args) {
        const { name, field, fieldMeta } = this.onCollectCommon(
          name_,
          action,
          args
        );
        const { validate } = fieldMeta;

        // 给需要验证的 field 加入脏检查机制
        this.fieldsStore.setFieldsAsDirty();

        // 单独检查该字段的脏检查
        const newField = {
          ...field,
          dirty: hasRules(validate)
        };
        this.setFields({
          [name]: newField
        });
      },

      onCollectValidate(name_, action, ...args) {
        const { field, fieldMeta } = this.onCollectCommon(name_, action, args);
        const newField = {
          ...field,
          dirty: true
        };

        this.fieldsStore.setFieldsAsDirty();

        // 应该是个校验值是否符合规则的检查函数
        this.validateFieldsInternal([newField], {
          action,
          options: {
            // firstFields：指定表单域会在碰到第一个失败了的校验规则后停止校验
            firstFields: !!fieldMeta.validateFirst
          }
        });
      },

      // 给传入的 action 函数，绑定 this 指针
      getCacheBind(name, action, fn) {
        if (!this.cachedBind[name]) {
          this.cachedBind[name] = {};
        }
        const cache = this.cachedBind[name];
        if (!cache[action] || cache[action].oriFn !== fn) {
          cache[action] = {
            fn: fn.bind(this, name, action),
            oriFn: fn
          };
        }
        return cache[action].fn;
      },

      // 核心功能
      // name 字段名
      // fieldOption 可选项
      // 经过 getFieldDecorator 包装的控件，
      // 表单控件会自动添加 value（或 valuePropName 指定的其他属性） onChange（或 trigger 指定的其他属性），数据同步将被 Form 接管
      getFieldDecorator(name, fieldOption) {
        // 返回了一个完成了事件绑定（收集事件和验证事件）的 props
        const props = this.getFieldProps(name, fieldOption);
        // getFieldDecorator 会返回一个函数，函数接收一个 ReactElement 作为入参
        return fieldElem => {
          // We should put field in record if it is rendered
          this.renderFields[name] = true;

          const fieldMeta = this.fieldsStore.getFieldMeta(name);
          const originalProps = fieldElem.props;
          if (process.env.NODE_ENV !== "production") {
            const valuePropName = fieldMeta.valuePropName;
            warning(
              !(valuePropName in originalProps),
              `\`getFieldDecorator\` will override \`${valuePropName}\`, ` +
                `so please don't set \`${valuePropName}\` directly ` +
                `and use \`setFieldsValue\` to set it.`
            );
            const defaultValuePropName = `default${valuePropName[0].toUpperCase()}${valuePropName.slice(
              1
            )}`;
            warning(
              !(defaultValuePropName in originalProps),
              `\`${defaultValuePropName}\` is invalid ` +
                `for \`getFieldDecorator\` will set \`${valuePropName}\`,` +
                ` please use \`option.initialValue\` instead.`
            );
          }
          fieldMeta.originalProps = originalProps;
          // 在这里做一个指针引用，引用原始节点
          fieldMeta.ref = fieldElem.ref;
          // 函数最后会返回一个 fieldElem 的 clone 节点，而新的节点包含了注册过后的 props，例如 onChange、value
          // 这些 props 会被传递给下一级的节点，例如 Input 节点，然后完成事件和属性绑定，获得一个超级节点（如 Input）
          return React.cloneElement(fieldElem, {
            ...props,
            ...this.fieldsStore.getFieldValuePropValue(fieldMeta)
          });
        };
      },

      /**
       * 获取字段的属性
       * @param usersFieldOption 包含的属性如下
       * @param {function} options.getValueFromEvent(...args) 可以把 onChange 的参数（如 event）转化为控件的值
       * @param {any} options.initialValue 子节点的初始值，类型、可选值均由子节点决定(注意：由于内部校验时使用 === 判断是否变化，建议使用变量缓存所需设置的值而非直接使用字面量))
       * @param {function} options.normalize function(value, prevValue, allValues): any 转换默认的 value 给控件，需要返回转化后的 value（可用于 checkbox 全选和少选）
       * @param {boolean} options.preserve 即便字段不再使用，也保留该字段的值
       * @param {object[]} options.rules 校验规则
       * @param {string} options.trigger 收集子节点的时机
       * @param {boolean} options.validateFirst 当某一规则不通过时，是否停止剩下的规则的校验
       * @param {string|string[]} options.validateTrigger 校验子节点值的时机
       * @param {string} valuePropName 子节点的值的属性，如 Switch 的是 'checked'
       * */
      getFieldProps(name, usersFieldOption = {}) {
        if (!name) {
          throw new Error("Must call `getFieldProps` with valid name string!");
        }
        if (process.env.NODE_ENV !== "production") {
          warning(
            this.fieldsStore.isValidNestedFieldName(name),
            `One field name cannot be part of another, e.g. \`a\` and \`a.b\`. Check field: ${name}`
          );
          warning(
            !("exclusive" in usersFieldOption),
            "`option.exclusive` of `getFieldProps`|`getFieldDecorator` had been remove."
          );
        }

        delete this.clearedFieldMetaCache[name];

        const fieldOption = {
          name,
          trigger: DEFAULT_TRIGGER,
          valuePropName: "value",
          validate: [],
          ...usersFieldOption
        };

        const {
          rules,
          trigger,
          // validateTrigger 默认取 trigger 的值，所以校验子节点值的时机，默认与收集子节点的值的时机一致
          validateTrigger = trigger,
          validate
        } = fieldOption;

        const fieldMeta = this.fieldsStore.getFieldMeta(name);
        if ("initialValue" in fieldOption) {
          fieldMeta.initialValue = fieldOption.initialValue;
        }

        const inputProps = {
          ...this.fieldsStore.getFieldValuePropValue(fieldOption),
          ref: this.getCacheBind(name, `${name}__ref`, this.saveRef)
        };
        if (fieldNameProp) {
          inputProps[fieldNameProp] = formName ? `${formName}_${name}` : name;
        }

        // 根据 validateTrigger 生成 trigger
        const validateRules = normalizeValidateRules(
          validate,
          rules,
          validateTrigger
        );
        const validateTriggers = getValidateTriggers(validateRules);
        // action 为 onChange、onBlur 这类的事件
        // validateTrigger 使用的是 forEach 进行绑定，所以可以接收多个 validateTrigger
        validateTriggers.forEach(action => {
          if (inputProps[action]) return;
          inputProps[action] = this.getCacheBind(
            name,
            action,
            this.onCollectValidate
          );
        });

        // make sure that the value will be collect
        // 如果 trigger 和 validateTrigger 的触发事件一致，则不单独绑定收集值事件
        // onCollectValidate 在验证值的同时会完成值的收集
        if (trigger && validateTriggers.indexOf(trigger) === -1) {
          inputProps[trigger] = this.getCacheBind(
            name,
            trigger,
            this.onCollect
          );
        }

        const meta = {
          ...fieldMeta,
          ...fieldOption,
          validate: validateRules
        };
        this.fieldsStore.setFieldMeta(name, meta);
        if (fieldMetaProp) {
          inputProps[fieldMetaProp] = meta;
        }

        if (fieldDataProp) {
          inputProps[fieldDataProp] = this.fieldsStore.getField(name);
        }

        // This field is rendered, record it
        this.renderFields[name] = true;

        return inputProps;
      },

      getFieldInstance(name) {
        return this.instances[name];
      },

      getRules(fieldMeta, action) {
        const actionRules = fieldMeta.validate
          .filter(item => {
            return !action || item.trigger.indexOf(action) >= 0;
          })
          .map(item => item.rules);
        return flattenArray(actionRules);
      },

      setFields(maybeNestedFields, callback) {
        const fields = this.fieldsStore.flattenRegisteredFields(
          maybeNestedFields
        );
        this.fieldsStore.setFields(fields);
        // 当字段发生改变时，触发该函数
        if (onFieldsChange) {
          const changedFields = Object.keys(fields).reduce(
            (acc, name) => set(acc, name, this.fieldsStore.getField(name)),
            {}
          );
          onFieldsChange(
            {
              [formPropName]: this.getForm(),
              ...this.props
            },
            changedFields,
            this.fieldsStore.getNestedAllFields()
          );
        }
        this.forceUpdate(callback);
      },

      // ({ [fieldName]: value }, callback: Function ) => void
      // 设置一组输入控件的值
      // 该函数会同时触发 onValuesChange 和 onFieldsChange 事件
      setFieldsValue(changedValues, callback) {
        const { fieldsMeta } = this.fieldsStore;
        const values = this.fieldsStore.flattenRegisteredFields(changedValues);
        const newFields = Object.keys(values).reduce((acc, name) => {
          const isRegistered = fieldsMeta[name];
          if (process.env.NODE_ENV !== "production") {
            warning(
              isRegistered,
              "Cannot use `setFieldsValue` until " +
                "you use `getFieldDecorator` or `getFieldProps` to register it."
            );
          }
          if (isRegistered) {
            const value = values[name];
            acc[name] = {
              value
            };
          }
          return acc;
        }, {});
        // 设置字段
        this.setFields(newFields, callback);
        if (onValuesChange) {
          const allValues = this.fieldsStore.getAllValues();
          onValuesChange(
            {
              [formPropName]: this.getForm(),
              ...this.props
            },
            changedValues,
            allValues
          );
        }
      },

      saveRef(name, _, component) {
        if (!component) {
          const fieldMeta = this.fieldsStore.getFieldMeta(name);
          if (!fieldMeta.preserve) {
            // after destroy, delete data
            this.clearedFieldMetaCache[name] = {
              field: this.fieldsStore.getField(name),
              meta: fieldMeta
            };
            this.clearField(name);
          }
          delete this.domFields[name];
          return;
        }
        this.domFields[name] = true;
        this.recoverClearedField(name);
        const fieldMeta = this.fieldsStore.getFieldMeta(name);
        if (fieldMeta) {
          const ref = fieldMeta.ref;
          if (ref) {
            if (typeof ref === "string") {
              throw new Error(`can not set ref string for ${name}`);
            } else if (typeof ref === "function") {
              ref(component);
            } else if (Object.prototype.hasOwnProperty.call(ref, "current")) {
              ref.current = component;
            }
          }
        }
        this.instances[name] = component;
      },

      cleanUpUselessFields() {
        const fieldList = this.fieldsStore.getAllFieldsName();
        const removedList = fieldList.filter(field => {
          const fieldMeta = this.fieldsStore.getFieldMeta(field);
          return (
            !this.renderFields[field] &&
            !this.domFields[field] &&
            !fieldMeta.preserve
          );
        });
        if (removedList.length) {
          removedList.forEach(this.clearField);
        }
        this.renderFields = {};
      },

      clearField(name) {
        this.fieldsStore.clearField(name);
        delete this.instances[name];
        delete this.cachedBind[name];
      },

      resetFields(ns) {
        const newFields = this.fieldsStore.resetFields(ns);
        if (Object.keys(newFields).length > 0) {
          this.setFields(newFields);
        }
        if (ns) {
          const names = Array.isArray(ns) ? ns : [ns];
          names.forEach(name => delete this.clearedFieldMetaCache[name]);
        } else {
          this.clearedFieldMetaCache = {};
        }
      },

      recoverClearedField(name) {
        if (this.clearedFieldMetaCache[name]) {
          this.fieldsStore.setFields({
            [name]: this.clearedFieldMetaCache[name].field
          });
          this.fieldsStore.setFieldMeta(
            name,
            this.clearedFieldMetaCache[name].meta
          );
          delete this.clearedFieldMetaCache[name];
        }
      },

      validateFieldsInternal(
        fields,
        { fieldNames, action, options = {} },
        callback
      ) {
        const allRules = {};
        const allValues = {};
        const allFields = {};
        const alreadyErrors = {};
        fields.forEach(field => {
          const name = field.name;
          if (options.force !== true && field.dirty === false) {
            if (field.errors) {
              set(alreadyErrors, name, { errors: field.errors });
            }
            return;
          }
          const fieldMeta = this.fieldsStore.getFieldMeta(name);
          const newField = {
            ...field
          };
          newField.errors = undefined;
          // 在验证值的同时，validating 值已经变成了 true，但是在控制台实际输出的是 false，猜想在某处该值又被重置为了 false
          newField.validating = true;
          newField.dirty = true;
          allRules[name] = this.getRules(fieldMeta, action);
          allValues[name] = newField.value;
          allFields[name] = newField;
        });
        this.setFields(allFields);
        // in case normalize
        Object.keys(allValues).forEach(f => {
          allValues[f] = this.fieldsStore.getFieldValue(f);
        });
        if (callback && isEmptyObject(allFields)) {
          callback(
            isEmptyObject(alreadyErrors) ? null : alreadyErrors,
            this.fieldsStore.getFieldsValue(fieldNames)
          );
          return;
        }
        const validator = new AsyncValidator(allRules);
        if (validateMessages) {
          validator.messages(validateMessages);
        }
        // 由 AsyncValidator 接管实际的校验工作
        validator.validate(allValues, options, errors => {
          const errorsGroup = {
            ...alreadyErrors
          };
          if (errors && errors.length) {
            errors.forEach(e => {
              const errorFieldName = e.field;
              let fieldName = errorFieldName;

              // Handle using array validation rule.
              // ref: https://github.com/ant-design/ant-design/issues/14275
              Object.keys(allRules).some(ruleFieldName => {
                const rules = allRules[ruleFieldName] || [];

                // Exist if match rule
                if (ruleFieldName === errorFieldName) {
                  fieldName = ruleFieldName;
                  return true;
                }

                // Skip if not match array type
                if (
                  rules.every(({ type }) => type !== "array") &&
                  errorFieldName.indexOf(ruleFieldName) !== 0
                ) {
                  return false;
                }

                // Exist if match the field name
                const restPath = errorFieldName.slice(ruleFieldName.length + 1);
                if (/^\d+$/.test(restPath)) {
                  fieldName = ruleFieldName;
                  return true;
                }

                return false;
              });

              const field = get(errorsGroup, fieldName);
              if (typeof field !== "object" || Array.isArray(field)) {
                set(errorsGroup, fieldName, { errors: [] });
              }
              const fieldErrors = get(errorsGroup, fieldName.concat(".errors"));
              fieldErrors.push(e);
            });
          }
          const expired = [];
          const nowAllFields = {};
          Object.keys(allRules).forEach(name => {
            const fieldErrors = get(errorsGroup, name);
            const nowField = this.fieldsStore.getField(name);
            // avoid concurrency problems
            if (!eq(nowField.value, allValues[name])) {
              expired.push({
                name
              });
            } else {
              nowField.errors = fieldErrors && fieldErrors.errors;
              nowField.value = allValues[name];
              // validating 的状态，在校验完成后就会变成 false，如果校验是一个网络请求，那么校验时间就会较长，否则校验的过程基本上是无法察觉的
              nowField.validating = false;
              nowField.dirty = false;
              nowAllFields[name] = nowField;
            }
          });
          this.setFields(nowAllFields);
          // 这个 callback 来自 validateFields
          if (callback) {
            if (expired.length) {
              expired.forEach(({ name }) => {
                const fieldErrors = [
                  {
                    message: `${name} need to revalidate`,
                    field: name
                  }
                ];
                set(errorsGroup, name, {
                  expired: true,
                  errors: fieldErrors
                });
              });
            }

            callback(
              isEmptyObject(errorsGroup) ? null : errorsGroup,
              this.fieldsStore.getFieldsValue(fieldNames)
            );
          }
        });
      },

      // validateFields ([fieldNames: string[]], [options: object], callback(errors, values)) => void
      // 校验并获取一组输入域的值与 Error，若 fieldNames 参数为空，则校验全部组件
      // 从组件看来该方法同样支持 Promise 的调用方法
      validateFields(ns, opt, cb) {
        const pending = new Promise((resolve, reject) => {
          // 按照先后顺序获取参数
          const { names, options } = getParams(ns, opt, cb);
          let { callback } = getParams(ns, opt, cb);
          if (!callback || typeof callback === "function") {
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
          }
          const fieldNames = names
            ? this.fieldsStore.getValidFieldsFullName(names)
            : this.fieldsStore.getValidFieldsName();
          const fields = fieldNames
            .filter(name => {
              const fieldMeta = this.fieldsStore.getFieldMeta(name);
              return hasRules(fieldMeta.validate);
            })
            .map(name => {
              const field = this.fieldsStore.getField(name);
              field.value = this.fieldsStore.getFieldValue(name);
              return field;
            });
          if (!fields.length) {
            callback(null, this.fieldsStore.getFieldsValue(fieldNames));
            return;
          }
          if (!("firstFields" in options)) {
            options.firstFields = fieldNames.filter(name => {
              const fieldMeta = this.fieldsStore.getFieldMeta(name);
              return !!fieldMeta.validateFirst;
            });
          }
          this.validateFieldsInternal(
            fields,
            {
              fieldNames,
              options
            },
            callback
          );
        });
        pending.catch(e => {
          if (console.error && process.env.NODE_ENV !== "production") {
            console.error(e);
          }
          return e;
        });
        return pending;
      },

      isSubmitting() {
        if (
          process.env.NODE_ENV !== "production" &&
          process.env.NODE_ENV !== "test"
        ) {
          warning(
            false,
            "`isSubmitting` is deprecated. " +
              "Actually, it's more convenient to handle submitting status by yourself."
          );
        }
        return this.state.submitting;
      },

      submit(callback) {
        if (
          process.env.NODE_ENV !== "production" &&
          process.env.NODE_ENV !== "test"
        ) {
          warning(
            false,
            "`submit` is deprecated. " +
              "Actually, it's more convenient to handle submitting status by yourself."
          );
        }
        const fn = () => {
          this.setState({
            submitting: false
          });
        };
        this.setState({
          submitting: true
        });
        callback(fn);
      },

      render() {
        const { wrappedComponentRef, ...restProps } = this.props; // eslint-disable-line
        const formProps = {
          [formPropName]: this.getForm()
        };
        if (withRef) {
          if (
            process.env.NODE_ENV !== "production" &&
            process.env.NODE_ENV !== "test"
          ) {
            warning(
              false,
              "`withRef` is deprecated, please use `wrappedComponentRef` instead. " +
                "See: https://github.com/react-component/form#note-use-wrappedcomponentref-instead-of-withref-after-rc-form140"
            );
          }
          formProps.ref = "wrappedComponent";
        } else if (wrappedComponentRef) {
          formProps.ref = wrappedComponentRef;
        }
        const props = mapProps.call(this, {
          ...formProps,
          ...restProps
        });
        return <WrappedComponent {...props} />;
      }
    });

    return argumentContainer(Form, WrappedComponent);
  };
}

export default createBaseForm;
```
