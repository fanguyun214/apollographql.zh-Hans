---
title: 使用 Babel 编译查询
---

如果您希望在 Javascript 文件中协同定位 GraphQL 查询，通常使用 [graphql-tag](https://github.com/apollographql/graphql-tag)库来编写它们。这需要将查询字符串处理为 GraphQL AST，这将增加应用程序的启动时间，特别是如果您有许多查询。

为了避免这种运行时开销，您可以预先编译 `graphql-tag` 创建的查询使用 [Babel](http://babeljs.io/). Here are two ways you can do this:

1. 使用 [babel-plugin-graphql-tag](#using-babel-plugin-graphql-tag)
2. 使用 [graphql-tag.macro](#using-graphql-tagmacro)

如果您希望将 GraphQL 代码保存在单独的文件中 (`.graphql` or `.gql`) 你可以使用 [babel-plugin-import-graphql](https://github.com/detrohutt/babel-plugin-import-graphql)。 这个插件仍然在引擎下使用 `graphql-tag` ，但是透明的。 您只需导入操作/片段，就好像每个都是从 GraphQL 文件导出一样。 这具有与上述方法相同的预编译益处。

## 使用 babel-plugin-graphql-tag

这种方法可以像往常一样使用 `graphql-tag` 库，当使用这个 babel 插件处理文件时，对该库的调用将被预编译的结果所取代。

在开发依赖项中安装插件：

```
# with npm
npm install --save-dev babel-plugin-graphql-tag

# or with yarn
yarn add --dev babel-plugin-graphql-tag
```

然后在 `.babelrc` 配置文件中添加插件：

```
{
  "plugins": [
    "graphql-tag"
  ]
}
```

就是这样！ 将删除 `graphql-tag` 中的 `import gql` 的所有用法，并且将对编译版本替换对 `gql` 的调用。

## 使用 graphql-tag.macro

这种方法更加明确，因为你改变了 `graphql-tag.macro` 的 `graphql-tag` 的所有用法， 它导出一个`gql`函数，你可以使用与原始函数相同的方法。 这个 macro 需要 [babel-macros](https://github.com/kentcdodds/babel-macros) 插件, 这将与前一种方法相同，但仅限于来自 macro 导入的调用，而不会对常规调用`graphql-tag`库进行调用。

你为什么喜欢这种方法？ 主要是因为它需要较少的配置 (`babel-macros`适用于所有类型的 macro，所以如果你已经安装它，你不必做任何其他事情)，因为显而易见。 您可以阅读有关使用 `babel-macros`  理由的更多信息 [在这篇博文中](http://babeljs.io/blog/2017/09/11/zero-config-with-babel-macros).

要使用它，只要你 [已经安装了 babel-macros](https://github.com/kentcdodds/babel-macros#installation) 和 [configured](https://github.com/kentcdodds/babel-macros/blob/master/other/docs/user.md), 你只需要修改这些：

```js
import gql from 'graphql-tag';

const query = gql`
  query {
    hello {
      world
    }
  }
`;
```

对应:

```js
import gql from 'graphql-tag.macro'; // <-- Use the macro

const query = gql`
  query {
    hello {
      world
    }
  }
`;
```

## 使用 babel-plugin-import-graphql

在开发依赖项中安装插件：

```
# with npm
npm install --save-dev babel-plugin-import-graphql

# or with yarn
yarn add --dev babel-plugin-import-graphql
```

然后在 `.babelrc` 配置文件中添加插件：

```
{
  "plugins": [
    "import-graphql"
  ]
}
```

现在，从 GraphQL 文件类型导入的任何 `import` 语句都将返回一个可立即使用的 GraphQL DocumentNode 对象。

```javascript
import React, { Component } from 'react';
import { graphql } from 'react-apollo';
import myImportedQuery from './productsQuery.graphql';
// or for files with multiple operations:
// import { query1, query2 } from './queries.graphql';

class QueryingComponent extends Component {
  render() {
    if (this.props.data.loading) return <h3>Loading...</h3>;
    return <div>{`This is my data: ${this.props.data.queryName}`}</div>;
  }
}

export default graphql(myImportedQuery)(QueryingComponent);
```

## Fragments

有这些方法都支持 Fragments 的使用。

对于前两种方法，您可以在对 `gql` 的不同调用中定义 Fragments（在同一文件中或在不同文件中）。 然后，您可以使用插值将它们包含到主查询中，如下所示：

```js
import gql from 'graphql-tag';
// or import gql from 'graphql-tag.macro';

const fragments = {
  hello: gql`
    fragment HelloStuff on Hello {
      universe
      galaxy
    }
  `
};

const query = gql`
  query {
    hello {
      world
      ...HelloStuff
    }
  }

  ${fragments.hello}
`;
```

使用 `babel-plugin-import-graphql`，您可以将您的 Fragments 包含在 GraphQL 文件中，无论使用它还是什么，甚至可以使用 `#import` 语法从单独的文件中导入它。 更多信息见 [README](https://github.com/detrohutt/babel-plugin-import-graphql) 
