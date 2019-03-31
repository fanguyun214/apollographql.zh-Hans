---
title: 使用 Apollo 和 TypeScript
sidebar_title: 使用 TypeScript
---

**Note:  Apollo 不再附带 Flow 类型。 如果您想帮助添加这些，请联系沟通!**

随着应用程序的增长，您可能会发现包含类型系统帮助开发很有帮助。 Apollo 支持开箱即用的 TypeScript 类型定义。 `apollo-client` 和 `react-apollo` 都带有 npm 包中的定义，因此在项目中包含库之后会为您完成安装。

这些文档假设您已经在项目中配置了TypeScript，如果没有开始 [参考此处](https://github.com/Microsoft/TypeScript-React-Conversion-Guide#typescript-react-conversion-guide).

使用带有 GraphQL 的类型系统时，最常见的需求是键入操作的结果。 鉴于 GraphQL 服务器的模式是强类型的， 我们甚至可以使用自动生成TypeScript定义的工具，像 [apollo-codegen](https://github.com/apollographql/apollo-codegen). 但是，在本文档中，我们将手动编写结果类型。

<h2 id="typing-components">输入组件API</h2>

将 Apollo 与 TypeScript 一起使用并不比使用 React Apollo 2.1 中发布的组件 API 更容易：

```js
interface Data {
  allPeople: {
    people: Array<{ name: string }>;
  };
};

interface Variables {
  first: number;
};

class AllPeopleQuery extends Query<Data, Variables> {}
```

现在我们可以在树中使用 `AllPeopleQuery` 代替 `Query` 来获得完整的 TypeScript 支持！ 由于我们没有将任何 props 映射到我们的组件中，也没有重写传递下来的 props，我们只需要提供我们的数据模型及其工作所需的变量！ 其他一切都由 React Apollo 强大的类型定义来处理。

对于 `<Query />`， `<Mutation />` 和 `<Subscription />` 组件，这种方法完全相同！ 一次学习，共用 Apollo 获得有史以来最好的类型。


<h2 id="operation-result">输入高阶组件</h2>

由于查询的结果将作为 props 发送到包装组件，我们希望能够告诉我们的类型系统这些 props 的模型。 下面是使用 `graphql` 高阶组件的操作的示例设置类型（** note **：以下部分也适用于查询，变异和预订）:

```javascript
import React from "react";
import gql from "graphql-tag";
import { ChildDataProps, graphql } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string;
  id: string;
  appearsIn: string[];
  friends: Hero[];
};

type Response = {
  hero: Hero;
};

type Variables = {
  episode: string;
};

type ChildProps = ChildDataProps<{}, Response, Variables>;

// Note that the first parameter here is an empty Object, which means we're
// not checking incoming props for type safety in this example. The next
// example (in the "Options" section) shows how the type safety of incoming
// props can be ensured.
const withCharacter = graphql<{}, Response, Variables, ChildProps>(HERO_QUERY, {
  options: () => ({
    variables: { episode: "JEDI" }
  })
});

export default withCharacter(({ data: { loading, hero, error } }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// actual component with data;
});
```

<h3 id="options">选项</h3>

通常，查询的变量将根据包装器组件的 props 计算。 无论组件在您的应用程序中使用何处，调用者都会传递参数，我们希望我们的类型系统验证这些 props 的类型可能是什么样子。 以下是设置 props 类型的示例：

```javascript
import React from "react";
import gql from "graphql-tag";
import { ChildDataProps, graphql } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string;
  id: string;
  appearsIn: string[];
  friends: Hero[];
};

type Response = {
  hero: Hero;
};

type InputProps = {
  episode: string;
};

type Variables = {
  episode: string;
};

type ChildProps = ChildDataProps<InputProps, Response, Variables>;

const withCharacter = graphql<InputProps, Response, Variables, ChildProps>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
});

export default withCharacter(({ data: { loading, hero, error } }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// actual component with data;
});
```

当访问通过 props 传递给组件的深层嵌套对象时，这尤其有用。 例如，在添加 prop 类型时，使用 TypeScript 的项目将开始显示传递的 props 无效的错误：

```javascript
import React from "react";
import { ApolloClient } from "apollo-client";
import { createHttpLink } from "apollo-link-http";
import { InMemoryCache } from "apollo-cache-inmemory";
import { ApolloProvider } from "react-apollo";

import Character from "./Character";

export const link = createHttpLink({
  uri: "https://mpjk0plp9.lp.gql.zone/graphql"
});

export const client = new ApolloClient({
  cache: new InMemoryCache(),
  link,
});

export default () =>
  <ApolloProvider client={client}>
    // $ExpectError property `episode`. Property not found in. See: src/Character.js:43
    <Character />
  </ApolloProvider>;
```

<h3 id="props">Props</h3>

React 集成最强大的功能之一是 `props` 功能，它允许您将操作中的结果数据重新整形为包装组件的新模型 props。 GraphQL 非常棒，只需要您从服务器请求所需的数据。客户端仍然经常需要根据这些结果重新整形或进行客户端计算。 返回值甚至可以根据操作的状态（即 loading, error, recieved 的数据）而有所不同，因此通知我们的类型系统选择这些可能的值对于确保我们的组件不会有运行时错误非常重要。

来自 `react-apollo` 的 `graphql` 包装器支持手动声明结果 props 的模型。

```javascript
import React from "react";
import gql from "graphql-tag";
import { graphql, ChildDataProps } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string;
  id: string;
  appearsIn: string[];
  friends: Hero[];
};

type Response = {
  hero: Hero;
};

type InputProps = {
  episode: string
};

type Variables = {
  episode: string
};

type ChildProps = ChildDataProps<InputProps, Response, Variables>;

const withCharacter = graphql<InputProps, Response, Variables, ChildProps>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
  props: ({ data }) => ({ ...data })
});

export default withCharacter(({ loading, hero, error }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// actual component with data;
});
```

由于我们已经输入了响应模型，props 模型以及将传递给客户端的模型，因此我们可以防止多个地方出现错误。 我们在 `graphql` 包装器中的选项和 props function 现在是类型安全的，我们的渲染组件受到保护，我们的组件树已经强制执行所需的 props。

```javascript
export const withCharacter = graphql<InputProps, Response, Variables, Props>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
  props: ({ data, ownProps }) => ({
    ...data,
    // $ExpectError [string] This type cannot be compared to number
    episode: ownProps.episode > 1,
    // $ExpectError property `isHero`. Property not found on object type
    isHero: data && data.hero && data.hero.isHero
  })
});
```

通过这种添加，可以静态地键入 Apollo 和 React 之间的整体集成。 结合每个系统提供的强大工具，它可以大大改善应用程序和开发人员的体验。

<h3 id="classes-vs-functions">类与函数</h3>

所有上述示例都显示了使用 `graphql` 包装器包装一个只是函数的组件。有时，依赖于 GraphQL 数据的组件需要状态，并使用 `class MyComponent extends React.Component` 模式形成。 在这些用例中，TypeScript 需要将 prop 模型添加到类实例中。 为了支持这一点，`react-apollo` 导出类型以支持轻松创建结果类型。

```javascript
import { ChildProps } from "react-apollo";

const withCharacter = graphql<InputProps, Response>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  })
});

class Character extends React.Component<ChildProps<InputProps, Response>, {}> {
  render(){
    const { loading, hero, error } = this.props.data;
    if (loading) return <div>Loading</div>;
    if (error) return <h1>ERROR</h1>;
    return ...// actual component with data;
  }
}

export default withCharacter(Character);
```

<h3 id="using-name">使用 `name` 属性</h3>
如果在 `graphql` 包装器的配置中使用 `name` 属性，则需要手动将响应类型附加到 `props` 函数。 使用 TypeScript 的示例如下：

```javascript
import { NamedProps, QueryProps } from 'react-apollo';

export const withCharacter = graphql<InputProps, Response, {}, Prop>(HERO_QUERY, {
  name: 'character',
  props: ({ character, ownProps }: NamedProps<{ character: QueryProps & Response }, Props) => ({
    ...character,
    // $ExpectError [string] This type cannot be compared to number
    episode: ownProps.episode > 1,
    // $ExpectError property `isHero`. Property not found on object type
    isHero: character && character.hero && character.hero.isHero
  })
});
```
