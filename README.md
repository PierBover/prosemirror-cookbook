# prosemirror-cookbook
A series of short examples for understanding ProseMirror.

## Decorations
A plugin for applying a decoration to the node where the cursor is:
```js
new Plugin({
  props: {
    decorations(state) {
      const selection = state.selection;
      const resolved = state.doc.resolve(selection.from);
      const decoration = Decoration.node(resolved.before(), resolved.after(), {class: 'selected'});
      // equivalent to
      // const decoration = Decoration.node(resolved.start() - 1, resolved.end() + 1, {class: 'selected'});
      return DecorationSet.create(state.doc, [decoration]);
    }
  }
})
```
A plugin for applying a decoration to the all the nodes that are "touched" by a selection:
```js
new Plugin({
  props: {
    decorations(state) {
      const selection = state.selection;
      const decorations = [];

      state.doc.nodesBetween(selection.from, selection.to, (node, position) => {
        if (node.isBlock) {
          decorations.push(Decoration.node(position, position + node.nodeSize, {class: 'selected'}));
        }
      });

      return DecorationSet.create(state.doc, decorations);
    }
  }
})
```
