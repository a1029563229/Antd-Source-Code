# responsiveObserve

```ts
// matchMedia polyfill for
// https://github.com/WickyNilliams/enquire.js/issues/82
let enquire: any;

if (typeof window !== 'undefined') {
  const matchMediaPolyfill = (mediaQuery: string) => {
    return {
      media: mediaQuery,
      matches: false,
      addListener() {},
      removeListener() {},
    };
  };
  window.matchMedia = window.matchMedia || matchMediaPolyfill;
  // 响应css媒体查询的轻量级javascript库
  enquire = require('enquire.js');
}

export type Breakpoint = 'xxl' | 'xl' | 'lg' | 'md' | 'sm' | 'xs';
export type BreakpointMap = Partial<Record<Breakpoint, string>>;

export const responsiveArray: Breakpoint[] = ['xxl', 'xl', 'lg', 'md', 'sm', 'xs'];

// 定义不同属性的触发条件
export const responsiveMap: BreakpointMap = {
  xs: '(max-width: 575px)',
  sm: '(min-width: 576px)',
  md: '(min-width: 768px)',
  lg: '(min-width: 992px)',
  xl: '(min-width: 1200px)',
  xxl: '(min-width: 1600px)',
};

type SubscribeFunc = (screens: BreakpointMap) => void;

let subscribers: Array<{
  token: string;
  func: SubscribeFunc;
}> = [];
let subUid = -1;
let screens = {};

const responsiveObserve = {
  dispatch(pointMap: BreakpointMap) {
    // pointMap 是媒体查询信息集合，例：{ xs: false, sm: false, md: false, lg: false, xl: false, xxl: true }
    screens = pointMap;
    // 没有订阅者则退出
    if (subscribers.length < 1) {
      return false;
    }

    // 依次通知到每个订阅者
    subscribers.forEach(item => {
      item.func(screens);
    });

    return true;
  },

  // 订阅
  subscribe(func: SubscribeFunc) {
    // 如果订阅者为0（第一次订阅），则调用注册 API
    if (subscribers.length === 0) {
      this.register();
    }
    // token 就是一个递增的唯一性标识
    const token = (++subUid).toString();
    // 在订阅者列表添加新的订阅者
    subscribers.push({
      token: token,
      func: func,
    });
    // 执行函数，将设备信息作为参数传入，返回 token
    func(screens);
    return token;
  },

  // 取消订阅
  unsubscribe(token: string) {
    // 取出指定 token 的订阅者
    subscribers = subscribers.filter(item => item.token !== token);
    // 如果订阅者为 0，则取消对媒体适配信息的监听事件
    if (subscribers.length === 0) {
      this.unregister();
    }
  },

  // 取消对媒体适配信息的监听事件
  unregister() {
    Object.keys(responsiveMap).map((screen: Breakpoint) =>
      enquire.unregister(responsiveMap[screen]),
    );
  },

  register() {
    // 遍历 responsiveMap
    Object.keys(responsiveMap).map((screen: Breakpoint) =>
      // screen 为屏幕兼容信息、media 查询，例如：(max-width: 575px)
      // 注册媒体查询相关信息
      enquire.register(responsiveMap[screen], {
        match: () => {
          const pointMap = {
            ...screens,
            [screen]: true,
          };
          // 媒体匹配时 触发 dispatch 通知
          this.dispatch(pointMap);
        },
        unmatch: () => {
          const pointMap = {
            ...screens,
            [screen]: false,
          };
          // 媒体不匹配时 触发 dispatch 通知
          this.dispatch(pointMap);
        },
        // Keep a empty destory to avoid triggering unmatch when unregister
        destroy() {},
      }),
    );
  },
};

export default responsiveObserve;
```