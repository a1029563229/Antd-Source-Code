# Icon

## IconFont.tsx
```tsx
import Icon, { IconProps } from './index';
import * as React from 'react';

// 用 Set 创建一个缓存区，如果遇到重复加载则自动忽略
const customCache = new Set<string>();

export interface CustomIconOptions {
  scriptUrl?: string;
  extraCommonProps?: { [key: string]: any };
}

export default function create(options: CustomIconOptions = {}): React.SFC<IconProps> {
  const { scriptUrl, extraCommonProps = {} } = options;

  /**
   * DOM API required.
   * Make sure in browser environment.
   * The Custom Icon will create a <script/>
   * that loads SVG symbols and insert the SVG Element into the document body.
   */
  if (
    typeof document !== 'undefined' &&
    typeof window !== 'undefined' &&
    typeof document.createElement === 'function' &&
    typeof scriptUrl === 'string' &&
    scriptUrl.length &&
    !customCache.has(scriptUrl)
  ) {
    // 动态创建一个 script 标签用于加载 Icon svg
    const script = document.createElement('script');
    script.setAttribute('src', scriptUrl);
    script.setAttribute('data-namespace', scriptUrl);
    customCache.add(scriptUrl);
    document.body.appendChild(script);
  }

  const Iconfont: React.SFC<IconProps> = props => {
    const { type, children, ...restProps } = props;

    // component > children > type
    let content = null;
    if (props.type) {
      // 使用 use 标签寻找挂载在别处的 svg 图标
      content = <use xlinkHref={`#${type}`} />;
    }
    if (children) {
      content = children;
    }
    return (
      <Icon {...restProps} {...extraCommonProps}>
        {content}
      </Icon>
    );
  };

  Iconfont.displayName = 'Iconfont';

  return Iconfont;
}
```

## index.tsx
```tsx
import * as React from 'react';
import classNames from 'classnames';
import * as allIcons from '@ant-design/icons/lib/dist';
import ReactIcon from '@ant-design/icons-react';
import createFromIconfontCN from './IconFont';
import {
  svgBaseProps,
  withThemeSuffix,
  removeTypeTheme,
  getThemeFromTypeName,
  alias,
} from './utils';
import warning from '../_util/warning';
import LocaleReceiver from '../locale-provider/LocaleReceiver';
import { getTwoToneColor, setTwoToneColor } from './twoTonePrimaryColor';

// Initial setting
ReactIcon.add(...Object.keys(allIcons).map(key => (allIcons as any)[key]));
setTwoToneColor('#1890ff');
let defaultTheme: ThemeType = 'outlined';
let dangerousTheme: ThemeType | undefined = undefined;

export interface TransferLocale {
  icon: string;
}

export interface CustomIconComponentProps {
  width: string | number;
  height: string | number;
  fill: string;
  viewBox?: string;
  className?: string;
  style?: React.CSSProperties;
  spin?: boolean;
  rotate?: number;
  ['aria-hidden']?: React.AriaAttributes['aria-hidden'];
}

export type ThemeType = 'filled' | 'outlined' | 'twoTone';

export interface IconProps {
  tabIndex?: number;
  type?: string;
  className?: string;
  theme?: ThemeType;
  title?: string;
  onKeyUp?: React.KeyboardEventHandler<HTMLElement>;
  onClick?: React.MouseEventHandler<HTMLElement>;
  component?: React.ComponentType<CustomIconComponentProps>;
  twoToneColor?: string;
  viewBox?: string;
  spin?: boolean;
  rotate?: number;
  style?: React.CSSProperties;
  prefixCls?: string;
  role?: string;
}

export interface IconComponent<P> extends React.SFC<P> {
  createFromIconfontCN: typeof createFromIconfontCN;
  getTwoToneColor: typeof getTwoToneColor;
  setTwoToneColor: typeof setTwoToneColor;
  unstable_ChangeThemeOfIconsDangerously?: typeof unstable_ChangeThemeOfIconsDangerously;
  unstable_ChangeDefaultThemeOfIcons?: typeof unstable_ChangeDefaultThemeOfIcons;
}

const Icon: IconComponent<IconProps> = props => {
  const {
    // affect outter <i>...</i>
    className,

    // affect inner <svg>...</svg>
    type,
    component: Component,
    viewBox,
    spin,
    rotate,

    tabIndex,
    onClick,

    // children
    children,

    // other
    theme, // default to outlined
    twoToneColor,

    ...restProps
  } = props;

  warning(
    Boolean(type || Component || children),
    'Icon',
    'Should have `type` prop or `component` prop or `children`.',
  );

  const classString = classNames(
    {
      [`anticon`]: true,
      [`anticon-${type}`]: Boolean(type),
    },
    className,
  );

  // 给部分满足条件的选项加上旋转样式
  const svgClassString = classNames({
    [`anticon-spin`]: !!spin || type === 'loading',
  });

  let innerNode: React.ReactNode;

  const svgStyle = rotate
    ? {
        msTransform: `rotate(${rotate}deg)`,
        transform: `rotate(${rotate}deg)`,
      }
    : undefined;

  const innerSvgProps: CustomIconComponentProps = {
    ...svgBaseProps,
    className: svgClassString,
    style: svgStyle,
    viewBox,
  };

  if (!viewBox) {
    delete innerSvgProps.viewBox;
  }

  // component > children > type
  if (Component) {
    innerNode = <Component {...innerSvgProps}>{children}</Component>;
  }

  if (children) {
    warning(
      Boolean(viewBox) ||
        (React.Children.count(children) === 1 &&
          React.isValidElement(children) &&
          React.Children.only(children).type === 'use'),
      'Icon',
      'Make sure that you provide correct `viewBox`' +
        ' prop (default `0 0 1024 1024`) to the icon.',
    );
    innerNode = (
      <svg {...innerSvgProps} viewBox={viewBox}>
        {children}
      </svg>
    );
  }

  if (typeof type === 'string') {
    let computedType = type;
    if (theme) {
      const themeInName = getThemeFromTypeName(type);
      warning(
        !themeInName || theme === themeInName,
        'Icon',
        `The icon name '${type}' already specify a theme '${themeInName}',` +
          ` the 'theme' prop '${theme}' will be ignored.`,
      );
    }
    computedType = withThemeSuffix(
      removeTypeTheme(alias(computedType)),
      dangerousTheme || theme || defaultTheme,
    );
    innerNode = (
      <ReactIcon
        className={svgClassString}
        type={computedType}
        primaryColor={twoToneColor}
        style={svgStyle}
      />
    );
  }

  let iconTabIndex = tabIndex;
  if (iconTabIndex === undefined && onClick) {
    iconTabIndex = -1;
  }

  return (
    <LocaleReceiver componentName="Icon">
      {(locale: TransferLocale) => (
        <i
          aria-label={type && `${locale.icon}: ${type}`}
          {...restProps}
          tabIndex={iconTabIndex}
          onClick={onClick}
          className={classString}
        >
          {innerNode}
        </i>
      )}
    </LocaleReceiver>
  );
};

function unstable_ChangeThemeOfIconsDangerously(theme?: ThemeType) {
  warning(
    false,
    'Icon',
    `You are using the unstable method 'Icon.unstable_ChangeThemeOfAllIconsDangerously', ` +
      `make sure that all the icons with theme '${theme}' display correctly.`,
  );
  dangerousTheme = theme;
}

function unstable_ChangeDefaultThemeOfIcons(theme: ThemeType) {
  warning(
    false,
    'Icon',
    `You are using the unstable method 'Icon.unstable_ChangeDefaultThemeOfIcons', ` +
      `make sure that all the icons with theme '${theme}' display correctly.`,
  );
  defaultTheme = theme;
}

Icon.createFromIconfontCN = createFromIconfontCN;
Icon.getTwoToneColor = getTwoToneColor;
Icon.setTwoToneColor = setTwoToneColor;

export default Icon;
```