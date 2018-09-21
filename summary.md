# Imperative Shadow DOM Distribution API

kymuto@google.com

With Imperative slotting API, web developers can specify node-to-slot assignments inperatively;They don't have to put slot="" attributes everywhere, getting much cleaner markup and flexibility.

# Imperative Slotting API

Web developers have to show their *opt-in* to use imperative API for each shadow tree.

A shadow root has an associated *slotting*. Web developers can set shadow root's *slotting\
* to *manual* by specifying it in attachShadow:

```js
const sr = attachShadow({ mode: ..., (optional) slotting: 'manual' })
```

The *manual* means "we support only imperative APIs for the shadow tree".
The default is "we support only declarative API for the shadow tree".

In addition to [assigned nodes], which is already defined in DOM Standard,
a slot has an associated *manually-assigned-nodes* (ordered list). Unless stated otherwise, it is empty.

[assigned nodes]: https://dom.spec.whatwg.org/#slot-assigned-nodes

A slot gets new API, called *assign* (tentative name).

See also the Example for details.

# Examples

## Example 1: How imperative slotting API works in slotting=manual.

``` text
host
├──/shadowroot (slotting=manual)
│   ├── slot1
│   └── slot2
├── A
└── B
```

``` javascript
// '==' means ArrayEquals.
assert(slot1.assignedNodes() == []);
assert(slot2.assignedNodes() == []);

slot2.assign([A]);

assert(slot2.assignedNodes() == [A]);

slot2.assign([B, A]);  // The order doesn't matter.

assert(slot2.assignedNodes() == [A, B]);

slot1.assign(A);

assert(slot1.assignedNodes() == [A]);  // slot1 got A.
assert(slot2.assignedNodes() == [B]);  // slot2 lost A.

slot1.assign([A, A, A, host]);   // We don't throw an exepction here.

assert(slot1.assignedNodes() == [A]);
assert(slot2.assignedNodes() == [B]);

slot1.assign([]);

assert(slot1.assignedNodes() == []);
assert(slot2.assignedNodes() == [A, B]);

slot2.assign([]);

assert(slot1.assignedNodes() == []);
assert(slot2.assignedNodes() == []);

```

## Example 2: Imperative slotting API doesn't have any effect in a shadow root with slotting=auto.


``` text

host
├──/shadowroot (slotting=auto) (default)
│   ├── slot1 name=slot1
│   └── slot2 name=slot2
├── A slot=slot1
└── B slot=slot2
```

``` javascript

assert(slot1.assignedNodes() == [A]);
assert(slot2.assignedNodes() == [B]);

slot1.assign([A, B]);  // This doesn't have any effect because this shadow tree's slotting is auto

assert(slot1.assignedNodes() == [A]);
assert(slot2.assignedNodes() == [B]);

```


## Example 3: Inappropriate nodes are ignored, and doesn't appear in slot.assignedNodes()


``` text

host1
├──/shadowroot1 (slotting=manual)
│   └── slot1
└── A

host2
├──/shadowroot2 (slotting=manual)
│   └── slot2
└── B
    └── C

```

``` javascript

slot2.assign([A, B, C]);
assert(slot2.assignedNodes() == [B]); // A is excluded here because A is other shadow tree's host's child.

slot1.assign([A]);
assert(slot1.assignedNodes() == [A]);

shadowroot2.append(slot1);
assert(slot1.assignedNodes() == []);  // A is no longer slot1's shadow tree's child.

shadowroot1.append(slot1);
assert(slot1.assignedNodes() == [A]); // Now A is slot1's shadow tree's child.

```


## Example 4: A node can be appended to a host after the node is imperatively assigned to a slot

``` text

host
└──/shadowroot (slotting=manual)
    └── slot1
```

``` javascript

assert(slot1.assignedNodes() == []);

const a = document.createElement('div');
const b = document.createElement('div');
slot1.assign([a, b]);   // We don't throw an exception

assert(slot1.assignedNodes() == []);   // Neither A nor B is slot1's shadow tree's host's child

host.append(a);
assert(slot1.assignedNodes() == [a]);

host.append(b);
assert(slot1.assignedNodes() == [a, b]);
```

# Create custom element

Web developers could use *assign* in stead of specifying slot= attribute for every shadow host's children

Example:
As the built-in element there is details/summary, developers become to implement this element by themselves.

```js

class MySummaryElement extends HTMLElement {
  constructor() {
    super();
  }
}
customElements.define("my-summary", MySummaryElement);

customElements.define("my-detail", class extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open", slotting: "manual" });
  }
  connectedCallback() {
    const target = this;
    if (!target.shadowRoot.querySelector(':scope > slot')) {
      const slot1 = document.createElement("slot");
      const slot2 = document.createElement("slot");
      const shadowRoot = target.shadowRoot;
      shadowRoot.appendChild(slot1);
      shadowRoot.appendChild(slot2);
      slot1.style.display = "block";
      const observer = new MutationObserver(function(mutations) {
        //Get the first <my-summary> element from <my-detail>'s direct children
        const my_summary = target.querySelector(':scope > my-summary');
        if (my_summary) {
          slot1.assign([my_summary]);
        } else {
          slot1.assign([]);
        }
        slot2.assign(target.childNodes);
        slot2.style.display = "none";
      });
    observer.observe(this, {childList: true});
    }
  }
});
```

```html
<my-details>
  <div>div</div>
  <my-summary>summary1</my-summary>
  <my-summary>summary2</my-summary>
</my-details>
```
