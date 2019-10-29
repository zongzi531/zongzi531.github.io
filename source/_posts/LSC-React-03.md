---
title: React 源码学习（三）：CSS 样式及 DOM 属性
date: 2019-04-03 00:02:37
categories: "React"
comments: true
tags:
- React
---

<!-- no node -->

<!-- more -->

> 阅读源码成了今年的学习目标之一，在选择 Vue 和 React 之间，我想先阅读 React 。
> 在考虑到读哪个版本的时候，我想先接触到源码早期的思想可能会更轻松一些，最终我选择阅读 `0.3-stable` 。
> 那么接下来，我将从几个方面来解读这个版本的源码。

1. [React 源码学习（一）：HTML 元素渲染](https://zongzi531.com/2019/04/01/LSC-React-01/)
2. [React 源码学习（二）：HTML 子元素渲染](https://zongzi531.com/2019/04/02/LSC-React-02/)
3. [React 源码学习（三）：CSS 样式及 DOM 属性](https://zongzi531.com/2019/04/03/LSC-React-03/)
4. [React 源码学习（四）：事务机制](https://zongzi531.com/2019/04/04/LSC-React-04/)
5. [React 源码学习（五）：事件机制](https://zongzi531.com/2019/04/05/LSC-React-05/)
6. [React 源码学习（六）：组件渲染](https://zongzi531.com/2019/04/06/LSC-React-06/)
7. [React 源码学习（七）：生命周期](https://zongzi531.com/2019/04/07/LSC-React-07/)
8. [React 源码学习（八）：组件更新](https://zongzi531.com/2019/04/08/LSC-React-08/)

## 生成 Markup 标记时如何插入 CSS 样式及 DOM 属性

关于 CSS 样式及 DOM 属性，在之前有提到过在这里实现，并且把代码隐藏了，那么这次我们来进行解读：

```javascript
// core/ReactNativeComponent.js
var STYLE = keyOf({style: null});

ReactNativeComponent.Mixin = {
  _createOpenTagMarkup: function() {
    var props = this.props;
    var ret = this._tagOpen;

    for (var propKey in props) {
      if (!props.hasOwnProperty(propKey)) {
        continue;
      }
      var propValue = props[propKey];
      if (propValue == null) {
        continue;
      }
      // 注册事件相关，本次不做解读
      if (registrationNames[propKey]) {
        putListener(this._rootNodeID, propKey, propValue);
      } else {
        // CSS 样式处理
        if (propKey === STYLE) {
          if (propValue) {
            propValue = props.style = merge(props.style);
          }
          propValue = CSSPropertyOperations.createMarkupForStyles(propValue);
        }
        // 创建 DOM 属性 markup 标记
        var markup =
          DOMPropertyOperations.createMarkupForProperty(propKey, propValue);
        if (markup) {
          // 拼接
          ret += ' ' + markup;
        }
      }
    }

    return ret + ' id="' + this._rootNodeID + '">';
  },
}
```

```javascript
// vendor/core/keyOf.js
// 比如上面的 STYLE 返回的就是 'style'
var keyOf = function(oneKeyObj) {
  var key;
  for (key in oneKeyObj) {
    if (!oneKeyObj.hasOwnProperty(key)) {
      continue;
    }
    return key;
  }
  return null;
};
```

下面我们依次来解读 CSS 样式 和 DOM 属性。

### 合并方法 merge

先来看下这个 `merge` 方法：

```javascript
// utils/merge.js
var merge = function(one, two) {
  var result = {};
  mergeInto(result, one);
  mergeInto(result, two);
  return result;
};
```

这里需要注意， `mergeInto` 函数会对 `one` , `two` 参数进行校验。

校验他们是否为 `object` ，并且不是 `array` 。

```javascript
// utils/mergeInto.js
function mergeInto(one, two) {
  checkMergeObjectArg(one);
  if (two != null) {
    checkMergeObjectArg(two);
    for (var key in two) {
      if (!two.hasOwnProperty(key)) {
        continue;
      }
      one[key] = two[key];
    }
  }
}
```

```javascript
// utils/mergeHelpers.js
var isTerminal = function(o) {
  return typeof o !== 'object' || o === null;
};

var mergeHelpers = {
  checkMergeObjectArgs: function(one, two) {
    mergeHelpers.checkMergeObjectArg(one);
    mergeHelpers.checkMergeObjectArg(two);
  },
  checkMergeObjectArg: function(arg) {
    throwIf(isTerminal(arg) || Array.isArray(arg), ERRORS.MERGE_CORE_FAILURE);
  }
};
```

## 生成 CSS 样式

校验下 `props.style` 传入的是否是对象，然后创建 CSS markup 标记。我们来看到 `CSSPropertyOperations.createMarkupForStyles` 方法：

```javascript
// domUtils/CSSPropertyOperations.js
var processStyleName = memoizeStringOnly(function(styleName) {
  return escapeTextForBrowser(hyphenate(styleName));
});

var CSSPropertyOperations = {
  createMarkupForStyles: function(styles) {
    var serialized = '';
    // 遍历 styles
    for (var styleName in styles) {
      if (!styles.hasOwnProperty(styleName)) {
        continue;
      }
      var styleValue = styles[styleName];
      if (typeof styleValue !== 'undefined') {
        // 按照样式名和样式值组合拼接
        serialized += processStyleName(styleName) + ':';
        serialized += dangerousStyleValue(styleName, styleValue) + ';';
      }
    }
    return serialized;
  },
};
```

### 驼峰处理函数 hyphenate

```javascript
// vendor/core/hyphenate.js
// 驼峰转为“-”的形式，如：
// > hyphenate('backgroundColor')
// < "background-color"
var _uppercasePattern = /([A-Z])/g;

function hyphenate(string) {
  return string.replace(_uppercasePattern, '-$1').toLowerCase();
}
```

### 缓存函数

在声明 `processStyleName` 赋值后， `memoizeStringOnly` 就已经创建了一个 `cache` 用来缓存被驼峰转换过的值，若存在则直接被取出。

```javascript
// utils/memoizeStringOnly.js
function memoizeStringOnly(callback) {
  var cache = {};
  return function(string) {
    if (cache.hasOwnProperty(string)) {
      return cache[string];
    } else {
      return cache[string] = callback.call(this, string);
    }
  };
}
```

### CSS 样式值处理函数

那么到此， CSS 样式的键已经实现完成，接下来看看他的值是如何进行操作的，接下来看到 `dangerousStyleValue` ：

```javascript
// domUtils/dangerousStyleValue.js
function dangerousStyleValue(styleName, value) {
  if (value === null || value === false || value === true || value === '') {
    return '';
  }
  // value 不是数字的情况
  if (isNaN(value)) {
    return !value ? '' : '' + value;
  }
  // 满足 isNumber 的情况下，返回 value 本身，否则返回 + px
  return CSSProperty.isNumber[styleName] ? '' + value : (value + 'px');
}
```

```javascript
// domUtils/CSSProperty.js
var isNumber = {
  fillOpacity: true,
  fontWeight: true,
  opacity: true,
  orphans: true,
  textDecoration: true,
  zIndex: true,
  zoom: true
};

var CSSProperty = {
  isNumber: isNumber
};
```

## 生成 DOM Attribute 及 DOM Property 相关的操作对象

键和值处理完后，进行对应的拼接即可。接下来要做的就是创建 DOM 属性的 markup 标记。我们来看到 `DOMPropertyOperations.createMarkupForProperty` 方法：

因为 `DOMPropertyOperations` 中涉及到很多 `DOMProperty` 的方法，所以这里先解读 `DOMProperty` 。

```javascript
// domUtils/DOMProperty.js
var DOMProperty = {
  // 检查属性名称是否为标准属性。
  isStandardName: {},
  // 从规范化名称映射到不同的属性名称。 在渲染标记或使用 `*Attribute()` 时使用属性名称。
  getAttributeName: {},
  // 从规范化名称映射到DOM节点实例上的属性。（这包括因外部因素而发生变异的属性。）
  getPropertyName: {},
  // 从规范化名称映射到变异方法。 只有在不能通过属性或 `setAttribute()` 简单地设置变异时，才会存在这种情况。
  getMutationMethod: {},
  // 是否必须访问属性并将其变为对象属性。
  mustUseAttribute: {},
  // 是否必须使用 `*Attribute()` 访问和变异属性。 （这包括 `<propName> 中的 <element>` 失败的任何内容。）
  mustUseProperty: {},
  // 设置为falsey值时是否应删除该属性。
  hasBooleanValue: {},
  // 是否设置值会导致副作用，例如触发资源加载或文本选择更改。 我们必须确保只有在更改后才设置该值。
  hasSideEffects: {},
  // 检查属性名称是否为自定义属性。
  isCustomAttribute: RegExp.prototype.test.bind(
    /^(data|aria)-[a-z_][a-z\d_.\-]*$/
  )
};

// 从规范化的 camelcased 属性名称映射到指定应如何访问或呈现关联DOM属性的配置。
var MustUseAttribute  = 0x1;
var MustUseProperty   = 0x2;
var HasBooleanValue   = 0x4;
var HasSideEffects    = 0x8;

var Properties = {
  // 标准属性
  accept: null,
  ajaxify: MustUseAttribute,
  allowFullScreen: MustUseAttribute | HasBooleanValue,
  autoplay: HasBooleanValue,
  checked: MustUseProperty | HasBooleanValue,
  className: MustUseProperty,
  value: MustUseProperty | HasSideEffects,
  // ...
};

// 未指定的属性名称使用 **小写** 规范化名称。
var DOMAttributeNames = {
  className: 'class',
  htmlFor: 'for',
  // ...
};

// 未指定的属性名称使用规范化名称。
var DOMPropertyNames = {
  autoComplete: 'autocomplete',
  spellCheck: 'spellcheck'
};

// 需要特殊变异方法的属性。
var DOMMutationMethods = {
  className: function(node, value) {
    node.className = value || '';
  }
};

// 遍历标准属性
for (var propName in Properties) {
  // isStandardName 对应的标准属性设置为 true
  DOMProperty.isStandardName[propName] = true;

  // 标准属性的映射，对应 DOMAttributeNames 进行操作
  DOMProperty.getAttributeName[propName] =
    DOMAttributeNames[propName] || propName.toLowerCase();

  // 同样的，标准属性的映射，对应 DOMPropertyNames 进行操作
  DOMProperty.getPropertyName[propName] =
    DOMPropertyNames[propName] || propName;

  // 检查特殊编译方法是否存在
  var mutationMethod = DOMMutationMethods[propName];
  if (mutationMethod) {
    // 赋值对应方法
    DOMProperty.getMutationMethod[propName] = mutationMethod;
  }

  // 获取标准属性的值
  var propConfig = Properties[propName];
  // 必须使用属性
  DOMProperty.mustUseAttribute[propName] = propConfig & MustUseAttribute;
  DOMProperty.mustUseProperty[propName]  = propConfig & MustUseProperty;
  // 具有布尔值
  DOMProperty.hasBooleanValue[propName]  = propConfig & HasBooleanValue;
  // 具有副作用
  DOMProperty.hasSideEffects[propName]   = propConfig & HasSideEffects;
}
```

### 生成 DOM Attribute 及 DOM Property 的 Markup 标记

```javascript
// domUtils/DOMPropertyOperations.js
// 同样这里也用到了 DOM 属性“键”的缓存
var processAttributeNameAndPrefix = memoizeStringOnly(function(name) {
  return escapeTextForBrowser(name) + '="';
});

var DOMPropertyOperations = {
  createMarkupForProperty: function(name, value) {
    // 首先检查 name 是否为标准属性
    if (DOMProperty.isStandardName[name]) {
      // 判断 name 是否具有布尔值，并且 value 通过隐式转换为 false 的情况
      if (value == null || DOMProperty.hasBooleanValue[name] && !value) {
        return '';
      }
      // 获取属性映射的对应 attributeName
      var attributeName = DOMProperty.getAttributeName[name];
      // 进行 markup 拼接处理
      return processAttributeNameAndPrefix(attributeName) +
        escapeTextForBrowser(value) + '"';
      // 若 name 为自定义样式，如： data- 这样的
    } else if (DOMProperty.isCustomAttribute(name)) {
      if (value == null) {
        return '';
      }
      // 同样的 markup 拼接处理
      return processAttributeNameAndPrefix(name) +
        escapeTextForBrowser(value) + '"';
    } else {
      // 其他情况
      return null;
    }
  }
};
```

那么到此，实现 CSS 样式及 DOM 属性。
