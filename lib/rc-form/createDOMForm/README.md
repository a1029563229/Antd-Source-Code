# createDOMForm

## createDOMForm.js
```tsx
import ReactDOM from 'react-dom';
// scrollIntoView：滚动到指定视图的指定位置
import scrollIntoView from 'dom-scroll-into-view';
// _.has(object, path) 检查 path 是否是object对象的直接属性。
import has from 'lodash/has';
import createBaseForm from './createBaseForm';
import { mixin as formMixin } from './createForm';
import { getParams } from './utils';

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

function getScrollableContainer(n) {
  let node = n;
  let nodeName;
  /* eslint no-cond-assign:0 */
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
  return nodeName === 'body' ? node.ownerDocument : node;
}

const mixin = {
  getForm() {
    return {
      ...formMixin.getForm.call(this),
      validateFieldsAndScroll: this.validateFieldsAndScroll,
    };
  },

  // validateFieldsAndScroll 与 validateFields 相似，但校验完后，如果校验不通过的菜单域不在可见范围内，则自动滚动进可见范围
  validateFieldsAndScroll(ns, opt, cb) {
    const { names, callback, options } = getParams(ns, opt, cb);

    const newCb = (error, values) => {
      if (error) {
        const validNames = this.fieldsStore.getValidFieldsName();
        // 第一个校验发生错误的节点
        let firstNode;
        let firstTop;

        validNames.forEach((name) => {
          if (has(error, name)) {
            const instance = this.getFieldInstance(name);
            if (instance) {
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
            }
          }
        });

        if (firstNode) {
          const c = options.container || getScrollableContainer(firstNode);
          scrollIntoView(firstNode, c, {
            onlyScrollIfNeeded: true,
            ...options.scroll,
          });
        }
      }

      if (typeof callback === 'function') {
        callback(error, values);
      }
    };

    return this.validateFields(names, options, newCb);
  },
};

function createDOMForm(option) {
  return createBaseForm({
    ...option,
  }, [mixin]);
}

// 相比较 createBaseForm 而言添加了一个功能：如果校验不通过的菜单域不在可见范围内，则自动滚动进可见范围
export default createDOMForm;
```