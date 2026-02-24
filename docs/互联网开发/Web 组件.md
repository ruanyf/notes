# Web 组件

## 概述

Web 组件的技术组成

- Shadow DOM 提供封装机制，使组件的样式和结构保持独立。
- 模板和插槽提供了组合模式。
- 原生事件系统为组件提供了一种强大的通信机制，避免了组件间的紧密耦合。

## 自定义元素

自定义元素的写法。

```javascript
class TaskCard extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <div class="task">
        <h3>${this.getAttribute('title')}</h3>
        <p>${this.getAttribute('description')}</p>
      </div>
    `;
  }
}
customElements.define('task-card', TaskCard);
```

## Shadow DOM 封装

Shadow DOM 封装的优点是，组件的样式不会泄露出去，全局样式也不会泄露进来（除非你明确允许）。

```javascript
class StyledCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        .card { padding: 1rem; border-radius: 8px; background: #f5f5f5; }
        h3 { margin: 0 0 0.5rem 0; color: #333; }
      </style>
      <div class="card">
        <h3><slot name="title"></slot></h3>
        <slot></slot>
      </div>
    `;
  }
}
```

## 通信

通过事件向上通信。

```javascript
this.dispatchEvent(new CustomEvent('item-selected', {
  detail: { itemId: this.selectedId, metadata: this.itemData },
  bubbles: true,
  composed: true
}));
```

向下通信：父组件更改子组件的某个属性时，子组件的`attributeChangedCallback`监听函数会触发。对于更复杂的数据，属性允许直接传递对象和数组。