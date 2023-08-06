---
title: 'AceEditor开发的坑'
date: 2020-09-20 14:01:33
tags:
- 'Javascript'
- 'ace-editor'
categories:
- '前端'
---

## 禁止首行编辑
使用到ace实现一个首行禁止编辑的功能，我们知道编辑操作可以这些：插入、删除、粘贴。只要阻止第一行的相关操作就可以了

但是，中文输入法输入会导致一些奇奇怪怪的问题。**所以我们另外对光标进行处理，只要光标点击第一行我们就给他跳转到第二行**

show me the code...
```Javascript
//光标跳转
this.ace.getSession().selection.on('changeCursor', (e)=>{
  if(this.ace.getSelectionRange().start.row == 0 || this.ace.getSelectionRange().end.row == 0){
    this.ace.gotoLine(2);
  }
});
//不能对第一行进行粘贴、删除操作
this.ace.commands.on("exec", (e)=>{
    const hasFirst = this.ace.getSelectionRange().start.row == 0 ;
    const command = e.command.description;
    if ((hasFirst && ( command == "Paste" || command == "Backspace"))) {
      e.preventDefault()
      e.stopPropagation()
    }
});
```

以上方法能解决大部分问题，但是**移动文本能成功修改**（2020-11-21）

下面是`stackoverflow`上面的一个相对来说比较完美解决办法（[传送门](https://stackoverflow.com/questions/39640328/how-to-make-multiple-chunk-of-lines-readonly-in-ace-editor/39640987#39640987)）

重点代码是对原有编辑器方法的拦截修改

我进行了一些修改和注释：

```js
/**
 * Ace Editor 锁定或只读行封装
 * @author CHANX 2020-11-21
 * @param {ace} editor ace编辑器实例
 * @param {number[]} [readOnlyLines = []] 需要锁定的行号
 */

function setReadOnly(editor, readOnlyLines) {
  const session = editor.session;
  const Range = editor;
  const readOnlyRanges = [];
  for (let i = 0; i < readOnlyLines.length; i += 1) {
    const newRange = [readOnlyLines[i] - 1, 0, readOnlyLines[i], 0];
    // 此处原有代码range是当前行头到下一行头，我修改为当前行头到当前行1000位置
    // 防止下一行头无法被编辑
    readOnlyRanges.push({
      start: {
        row: readOnlyLines[i] - 1,
        column: 0,
      },
      end: {
        row: readOnlyLines[i] - 1,
        column: 1000,
      },
    });
  }
  function before(obj, method, wrapper) {
    const orig = obj[method];
    obj[method] = function (...arg) {
      const args = Array.prototype.slice.call(arg);
      return wrapper.call(this, () => orig.apply(obj, args), args);
    };
    return obj[method];
  }

  /**
   * 当前选中范围和传入范围是否冲突
   * @param {*} range
   */
  function intersects(range) {
    return editor.getSelectionRange().intersects(range);
  }
  function preventReadonly(next, args) {
    for (let i = 0; i < readOnlyRanges.length; i += 1) { if (intersects(readOnlyRanges[i])) return; }
    next();
  }
  /**
   * 当前选中范围和不可编辑行范围是否冲突
   * @param {*} newRange
   */
  function intersectsRange(newRange) {
    for (let i = 0; i < readOnlyRanges.length; i += 1) { if (newRange.intersects(readOnlyRanges[i])) return true; }
    return false;
  }
  /**
   * 是否在不可编辑行的末尾
   * @param {*} position
   */
  function onEnd(position) {
    const row = position.row;
    const column = position.column;
    for (let i = 0; i < readOnlyRanges.length; i += 1) { if (readOnlyRanges[i].end.row === row && readOnlyRanges[i].end.column === column) return true; }
    return false;
  }
  /**
   * 是否在不可编辑行的范围外
   * @param {*} position
   */
  function outSideRange(position) {
    const row = position.row;
    const column = position.column;
    for (let i = 0; i < readOnlyRanges.length; i += 1) {
      if (readOnlyRanges[i].start.row < row && readOnlyRanges[i].end.row > row) { return false; }
      if (readOnlyRanges[i].start.row === row && readOnlyRanges[i].start.column < column) {
        if (readOnlyRanges[i].end.row !== row || readOnlyRanges[i].end.column > column) { return false; }
      } else if (readOnlyRanges[i].end.row === row && readOnlyRanges[i].end.column > column) {
        return false;
      }
    }
    return true;
  }
  editor.keyBinding.addKeyboardHandler({
    handleKeyboard(data, hash, keyString, keyCode, event) {
      // 阻止不可编辑行行尾的回车
      if (Math.abs(keyCode) === 13 && onEnd(editor.getCursorPosition())) {
        return false;
      }
      if (hash === -1 || (keyCode <= 40 && keyCode >= 37)) return false;
      for (let i = 0; i < readOnlyRanges.length; i += 1) {
        if (intersects(readOnlyRanges[i])) {
          return { command: "null", passEvent: false };
        }
      }
    },
  });
  before(editor, "onPaste", preventReadonly);
  before(editor, "onCut", preventReadonly);

  const old$tryReplace = editor.$tryReplace;
  editor.$tryReplace = function (range, replacement) {
    return intersectsRange(range) ? null : old$tryReplace.apply(this, arguments);
  };

  /**
   * 重写insert方法： 若postion在不可编辑范围内则不可插入
   */
  const oldInsert = session.insert;
  session.insert = function (position, text) {
    return oldInsert.apply(this, [position, outSideRange(position) ? text : ""]);
  };
  /**
   * 重写remove方法： 若range在不可编辑范围内则不可以删除
   */
  const oldRemove = session.remove;
  session.remove = function (range) {
    return intersectsRange(range) ? false : oldRemove.apply(this, arguments);
  };

  /**
   * 重写moveText方法： 若toPosition在不可编辑范围内则可不以移动
   */
  const oldMoveText = session.moveText;
  session.moveText = function (fromRange, toPosition, copy) {
    if (intersectsRange(fromRange) || !outSideRange(toPosition)) return fromRange;
    return oldMoveText.apply(this, arguments);
  };
}

export default setReadOnly;

```



## 初始内容可被撤销/重置撤销栈

每次ace初始化完毕并赋值初始内容后，此时ctrl+z会发生内容消失的情况。因为赋值初始内容其实是一次输入（空 => 初始内容），此时撤销栈会压栈。

解决办法就是，在初始内容赋值完成后立即重置撤销栈。

```Javascript
//初始内容
editor.setValue("And now how can I reset the\nundo stack,-1");
//重置撤销栈UndoManager是保持所有历史的
editor.getSession().setUndoManager(new ace.UndoManager())
```





## ace添加自定义主题

### 1. 编辑样式代码

```CSS
.ace-vs-dark .ace_gutter {
  background: #1e1e1e;
  color: #858585;
  overflow: hidden;
}
.ace-vs-dark .ace_print-margin {
  width: 1px;
  background: #e8e8e8;
}
.ace-vs-dark {
  background-color: #1e1e1e;
  color: #dcdcdc;
}
.ace-vs-dark .ace_cursor {
  color: #dcdcdc;
}
.ace-vs-dark .ace_invisible {
  color: #ffffff40;
}
.ace-vs-dark .ace_constant.ace_buildin {
  color: #569cd6;
}
.ace-vs-dark .ace_constant.ace_language {
  color: #b4cea8;
}
.ace-vs-dark .ace_constant.ace_library {
  color: #b5cea8;
}
.ace-vs-dark .ace_invalid {
  background-color: transparent;
  color: #ff3333;
}
.ace-vs-dark .ace_fold {
}
.ace-vs-dark .ace_support.ace_function {
  color: #dcdcdc;
}
.ace-vs-dark .ace_support.ace_constant {
  color: #569cd6;
}
.ace-vs-dark .ace_support.ace_type,
.ace-vs-dark .ace_support.ace_class .ace-vs-dark .ace_support.ace_other {
  color: #4ec9b0;
}
.ace-vs-dark .ace_variable.ace_parameter {
  color: #dcdcdc;
}
.ace-vs-dark .ace_keyword.ace_operator {
  color: #dcdcdc;
}
.ace-vs-dark .ace_comment {
  color: #608b4e;
}
.ace-vs-dark .ace_comment.ace_doc {
  color: #608b4e;
}
.ace-vs-dark .ace_comment.ace_doc.ace_tag {
  color: #608b4e;
}
.ace-vs-dark .ace_constant.ace_numeric {
  color: #b5cea8;
}
.ace-vs-dark .ace_variable {
  color: #dcdcdc;
}
.ace-vs-dark .ace_xml-pe {
  /**/
  color: rgb(104, 104, 91);
}
.ace-vs-dark .ace_entity.ace_name.ace_function {
  color: #dcdcdc;
}
.ace-vs-dark .ace_heading {
  color: #569cd6;
}
.ace-vs-dark .ace_list {
  color: #dcdcdc;
}
.ace-vs-dark .ace_marker-layer .ace_selection {
  /**/
  background: rgb(181, 213, 255);
}
.ace-vs-dark .ace_marker-layer .ace_step {
  /**/
  background: rgb(252, 255, 0);
}
.ace-vs-dark .ace_marker-layer .ace_stack {
  /**/
  background: rgb(164, 229, 101);
}
.ace-vs-dark .ace_marker-layer .ace_bracket {
  /**/
  margin: -1px 0 0 -1px;
  border: 1px solid rgb(192, 192, 192);
}
.ace-vs-dark .ace_marker-layer .ace_active-line {
  /**/
  background: rgba(0, 0, 0, 0.07);
}
.ace-vs-dark .ace_gutter-active-line {
  background-color: #0f0f0f;
}
.ace-vs-dark .ace_marker-layer .ace_selected-word {
  /**/
  background: rgb(250, 250, 255);
  border: 1px solid rgb(200, 200, 250);
}
.ace-vs-dark .ace_storage,
.ace-vs-dark .ace_keyword,
.ace-vs-dark .ace_meta.ace_tag {
  color: #569cd6;
}
.ace-vs-dark .ace_string.ace_regex {
  /**/
  color: rgb(255, 0, 0);
}
.ace-vs-dark .ace_string {
  color: #d69d85;
}
.ace-vs-dark .ace_entity.ace_other.ace_attribute-name {
  color: #92caf4;
}
```

### 2. 编辑模块代码

将css样式文本复制粘贴进去即可

```JavaScript
/* eslint-disable */
ace.define("ace/theme/vs-dark",["require","exports","module","ace/lib/dom"], function(require, exports, module) {

  exports.isDark = true;
  exports.cssClass = "ace-vs-dark";
  exports.cssText = ".ace-vs-dark .ace_gutter {\
  background: #1e1e1e;\
  color: #858585;\
  overflow: hidden;\
}\
.ace-vs-dark .ace_print-margin {\
  width: 1px;\
  background: #e8e8e8;\
}\
.ace-vs-dark {\
  background-color: #1e1e1e;\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_cursor {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_invisible {\
  color: #ffffff40;\
}\
.ace-vs-dark .ace_constant.ace_buildin {\
  color: #569cd6;\
}\
.ace-vs-dark .ace_constant.ace_language {\
  color: #b4cea8;\
}\
.ace-vs-dark .ace_constant.ace_library {\
  color: #b5cea8;\
}\
.ace-vs-dark .ace_invalid {\
  background-color: transparent;\
  color: #ff3333;\
}\
.ace-vs-dark .ace_fold {\
}\
.ace-vs-dark .ace_support.ace_function {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_support.ace_constant {\
  color: #569cd6;\
}\
.ace-vs-dark .ace_support.ace_type,\
.ace-vs-dark .ace_support.ace_class .ace-vs-dark .ace_support.ace_other {\
  color: #4ec9b0;\
}\
.ace-vs-dark .ace_variable.ace_parameter {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_keyword.ace_operator {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_comment {\
  color: #608b4e;\
}\
.ace-vs-dark .ace_comment.ace_doc {\
  color: #608b4e;\
}\
.ace-vs-dark .ace_comment.ace_doc.ace_tag {\
  color: #608b4e;\
}\
.ace-vs-dark .ace_constant.ace_numeric {\
  color: #b5cea8;\
}\
.ace-vs-dark .ace_variable {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_xml-pe {\
  /**/\
  color: rgb(104, 104, 91);\
}\
.ace-vs-dark .ace_entity.ace_name.ace_function {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_heading {\
  color: #569cd6;\
}\
.ace-vs-dark .ace_list {\
  color: #dcdcdc;\
}\
.ace-vs-dark .ace_marker-layer .ace_selection {\
  /**/\
  background: rgb(181, 213, 255);\
}\
.ace-vs-dark .ace_marker-layer .ace_step {\
  /**/\
  background: rgb(252, 255, 0);\
}\
.ace-vs-dark .ace_marker-layer .ace_stack {\
  /**/\
  background: rgb(164, 229, 101);\
}\
.ace-vs-dark .ace_marker-layer .ace_bracket {\
  /**/\
  margin: -1px 0 0 -1px;\
  border: 1px solid rgb(192, 192, 192);\
}\
.ace-vs-dark .ace_marker-layer .ace_active-line {\
  /**/\
  background: rgba(0, 0, 0, 0.07);\
}\
.ace-vs-dark .ace_gutter-active-line {\
  background-color: #0f0f0f;\
}\
.ace-vs-dark .ace_marker-layer .ace_selected-word {\
  /**/\
  background: rgb(250, 250, 255);\
  border: 1px solid rgb(200, 200, 250);\
}\
.ace-vs-dark .ace_storage,\
.ace-vs-dark .ace_keyword,\
.ace-vs-dark .ace_meta.ace_tag {\
  color: #569cd6;\
}\
.ace-vs-dark .ace_string.ace_regex {\
  /**/\
  color: rgb(255, 0, 0);\
}\
.ace-vs-dark .ace_string {\
  color: #d69d85;\
}\
.ace-vs-dark .ace_entity.ace_other.ace_attribute-name {\
  color: #92caf4;\
}\n";


});
```

### 3. 引入自定义主题

```JavaScript
ace.config.setModuleUrl(
  "ace/theme/vs-dark",
  // eslint-disable-next-line import/no-webpack-loader-syntax
  require("file-loader?esModule=false!./vs-dark.js")
);


ace.edit(context, {
  fontSize: 15,
  theme: "ace/theme/vs-dark",
}
```