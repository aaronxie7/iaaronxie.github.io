---
title: 谈谈 Event Emitter
date: 2017-11-24 23:56:14
tags: Vue.js
---
[Vue.js](https://cn.vuejs.org/) 2.0 把 1.0 中的 `$dispatch` 和 `$broadcast` 删除了，并给出了下面的理由：

> 因为基于组件树结构的事件流方式实在是让人难以理解，并且在组件结构扩展的过程中会变得越来越脆弱。这种事件> 方式确实不太好，我们也不希望在以后让开发者们太痛苦。并且 `$dispatch` 和 `$broadcast` 也没有解决> 兄弟组件间的通信问题。

并且，Vue.js 本身推荐在复杂情况下应该使用  [Vuex](https://vuex.vuejs.org/)  来管理状态，我觉得这句话应该有一个前提，那就是你的组件本身与业务逻辑关联性很强，换句话说，就是这组件不适合迁移到别的项目或者共享给其他人。之所以要用到 Vuex 最主要的原因是因为项目已经拓展到一定规模，状态已经非常乱了，使用 Vuex 可以让状态管理清晰起来。如果项目本身是一个小官网，你想使用一个开源组件，但是这个组件却要求你加载 Vuex，这明显不合理。



`$dispatch` 和 `$broadcast` 本身并非一无是处，虽然在 Vue.js 2.0 废除了，但是其实很多开源框架中仍然有使用，比如 [iView](https://www.iviewui.com/)、[ElementUI](http://element.eleme.io/#/zh-CN) 等开源 UI 框架。



我们先看回 Vue.js。在 Vue.js 中，父子组件间交互的方式是：父组件将数据传递给子组件的方式是通过 `props` ，这里数据的传递是单向的，父组件传递给子组件的那些 `props` 改变了，那么，子组件能够实时监测到。而子组件想要把数据传递给父组件的话，得通过触发事件的方式来传递。这是父子组件间传递数据的方式，兄弟组件间传递数据的方式可以定义一个空的 Vue.js 实例，通过这个空实例，可以达到兄弟组件间数据的传递。兄弟、父子组件有了，祖父、孙子组件呢？祖父、孙子组件没办法跟父子组件传递数据的方式一样一层层传递下去，只能跟兄弟组件一样定义一个空实例来传递数据。



在使用开源组件过程中，一般我们以下面的方式调用组件：
```
  <component-name
    :foo="true"
    @bar="bar">
  </component-name>
```
如果这个开源组件因为本身的原因，内部有一个孙子组件，`bar` 事件刚好作为孙子组件的单击事件，这时候应该怎么做？其实这个时候可以在孙子组件里面以 `$parent` 变量层层调用到 `<component-name>`，然后再触发事件。这里我们有一个前提，这是个开源组件，也就是说孙子组件我们是没法控制的，所以以空实例来作为事件总线有点不现实。



上面我们说到的  “以 `$parent` 变量层层调用到 `<component-name>`，然后再触发事件”  其实就是 `dispatch` 而相反以 `$children` 层层调用到特定子元素，则是 `broadcast`。我们可以看下下面这个是 ElementUI 本身对这两种操作的实现：
```
    function broadcast(componentName, eventName, params) {
      this.$children.forEach(child => {
        var name = child.$options.componentName;
        
        if (name === componentName) {
          child.$emit.apply(child, [eventName].concat(params));
        } else {
          broadcast.apply(child, [componentName, eventName].concat([params]));
        }
      });
    }
    export default {
      methods: {
        dispatch(componentName, eventName, params) {
          var parent = this.$parent || this.$root;
          var name = parent.$options.componentName;
    
          while (parent && (!name || name !== componentName)) {
            parent = parent.$parent;
            
            if (parent) {
              name = parent.$options.componentName;
            }
          }
          if (parent) {
            parent.$emit.apply(parent, [eventName].concat(params));
          }
        },
        broadcast(componentName, eventName, params) {
          broadcast.call(this, componentName, eventName, params);
        }
      }
    };
```
上面这段代码需要修改点代码才能用在别的地方，原因在于这句代码 `$options.componentName` 正常的 Vue.js 组件是不会有 `componentName` 这个字段的，你可以换成 `name` 字段。上面这段代码其实并不难，基本的逻辑都是一层层找到特定的组件，然后使用 `$emit` 来触发事件，并把参数传递过去。



最后，我们再谈谈为什么 Vue.js 要废除 `$dispatch` 和 `$broadcast`，对于业务组件在说，Vue.js 建议的是使用 Vuex，但是对于那些开源的组件呢？其实正如官方建议所说的，在组件拓展过程中使用 `$dispatch` 和 `$broadcast`  确实让代码变得脆弱，哪里都可以去触发事件，会让组件的清晰程度变低。究其深层意思，其实 Vue.js 更希望开源组件能以下面这种形式来调用：
```
    <grand-parent>
      <parent>
        <child></child>
      </parent>
    </grand-parent>
```
以上面形式调用就不用担心孙子组件怎么去触发祖父组件的事件，代码也非常地清晰。

完。
