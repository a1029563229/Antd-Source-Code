# createFormField

```es6
class Field {
  constructor(fields) {
    // 将 fields 属性赋值给当前类
    // 批量赋值
    Object.assign(this, fields);
  }
}

export function isFormField(obj) {
  return obj instanceof Field;
}

export default function createFormField(field) {
  if (isFormField(field)) {
    return field;
  }
  return new Field(field);
}
```