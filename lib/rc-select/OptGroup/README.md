# OptGroup.tsx
```tsx
import { Component } from 'react';

export interface IOptGroupProps {
  label: string;
  value: string | number;
  key: string | number;
  // Everything for testing
  testprop?: any;
}

// 只是多了个属性
export default class OptGroup extends Component<Partial<IOptGroupProps>> {
  public static isSelectOptGroup = true;
}
```