# Wave

# wave.tsx
```tsx
import * as React from 'react';

// ReactDOM.findDOMNode(component) 如果组件已经被挂载到 DOM 上，此方法会返回浏览器中相应的原生 DOM 元素。
// 此方法对于从 DOM 中读取值很有用，例如获取表单字段的值或者执行 DOM 检测。
// 大多数情况下，你可以绑定一个 ref 到 DOM 节点上，可以完全避免使用 findDOMNode
// 注意：findDOMNode 是一个访问底层 DOM 节点的应急方案（escape hatch）。在大多数情况下，不推荐使用该方法，因为它会破坏组件的抽象结构。
import { findDOMNode } from 'react-dom';

// anim(el,animationName,function(){}); 使用这个方法可以让 el 节点播放指定 animationName 动画
import TransitionEvents from 'css-animation/lib/Event';

// 对动画的一个 polyfill
import raf from '../_util/raf';
import { ConfigConsumer, ConfigConsumerProps, CSPConfig } from '../config-provider';

let styleForPesudo: HTMLStyleElement | null;

// Where el is the DOM element you'd like to test for visibility
function isHidden(element: HTMLElement) {
  if (process.env.NODE_ENV === 'test') {
    return false;
  }

  // 元素是否存在
  // offsetParent：在 Webkit 中，如果元素为隐藏的（该元素或其祖先元素的 style.display 为 "none"），或者该元素的 style.position 被设为 "fixed"，则该属性返回 null。
  return !element || element.offsetParent === null;
}

export default class Wave extends React.Component<{ insertExtraNode?: boolean }> {
  private instance?: {
    cancel: () => void;
  };

  private extraNode: HTMLDivElement;
  private clickWaveTimeoutId: number;
  private animationStartId: number;
  private animationStart: boolean = false;
  private destroy: boolean = false;
  private csp?: CSPConfig;

  // 是否不为灰色
  isNotGrey(color: string) {
    const match = (color || '').match(/rgba?\((\d*), (\d*), (\d*)(, [\.\d]*)?\)/);
    if (match && match[1] && match[2] && match[3]) {
      return !(match[1] === match[2] && match[2] === match[3]);
    }
    return true;
  }

  onClick = (node: HTMLElement, waveColor: string) => {
    if (!node || isHidden(node) || node.className.indexOf('-leave') >= 0) {
      return;
    }
    const { insertExtraNode } = this.props;
    this.extraNode = document.createElement('div');
    const extraNode = this.extraNode;
    extraNode.className = 'ant-click-animating-node';
    const attributeName = this.getAttributeName();
    node.setAttribute(attributeName, 'true');
    // Not white or transparnt or grey
    styleForPesudo = styleForPesudo || document.createElement('style');
    if (
      waveColor &&
      waveColor !== '#ffffff' &&
      waveColor !== 'rgb(255, 255, 255)' &&
      this.isNotGrey(waveColor) &&
      !/rgba\(\d*, \d*, \d*, 0\)/.test(waveColor) && // any transparent rgba color
      waveColor !== 'transparent'
    ) {
      // Add nonce if CSP exist
      if (this.csp && this.csp.nonce) {
        styleForPesudo.nonce = this.csp.nonce;
      }

      extraNode.style.borderColor = waveColor;
      styleForPesudo.innerHTML = `
      [ant-click-animating-without-extra-node='true']::after, .ant-click-animating-node {
        --antd-wave-shadow-color: ${waveColor};
      }`;
      if (!document.body.contains(styleForPesudo)) {
        document.body.appendChild(styleForPesudo);
      }
    }
    if (insertExtraNode) {
      node.appendChild(extraNode);
    }
    TransitionEvents.addStartEventListener(node, this.onTransitionStart);
    TransitionEvents.addEndEventListener(node, this.onTransitionEnd);
  };

  bindAnimationEvent = (node: HTMLElement) => {
    if (
      !node ||
      !node.getAttribute ||
      node.getAttribute('disabled') ||
      node.className.indexOf('disabled') >= 0
    ) {
      return;
    }
    const onClick = (e: MouseEvent) => {
      // Fix radio button click twice
      if ((e.target as HTMLElement).tagName === 'INPUT' || isHidden(e.target as HTMLElement)) {
        return;
      }
      this.resetEffect(node);
      // Get wave color from target
      const waveColor =
        getComputedStyle(node).getPropertyValue('border-top-color') || // Firefox Compatible
        getComputedStyle(node).getPropertyValue('border-color') ||
        getComputedStyle(node).getPropertyValue('background-color');
      this.clickWaveTimeoutId = window.setTimeout(() => this.onClick(node, waveColor), 0);

      raf.cancel(this.animationStartId);
      this.animationStart = true;

      // Render to trigger transition event cost 3 frames. Let's delay 10 frames to reset this.
      this.animationStartId = raf(() => {
        this.animationStart = false;
      }, 10);
    };
    node.addEventListener('click', onClick, true);
    return {
      cancel: () => {
        node.removeEventListener('click', onClick, true);
      },
    };
  };

  getAttributeName() {
    const { insertExtraNode } = this.props;
    return insertExtraNode ? 'ant-click-animating' : 'ant-click-animating-without-extra-node';
  }

  resetEffect(node: HTMLElement) {
    if (!node || node === this.extraNode || !(node instanceof Element)) {
      return;
    }
    const { insertExtraNode } = this.props;
    const attributeName = this.getAttributeName();
    node.setAttribute(attributeName, 'false'); // edge has bug on `removeAttribute` #14466
    this.removeExtraStyleNode();
    if (insertExtraNode && this.extraNode && node.contains(this.extraNode)) {
      node.removeChild(this.extraNode);
    }
    TransitionEvents.removeStartEventListener(node, this.onTransitionStart);
    TransitionEvents.removeEndEventListener(node, this.onTransitionEnd);
  }

  onTransitionStart = (e: AnimationEvent) => {
    if (this.destroy) return;

    const node = findDOMNode(this) as HTMLElement;
    if (!e || e.target !== node) {
      return;
    }

    if (!this.animationStart) {
      this.resetEffect(node);
    }
  };

  onTransitionEnd = (e: AnimationEvent) => {
    if (!e || e.animationName !== 'fadeEffect') {
      return;
    }
    this.resetEffect(e.target as HTMLElement);
  };

  removeExtraStyleNode() {
    if (styleForPesudo) {
      styleForPesudo.innerHTML = '';
    }
  }

  componentDidMount() {
    const node = findDOMNode(this) as HTMLElement;
    if (!node || node.nodeType !== 1) {
      return;
    }
    this.instance = this.bindAnimationEvent(node);
  }

  componentWillUnmount() {
    if (this.instance) {
      this.instance.cancel();
    }
    if (this.clickWaveTimeoutId) {
      clearTimeout(this.clickWaveTimeoutId);
    }

    this.destroy = true;
  }

  renderWave = ({ csp }: ConfigConsumerProps) => {
    const { children } = this.props;
    this.csp = csp;

    return children;
  };

  render() {
    return <ConfigConsumer>{this.renderWave}</ConfigConsumer>;
  }
}

```