# ProseMirror for dummies
This repo used to be a simple cookbook but I decided to convert this repo into "ProseMirror for dummies". In part because I wil most likely forget all the info written here, and in part because I'd like to help you, dear reader, in your struggle to use this library which is not for the faint of heart.

This is a work in progress. If you have any suggestion, please do not hesitate to create an issue or a PR.

Also, check the [ProseMirror Utils](https://github.com/atlassian/prosemirror-utils) repo by Atlassian. Not only it is useful *per se*, but the source code offers a lot of information on how to to certain things.

## Basics

#### Triggering changes from the keyboard
By default, a barebones editor will trigger text changes on a `contentEditable` element and no more. That is, move the cursor forwards and backwards, insert text, delete, etc. It won't create new paragraphs when pressing enter since ProseMirror is totally agnostic and this behavior is something that needs to be configured.

The `prosemirror-keymap` module allows us to hookup on keyboard events triggered by the `contentEditable` elements. This plugin by itself will not trigger changes either.

For example, we can configure a `keymap()` to trigger the `undo` and `redo` functions from the `prosemirror-history` module which would in turn interact with the editor and trigger those changes.

Common keyboard commands (eg: creating a new paragraph when pressing enter, etc) are configured in the `baseKeymap` config of the `prosemirror-commands` module.

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

These packages only exist for our convenience and are not part of the core. We can replace them with our own later on.

There are other commands in `prosemirror-commands` which can be used to trigger changes. Here are some common ones:

* `toggleMark` which allows you to toggle inline marks on/off on the selected text (eg: bold, italics, etc).
* `setBlockType` which allows you to change the type of the current block (eg: paragraph, H1, image, etc).

All the available commands are documented here: [https://prosemirror.net/docs/ref/#commands](https://prosemirror.net/docs/ref/#commands)

#### Triggering changes from your code
To dispatch changes we need to get the current state and the use the `dispatch` function from our editor view and call methods on `state.tr`which is an instance of the [Transaction](https://prosemirror.net/docs/ref/#state.Transaction) class.

For example, if we wanted to delete the current selction, we could do something like this:

```js
function deleteSelection (state, dispatch) {
  if (state.selection.empty) return false;
  if (dispatch) dispatch(state.tr.deleteSelection())
  return true;
}

deleteSelection(state, dispatch);
```

It's a convention in ProseMirror that most commands can be tested without making any changes to know if it can be executed. To do that we just call the function without passing `dispatch`. In this case we'd return `true` without deleting anything.

Again, you could call this `deleteSelection` command from a keyboard shortcut using the `prosemirror-keymap` as explained earlier, or from your own logic. Commands are explained in more detail here: [https://prosemirror.net/docs/guide/#commands](https://prosemirror.net/docs/guide/#commands).

Check the [custom menu example](https://prosemirror.net/examples/menu/) for more info on triggering commands from your UI.

#### About `chainCommands`

Modularity and agnosticism are taken to the extreme in ProseMirror which is one of the reasons it's so flexible. Once you start digging into how comands work you'll see every little behavior needs its own command. In consequence, you will need the same keyboard shortcut to trigger a number of commands that can cancel each other out. This is done using the `chainCommands` function.

If you take a look at the `prosemirror-commands` package you will see this example for the `Enter` key;
```
"Enter": chainCommands(newlineInCode, createParagraphNear, liftEmptyBlock, splitBlock),
```
How this works is that `chainCommands` will trigger the commands in order until one command returns true and stop there.

For example, the first command `newlineInCode` would test if it can be triggered (eg: if it's in a code block) and return true after it has disptached a transaction. If not, it would return `false` or `undefined` and `chainCommands()` would go on to `createParagraphNear` and so on.

If no command returns `true` then `chainCommands` will return false and ProseMirror will continue looking for matches for the triggered key in other `keymap` plugin instances.

Finally, if no configured shortcut returns `true`, ProseMirror will allow the keyboard event to execute in the `contentEditable` element.

I found `chainCommands` to be very confusing at first, so I created my own version to show the name of the function that was actually executed:

```js
export function chainCommands(...commands) {
  return function(state, dispatch, view) {
    console.log('chainCommands');

    for (let i = 0; i < commands.length; i++) {
      const result = commands[i](state, dispatch, view);
      // If the command has executed `result` will be true
      if (result) {
      	// On creation, some commands create a closure and return
      	// an anonymous function so we can't really know their name...
        console.log(commands[i].name || 'anonymous function');
        return true;
      }
    }

    console.log('no command matched!');
    return false
  }
}
```

#### How to move the cursor

At some point you will need to start making your own commands. A very common need is to move the cursor somewhere.

This is how you would do it from inside a command (with access to a `dispatch` function and `state` object) once you have figured out the position where you want your cursor to go:

```js
dispatch(state.tr.setSelection(TextSelection.create(state.doc, 345)))
```

The idea here is that we're defining an empty [`TextSelection`](https://prosemirror.net/docs/ref/#state.TextSelection) with a single absolute position in the document. It's called an empty selection because both the start and end cursor of the selection would be the same. In the ProseMirror docs these two cursors are referred to as `anchor` and `head`.

#### Get an HTML string from an editor state
```js
import {DOMSerializer} from 'prosemirror-model';

function getHTMLStringFromState (state) {
  const fragment = DOMSerializer.fromSchema(state.schema).serializeFragment(state.doc.content);
  const div = document.createElement("div");
  div.appendChild(fragment);
  return div.innerHTML;
}
```
## Editor

#### Get notified of updates and changes
Here's a simple plugin to know when something has changed in the editor (cursor position, selection, etc) and hook up on the updates:
```js
import {EditorState, Plugin} from "prosemirror-state";

const onUpdatePlugin = new Plugin({
  view () {
    return {
      update (updatedEditorView) {
      	// For example, let's print the cursor position:
        const $cursor = updatedEditorView.state.selection.$cursor;
        console.log('cursor position:', $cursor.pos);
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
This is a helper function that returns an array with the active marks in the current selection. This is helpful when we need to highlight the buttons of the marks that are applied (bold, italic, etc):
```js
function getCurrentMarks (editorView) {
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
This is a helper function that returns an array with the node types that can be applied in the current selection or cursor position. This is helpful, for example, if we want to enable/disable/highlight buttons to switch from heading to paragraph.
```js
import {setBlockType} from 'prosemirror-commands';

function getAvailableBlockTypes (editorView, schema) {
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
Here's a plugin that automatically applies a decoration to the node where the cursor is by adding `current-element` CSS class to the element:
```js
new Plugin({
  props: {
    decorations(state) {
      const selection = state.selection;
      const resolved = state.doc.resolve(selection.from);
      const decoration = Decoration.node(resolved.before(), resolved.after(), {class: 'current-element'});
      // This is equivalent to:
      // const decoration = Decoration.node(resolved.start() - 1, resolved.end() + 1, {class: 'current-element'});
      return DecorationSet.create(state.doc, [decoration]);
    }
  }
})
```
#### Apply a decoration to the selected node(s)
Here's a plugin that automatically applies a decoration to all the nodes that are touched by a selection. In this case we're adding the `selected` CSS class:
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