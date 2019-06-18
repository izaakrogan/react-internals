## Notes on React internals

React programmes output virtual DOM tree.

Think of virtual DOM is an object, it is.

ReactDOM.render builds a virtual DOM tree that will determine the next state of the DOM.

You can read ReactDOM.render(<App />, domContainer) as: “React, make the domContainer DOM tree match my virtual DOM object (App).”

React's first job is to build the virtual DOM tree.

React's second job is to make the real DOM tree match the provided virtual DOM tree. The process of figuring out the set of changes that are required to get the real DOM tree to match the new virtual DOM tree is called reconciliation.

Simplified virtual DOM tree:

```js
{
  type: 'dialog',
  props: {
    children: [{
      type: 'button',
      props: { className: 'blue' }
    }, {
      type: 'button',
      props: { className: 'red' }
    }]
  }
}
```

On each render, React will produce one entirely new object that represents the entire app.

React will then traverse both the old and new Virtual DOM trees and create a "diff" i.e. a set of differences between the trees. This diff will produce a minimal set of calls to the DOM API to make the host match the new virtual DOM tree. E.g.

```js
let inputNode = dialogNode.firstChild;
let pNode = document.createElement("p");
pNode.textContent = "I was just added here!";
dialogNode.insertBefore(pNode, inputNode);
```

React does this efficiently by relying on the fact that old virtual DOM tree and new virtual DOM tree have a similar structure. e.g.

```js
///Old tree
{
  type: 'dialog',
  props: {
    children: [
      {
        type: ‘null’,
      },
      {
        type: 'button',
        props: { className: 'red' }
      }
     ]
  }
}

///New tree
{
  type: 'dialog',
  props: {
    children: [
      {
        type: 'button',
        props: { className: 'blue' }
      }, 
      {
        type: 'button',
        props: { className: 'red' }
      },
    ]
  }
}
```

Notice the `null` value in the old tree. Remember that a) [components are not allowed to return adjacent react elements](https://stackoverflow.com/questions/31284169/parse-error-adjacent-jsx-elements-must-be-wrapped-in-an-enclosing-tag) and b) conditional renders will always return a single child or null. Therefore any React element will always have the same number of children\*. In this sense, the structure of the tree does not change. The one-to-one correspondence between elements in the old and new virtual DOM tree allows React to compare one element to another by simply traversing both trees and checking each element. When doing this traversal, React diffs the old and new tree using the `type`.

By checking the `type`, React can determine whether or not to ignore, replace or mutate a real DOM element. This works in the following way:

- If the `type` has changed React will replace the real DOM element with a new DOM element.
- If the `type` is the same React will check to see if any properties have changed.
  - If not, React will ignore the DOM element.
  - If the properties have changed, React will mutate the DOM element so that it reflects the new virtual DOM tree.

e.g. let’s say React is traversing the new and old Virtual DOM tree and finds that the first child of ContainerComponent has the same type but has been given the new className ‘pink’. In this case our diffing algorithm will output something like

let someDomNode = ContainerComponent.firstChild;
someDomNode.className = ‘pink’;

Notice that the code above are calls the real DOM API.

So simply put the the process is:

- the app state changes (e.g. call setState)
- A new virtual DOM tree is created
- React loops through the old virtual DOM tree and the new virtual DOM tree to see what’s been changed
- The changes will determine a set of real DOM api calls
- These calls will change the DOM to match the new virtual DOM tree and update our UI as expected

It’s worth noting that the real DOM api calls only get passed to the DOM once the traversal is complete and the diffing algorithm has produced a full set of required DOM manipulations. So React will batch the updates and performs these batched updates synchronously. It will only every send a full set of instructions required to match the real DOM tree with the new virtual DOM tree. This is to avoid our user seeing a partially mutated DOM.

When diffing old and new trees, React uses the `type` to determine whether or not to create, replace or mutate a DOM element. This can cause issues in a list. If you have a list of elements of the same type, and we reorder that list, React may try to mutate the List elements. Think about this example:

- Given a list where every item in the list has an input box
- We type ‘hello world’ into the input box in the first item
- The list is reordered and the first item in the list is moved to the second position
- React diffs the old and new virtual DOM trees as described above
- Unfortunately the text ‘hello world’ has not moved with the item.
- The new first item will have the text ‘hello world’ and the old first item will not.

The solution to this is to add a key to each list item. Doing so gives React enough information to associate corresponding list items in old and new virtual DOM trees.

\*Clarification: Clearly a new react element can have any number of children so the structure can in-fact change dramatically. What I really mean is that, if a React element exists in a virtual DOM tree and, given an update, it still exists in the new virtual DOM tree, that element will always have the same number of children. Also note that lists can break the pattern of consistent number of children for a given parent i.e. if we change the number of items in a list we have changed the number of children. That’s why we have to pass a key to each child.
