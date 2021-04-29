### Vue.js源码解析系列：模板编译
众所周知，Vue模板编译可以分为三步：
1. 解析模板并生成AST
2. AST优化
3. 根据AST生成Render函数

这篇文章将分析第一步：解析模板并生成AST中的第一步：解析模板

### 什么是AST？
It is a hierarchical program representation that presents source code structure according to the grammar of a programming language, each AST node corresponds to an item of a source code.

简单来说，就是用一个对象去描述一段代码


### 整体流程介绍
整体流程其实很简单，首先会有一些前置处理，比如对参数进行处理、对模板进行缓存、判断new Function()是否可用等。
然后调用parseHTML方法进行模板解析，将传入一些配置项以及一些生命周期函数（start、end、chars、comment）。
这些生命周期函数都是用来生成AST的，将会在解析模板的过程中被调用。
在Vue.js源码中，生命周期的编码技巧频繁有用到，可以很好地将不同功能的实现分离开来，利于维护。
比如在这个case，就很好地将生成AST与模板解析两个不同功能的代码实现进行了分离。
```
源码文件路径：src/compiler/parser/index.js
import { parseHTML } from './html-parser'

parseHTML(template, {
  warn,
  expectHTML: options.expectHTML,
  isUnaryTag: options.isUnaryTag,
  canBeLeftOpenTag: options.canBeLeftOpenTag,
  shouldDecodeNewlines: options.shouldDecodeNewlines,
  shouldDecodeNewlinesForHref: options.shouldDecodeNewlinesForHref,
  shouldKeepComment: options.comments,
  outputSourceRange: options.outputSourceRange,

  start (tag, attrs, unary, start, end) {  },  
  end (tag, start, end) {  },
  chars (text: string, start: number, end: number) {  },
  comment (text: string, start, end) {  }
})
```


### 解析template
调用parseHTML方法进行模板解析，其核心是一个while循环，每次循环都从前到后地使用正则表达式去匹配模板的一部分，匹配成功后将针对不同的情况做不同的处理。  

#### 举个例子
比如说有有一段模板`<div><p v-if="flag">123</p></div>`，解析步骤将是：
1. `<div>`
2. `<p v-if=”flag”>`，注意：类似v-if="flag"的属性将会被单独处理
3. `123`
4. `</p>`
5. `</div>`


#### 正则表达式：
此处不做正则的具体介绍，大家有兴趣可以自己研究一下。
```
源码文件路径：src/compiler/parser/html-parser.js

export const unicodeRegExp = /a-zA-Z\u00B7\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u037D\u037F-\u1FFF\u200C-\u200D\u203F-\u2040\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD/
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const dynamicArgAttribute = /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+?\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const ncname = `[a-zA-Z_][\\-\\.0-9_a-zA-Z${unicodeRegExp.source}]*`
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)
const startTagClose = /^\s*(\/?)>/
const endTag = new RegExp(`^<\\/${qnameCapture}[^>]*>`)
const doctype = /^<!DOCTYPE [^>]+>/i
const comment = /^<!\--/
const conditionalComment = /^<!\[/
```

#### parseHTML方法的精简版源码
```
源码文件路径：src/compiler/parser/html-parser.js

const stack = []
let index = 0 // 当前解析到的位置的索引（相对于原始的模板字符串）
let last // 上一次进入循环时的模板字符串
let lastTag // 上一次解析到的HTML Tag

while (html /* 需要解析的模板字符串 */) {
    last = html
    if (!lastTag || !isPlainTextElement(lastTag)) {
      // 第一次循环以及当lastTag不为script,style,textarea会走进该分支逻辑
      let textEnd = html.indexOf('<')

      // 处理存在HTML Tag的情况（当<在第一位的时候即可以认为是HTML Tag）
      if (textEnd === 0) {
        // Comment:
        if (comment.test(html)) { /* do something */ continue }
        if (conditionalComment.test(html)) { /* do something */ continue }
        // Doctype:
        const doctypeMatch = html.match(doctype)
        if (doctypeMatch) { /* do something */ continue }
        // End tag:
        const endTagMatch = html.match(endTag)
        if (endTagMatch) { /* do something */ continue }
        // Start tag:
        const startTagMatch = parseStartTag()
        if (startTagMatch) { /* do something */ continue }
      }
      let text, rest, next
      // 处理文本的情况
      if (textEnd >= 0) { /* do something * }
      // 找不到'<'，整段Tpl视为纯文本
      if (textEnd < 0) {
        text = html
      }
    } else {
      // 暂不考虑这部分逻辑
    }
  }
```

看到这里，其实模板解析的整体流程就很清晰了，接着我们来单独分析每一个case。

#### 注释、Doctype  
当匹配到注释节点时，做的事情很简单。
首先会调用options.comment去生成AST（生成AST的过程将在后面详细介绍，此处暂时先不谈），然后再调用advance方法（下面有advance方法的介绍）去截断模板。  
当匹配到Conditional Comment或者Doctype就更简单了，直接调用advance方法去截断模板，除此之外没有其他操作了。
```
// Comment:
if (comment.test(html)) {
  const commentEnd = html.indexOf('-->')
  if (commentEnd >= 0) {
    if (options.shouldKeepComment) {
      options.comment(html.substring(4, commentEnd), index, index + commentEnd + 3)
    }
    advance(commentEnd + 3)
    continue
  }
}
// Conditional Comment，详细信息可以查看下面的链接
// http://en.wikipedia.org/wiki/Conditional_comment#Downlevel-revealed_conditional_comment
if (conditionalComment.test(html)) {
  const conditionalEnd = html.indexOf(']>')
  if (conditionalEnd >= 0) {
    advance(conditionalEnd + 2)
    continue
  }
}
// Doctype
const doctypeMatch = html.match(doctype)
if (doctypeMatch) {
  advance(doctypeMatch[0].length)
  continue
}
```

关于advance方法：其实就是修改了index变量的值，然后对html变量做截断。
```
function advance (n) {
  index += n
  html = html.substring(n)
}
```

#### 开始标签
整体流程是：首先调用parseStartTag方法，如果parseStartTag方法返回值不为空则会调用handleStartTag方法。  
parseStartTag方法做的事情其实很简单，就是使用正则表达式去获取开始标签及其所有属性，最终返回一个对象，对象有tagName、attrs、start、end四个属性。  
handleStartTag方法首先会做些容错处理、
然后将attr处理成{ name: '', value: '' }的形式，增强可读性、
再然后会往stack中推入一条数据，用于判断模板是否合法、
最后通过options.start生成AST。
```
const startTagMatch = parseStartTag()
if (startTagMatch) {
  handleStartTag(startTagMatch)
  // shouldIgnoreFirstNewline暂时不介绍
  if (shouldIgnoreFirstNewline(startTagMatch.tagName, html)) {
    advance(1)
  }
  continue
}

function parseStartTag () {
  // 匹配Start Tag
  const start = html.match(startTagOpen)
  if (start) {
    const match = {
      tagName: start[1],
      attrs: [],
      start: index
    }
    advance(start[0].length)
    let end, attr
    // 匹配Attrs
    while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
      attr.start = index
      advance(attr[0].length)
      attr.end = index
      match.attrs.push(attr)
    }
    if (end) {
      match.unarySlash = end[1]
      advance(end[0].length)
      match.end = index
      return match
    }
  }
}

function handleStartTag (match) {
  const tagName = match.tagName
  const unarySlash = match.unarySlash
  // 容错处理，此处不介绍
  if (expectHTML) {
    if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
      parseEndTag(lastTag)
    }
    if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
      parseEndTag(tagName)
    }
  }
  const unary = isUnaryTag(tagName) || !!unarySlash
  // 处理attrs数组
  const l = match.attrs.length
  const attrs = new Array(l)
  for (let i = 0; i < l; i++) {
    const args = match.attrs[i]
    const value = args[3] || args[4] || args[5] || ''
    const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
      ? options.shouldDecodeNewlinesForHref
      : options.shouldDecodeNewlines
    attrs[i] = {
      name: args[1],
      value: decodeAttr(value, shouldDecodeNewlines)
    }
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
      attrs[i].start = args.start + args[0].match(/^\s*/).length
      attrs[i].end = args.end
    }
  }
  // 做html标签匹配相关内容
  if (!unary) {
    stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs, start: match.start, end: match.end })
    lastTag = tagName
  }
  // options.start生成AST
  if (options.start) {
    options.start(tagName, attrs, unary, match.start, match.end)
  }
}
```

#### 结束标签
当匹配到结束标签，首先调用advance方法，然后调用parseEndTag方法。
parseEndTag方法一眼看上去很复杂，但是实际上有很多都是兼容处理，可以先不考虑，parseEndTag方法做的最重要的两件事情是：
1. 通过stack判断标签嵌套是否合法
2. 调用options.end（AST相关）
```
const endTagMatch = html.match(endTag)
if (endTagMatch) {
  const curIndex = index
  advance(endTagMatch[0].length)
  parseEndTag(endTagMatch[1], curIndex, index)
  continue
}

function parseEndTag (tagName, start, end) {
  let pos, lowerCasedTagName
  if (start == null) start = index
  if (end == null) end = index
  
  // 在stack中找到最近push的且与该次解析到的Tag相同的索引
  if (tagName) {
    lowerCasedTagName = tagName.toLowerCase()
    for (pos = stack.length - 1; pos >= 0; pos--) {
      if (stack[pos].lowerCasedTag === lowerCasedTagName) {
        break
      }
    }
  } else {
    pos = 0
  }

  // 当用户书写的模板不规范的时候，会出现pos小于0的情况，此处不考虑这些情况。
  if (pos >= 0) {
    // Close all the open elements, up the stack
    for (let i = stack.length - 1; i >= pos; i--) {
      // 嵌套非法时，报错
      if (process.env.NODE_ENV !== 'production' && (i > pos || !tagName) && options.warn) {
        options.warn(
          `tag <${stack[i].tag}> has no matching end tag.`,
          { start: stack[i].start, end: stack[i].end }
        )
      }

      if (options.end) {
        options.end(stack[i].tag, start, end)
      }
    }

    // Remove the open elements from the stack
    stack.length = pos
    lastTag = pos && stack[pos - 1].tag
  } else if (lowerCasedTagName === 'br') {
    if (options.start) {
      options.start(tagName, [], true, start, end)
    }
  } else if (lowerCasedTagName === 'p') {
    if (options.start) {
      options.start(tagName, [], false, start, end)
    }
    if (options.end) {
      options.end(tagName, start, end)
    }
  }
}
```

#### 文本
当`textEnd >= 0`，则视为文本。
注意，此处出现的while循环是为了解决文本中出现'<'的情况，如：`<p>123<456</p>`，保证能匹配到完整的文本。
当`textEnd < 0`，即找不到'<'，则将整段Tpl视为纯文本。  
最后，会调用advance方法以及options.chars生成AST。
```
if (textEnd >= 0) {
  rest = html.slice(textEnd)
  while (
    !endTag.test(rest) &&
    !startTagOpen.test(rest) &&
    !comment.test(rest) &&
    !conditionalComment.test(rest)
  ) {
    // < in plain text, be forgiving and treat it as text
    next = rest.indexOf('<', 1)
    if (next < 0) break
    textEnd += next
    rest = html.slice(textEnd)
  }
  text = html.substring(0, textEnd)
}
if (textEnd < 0) {
  text = html
}
if (text) {
  advance(text.length)
}
if (options.chars && text) {
  options.chars(text, index - text.length, index)
}
```
