# ProseMirror Cookbook
A series of short examples for understanding ProseMirror.

Don't hesitate to contribute to the repo!

## General concepts

#### Triggering changes from the keyboard
By default, a barebones editor will trigger text changes on a `contentEditable` element and no more. It won't create new paragraphs when pressing enter since ProseMirror is totally agnostic and this behavior is something that needs to be configured.

The `prosemirror-keymap` plugin will allow you to hookup on keyboard events triggered by the editor. This plugin by itself will not trigger changes either.

For example, you can configure `prosemirror-keymap` to trigger the `undo` and `redo` functions from the `prosemirror-history` plugin which would in turn interact with the editor and trigger those changes. Common keyboard commands (eg: creating a new paragraph when pressing enter, etc) can be found in the `baseKeymap` module of the `prosemirror-commands` plugin.

```js
import {undo, redo, history} from "prosemirror-history";
import {keymap} from "prosemirror-keymap";
import {baseKeymap} from "prosemirror-commands";


let state = EditorState.create({
  schema,
  plugins: [
    history(),
    keymap({"Mod-z": undo, "Mod-y": redo}),
    keymap(baseKeymap)
  ]
});
```

These packages exist for your convenience and are not part of the core. You can replace them with your own once you are more comfortable with ProseMirror.

There are other commands in `prosemirror-commands` which can be used to trigger changes. Here are some common ones:

* `toggleMark` which allows you to toggle inline marks on/off on the selected text (eg: bold, italics, etc).
* `setBlockType` which allows you to change the type of the current block (eg: paragraph, H1, image, etc).

All the available commands are documented here: [https://prosemirror.net/docs/ref/#commands](https://prosemirror.net/docs/ref/#commands)

#### Triggering changes from your code
To dispatch changes you basically need to get the current state and the `dispatch` function from your editor view and call methods on `state.tr`. `tr` is an instance of the [Transaction](https://prosemirror.net/docs/ref/#state.Transaction) class.

```js
deleteSelection (view) {
  const state = view.state;
  const dispatch = view.dispatch;
  if (state.selection.empty) return false;
  if (dispatch) dispatch(state.tr.deleteSelection())
  return true;
}

deleteSelection(editorView);
```

Again, you could call this command we've just created from a keyboard shortcut using the `prosemirror-keymap` as explained earlier, or from your logic. Commands are explained in more detail here: [https://prosemirror.net/docs/guide/#commands](https://prosemirror.net/docs/guide/#commands).

Also check the [custom menu example](https://prosemirror.net/examples/menu/) for more info on triggering commands from your UI.

## Editor
#### Get notified of updates and changes
A plugin to know when something has changed in the editor (cursor position, selection, etc) and hook up on the updates:
```js
import {EditorState, Plugin} from "prosemirror-state";

const onUpdatePlugin = new Plugin({
  view () {
    return {
      update (updatedEditorView) {
        // do something here with updatedEditorView
      }
    }
  }
});

const editorState = EditorState.create({
  schema,
  plugins: [onUpdatePlugin]
});
```
You can also hookup on the transactions:
```js
const editorView = new EditorView(editorElement, {
  state: editorState,
  dispatchTransaction (transaction) {
    const newState = editorView.state.apply(transaction);
    editorView.updateState(newState);
    // do something here such as eventBus.dispatch('NEW-TRANSACTIION', transaction)
  }
});
```

## Commands
#### Check the current active marks
A helper function that returns an array with the active marks in the current selection. This is helpful if you have a custom menu and want to highlight the buttons of the marks that are applied (bold, italic, etc):
```js
getCurrentMarks (editorView) {
  const state = editorView.state;
  const from = state.selection.from;
  const to = state.selection.to;

  // Set instead of array because there can't be duplicated values
  const selectionMarks = new Set();

  state.doc.nodesBetween(from, to, (node) => {
    node.marks.forEach((mark) => {
      selectionMarks.add(mark.type.name);
    });
  });

  return Array.from(selectionMarks);
}
```
#### Check the current available node types
A helper function that returns an array with the node types that can be applied in the current selection or cursor position. This is helpful for example if you have a custom menu and want to enable/disable/highlight buttons to switch from heading to paragraph.
```js
getAvailableBlockTypes (editorView, schema) {
  // get all the available nodeTypes in the schema
  const nodeTypes = schema.nodes;

  // iterate all the nodeTypes and check which ones can be applied
  return Object.keys(nodeTypes).filter((key) => {
    const nodeType = nodeTypes[key];
    // setBlockType() returns a function which returns false when a node can't be applied
    return setBlockType(nodeType)(this.editorView.state, null, this.editorView);
  });
}
```

## Decorations

#### Apply a decoration where the cursor is
A plugin that applies a decoration to the node where the cursor is:
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
#### Apply a decoration to the selected node(s)
A plugin for applying a decoration to all the nodes that are "touched" by a selection:
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
