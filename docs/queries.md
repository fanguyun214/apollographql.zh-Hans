---
标题: Queries
描述: 学习如何使用 Query 组件获取数据
---

以简单可预测的方式获取数据是 Apollo Client 的核心功能之一。 在本指南中，您将学习如何构建 Query 组件以获取 GraphQL 数据并将结果附加到 UI。 您还将了解 Apollo Client 如何通过跟踪错误和加载状态来简化数据管理代码。


假设您熟悉构建 GraphQL 查询。 如果你想复习一下, 我们推荐 [阅读本指南](http://graphql.org/learn/queries/) 和练习 [在GraphiQL中运行查询](../features/developer-tooling.html#features). 由于 Apollo Client 查询是标准的 GraphQL，因此您在 GraphiQL 中成功运行的任何查询也将在 Apollo Query 组件中运行。

以下示例假设您已经设置了 Apollo Client 并将您的 React 应用程序包装在 `ApolloProvider` 组件中。 阅读我们的 [入门](./get-started.html) 指南, 如果您需要有关这两个步骤的帮助。

> 如果你想参考这些例子，请打开 CodeSandbox 上的 [启动项目](https://codesandbox.io/s/j2ly83749w)和 [this CodeSandbox](https://codesandbox.IO/S/32ypr38l61) 上的示例 GraphQL 服务器。 您可以查看应用程序的 [完整版本](https://codesandbox.io/s/n3jykqpxwm)。

<h2 id="basic">Query组件</h2>

Query 组件是 Apollo 应用程序最重要的构建块之一。 要创建 Query 组件， 只需将包含 `gql` 函数的 GraphQL 查询字符串传递给 `this.props.query` ，并为 `this.props.children` 提供一个函数，告诉 React 要呈现什么。
 `Query` 组件是使用 [render prop](https://reactjs.org/docs/render-props.html) 模式的React组件的示例。 React将使用 Apollo Client 中的对象调用您提供的 render prop 函数，该对象包含可用于呈现 UI 的加载，错误和数据属性。 我们来看一个例子：

首先，让我们创建 GraphQL 查询。 请记住将查询字符串包装在 `gql` 函数中，以便将其解析为查询文档。 一旦我们创建好 GraphQL 查询，让我们通过将它传递给 `query` prop 来将它附加到我们的 `Query` 组件。

我们还需要为 `Query` 组件提供一个子函数，它将告诉 React 我们想要呈现什么。 我们可以使用 `Query` 组件为我们提供的 `loading`，`error` 和 `data` 属性，以便根据查询的状态智能地呈现不同的 UI。 让我们看看这是什么样的！

```jsx
import gql from "graphql-tag";
import { Query } from "react-apollo";

const GET_DOGS = gql`
  {
    dogs {
      id
      breed
    }
  }
`;

const Dogs = ({ onDogSelected }) => (
  <Query query={GET_DOGS}>
    {({ loading, error, data }) => {
      if (loading) return "Loading...";
      if (error) return `Error! ${error.message}`;

      return (
        <select name="dog" onChange={onDogSelected}>
          {data.dogs.map(dog => (
            <option key={dog.id} value={dog.breed}>
              {dog.breed}
            </option>
          ))}
        </select>
      );
    }}
  </Query>
);
```

如果你在 `App` 组件中渲染 `Dogs`，一旦 Apollo Client 从服务器接收数据，你将首先看到加载状态，然后是带有狗品种列表的表单。 当表单值改变时，我们将通过 `this.props.onDogSelected` 将值发送给父组件， 最终将值传递给 `DogPhoto` 组件。

在下一步中，我们将通过构建一个 `DogPhoto` 组件将表单挂载到一个更复杂的变量查询。

<h2 id="data">接收数据</h2>

您已经在 render prop 函数中看到了如何处理查询结果。 当我们从 `Query` 组件中获取数据时，让我们深入了解 Apollo Client 幕后发生的事情。

1. 当 `Query` 组件安装时，Apollo Client 会为我们的查询创建一个 observable。 我们的组件通过 Apollo Client 缓存订阅查询结果。
2. 首先，我们尝试 从 Apollo 缓存加载查询结果。 如果缓存中不存在，我们将请求发送到服务器。
3. 一旦数据返回，我们将其标准化并将其存储在 Apollo 缓存中。 由于 `Query` 组件订阅了结果，因此它会自动更新数据。

要查看 Apollo Client 的缓存操作，让我们构建我们的 `DogPhoto` 组件。 `DogPhoto` 接受一个名为 `breed` 的道具，它反映了上面 `Dogs` 组件中我们表单的当前值。

```jsx
const GET_DOG_PHOTO = gql`
  query Dog($breed: String!) {
    dog(breed: $breed) {
      id
      displayImage
    }
  }
`;

const DogPhoto = ({ breed }) => (
  <Query query={GET_DOG_PHOTO} variables={{ breed }}>
    {({ loading, error, data }) => {
      if (loading) return null;
      if (error) return `Error!: ${error}`;

      return (
        <img src={data.dog.displayImage} style={{ height: 100, width: 100 }} />
      );
    }}
  </Query>
);
```

您会注意到 `Query` 组件上有一个新的配置选项。 prop`variable` 是一个包含我们想要传递给 GraphQL 查询的变量对象。 在这种情况下，我们希望将表格中的品种传递给我们的查询。

尝试从列表中选择 "bulldog" 以查看其照片显示。 然后，切换到另一个品种并切换回 "bulldog"。 你会注意到牛头犬照片第二次瞬间加载。 这是 Apollo 缓存在起作用！

接下来，让我们学习一些确保数据新鲜的技术，例如轮询和重新获取。

<h2 id="refetching">轮询和重新获取</h2>

Apollo Client 为您缓存数据真是太棒了，但是当我们需要新数据时我们应该怎么做？ 两种解决方案是轮询和重新获取。

轮询可以通过在指定的时间间隔内重新获取来帮助我们实现接近实时的数据。 要实现轮询，只需将 `pollInterval` prop传递给 `Query` 组件，间隔为 ms。 如果传入 0，查询将不会轮询。 您还可以通过在传递给 render prop 函数的结果对象上使用 `startPolling` 和 `stopPolling` 函数来实现动态轮询。

```jsx
const DogPhoto = ({ breed }) => (
  <Query
    query={GET_DOG_PHOTO}
    variables={{ breed }}
    skip={!breed}
    pollInterval={500}
  >
    {({ loading, error, data, startPolling, stopPolling }) => {
      if (loading) return null;
      if (error) return `Error!: ${error}`;

      return (
        <img src={data.dog.displayImage} style={{ height: 100, width: 100 }} />
      );
    }}
  </Query>
);
```

通过将 `pollInterval` 设置为 500 ，您应该每隔 0.5 秒看到一个新的狗图像。 轮询是实现近实时数据的绝佳方式，而无需复杂的设置 GraphQL 订阅。

如果要重新加载查询以响应用户操作而不是间隔，该怎么办？ 这就是 `refetch` 函数的用武之地！ 在这里，我们在 `DogPhoto` 组件中添加了一个按钮，该按钮会在单击时触发重新获取。`refetch` 接受变量，但如果我们不传入新变量，它将使用我们之前查询中的相同变量。

```jsx
const DogPhoto = ({ breed }) => (
  <Query
    query={GET_DOG_PHOTO}
    variables={{ breed }}
    skip={!breed}
  >
    {({ loading, error, data, refetch }) => {
      if (loading) return null;
      if (error) return `Error!: ${error}`;

      return (
        <div>
          <img
            src={data.dog.displayImage}
            style={{ height: 100, width: 100 }}
          />
          <button onClick={() => refetch()}>Refetch!</button>
        </div>
      );
    }}
  </Query>
);
```

如果单击该按钮，您会注意到我们的 UI 更新了最新的数据。 重新获取是保证新数据的一种很好的方法，但它在加载状态时引入了一些额外的复杂性。 在下一节中，您将学习处理复杂加载和错误状态的策略。

<h2 id="loading">加载和错误状态</h2>

我们已经看到 Apollo Client 如何在渲染函数中公开我们的查询的加载和错误状态。 这些属性对于初次查询加载时有用，但是当我们重新获取或轮询时，我们的加载状态会发生什么？

让我们回到上一节中的示例。 如果单击 “Refetch” 按钮，您将看到组件在新数据到达之前不会重新呈现。 如果我们想向用户表明我们正在重新获取照片怎么办？

幸运的是，Apollo Client 通过 render prop 函数中结果对象的 `networkStatus` 属性提供有关查询状态的细粒度信息。 我们还需要将 prop 的 notifyOnNetworkStatusChange 设置为 true ，以便我们的查询组件在重新加载时重新渲染。

```jsx
const DogPhoto = ({ breed }) => (
  <Query
    query={GET_DOG_PHOTO}
    variables={{ breed }}
    skip={!breed}
    notifyOnNetworkStatusChange
  >
    {({ loading, error, data, refetch, networkStatus }) => {
      if (networkStatus === 4) return "Refetching!";
      if (loading) return null;
      if (error) return `Error!: ${error}`;

      return (
        <div>
          <img
            src={data.dog.displayImage}
            style={{ height: 100, width: 100 }}
          />
          <button onClick={() => refetch()}>Refetch!</button>
        </div>
      );
    }}
  </Query>
);
```

`networkStatus` 属性是一个枚举，其数字值为 1-8，表示不同的加载状态。 4 对应重新获取，但也有用于轮询和分页的数字值。 有关所有可能加载状态的完整列表，请查看[参考指南](../api/react-apollo.html#graphql-query-data-networkStatus)。

虽然没有加载状态那么复杂，但是也可以通过 `Query` 组件上的 `errorPolicy` prop 来自定义组件中的错误。 `errorPolicy` 的默认值是 “none” ，我们将所有 GraphQL 错误视为运行时错误。 如果发生错误，Apollo Client 将丢弃随请求返回的任何数据，并将 render prop 函数中的 `error` 属性设置为 true。 如果您想显示任何部分数据以及任何错误信息，请将 `errorPolicy` 设置为 “all”。

<h2 id="manual-query">手动触发查询</h2>

当 React 安装一个 `Query` 组件时，Apollo Client 会自动触发你的查询。 如果您想延迟触发查询，直到用户执行操作（例如单击按钮），该怎么办？ 对于这种情况，我们想要使用 `ApolloConsumer` 组件并直接调用 `client.query()`。

```jsx
import React, { Component } from 'react';
import { ApolloConsumer } from 'react-apollo';

class DelayedQuery extends Component {
  state = { dog: null };

  onDogFetched = dog => this.setState(() => ({ dog }));

  render() {
    return (
      <ApolloConsumer>
        {client => (
          <div>
            {this.state.dog && <img src={this.state.dog.displayImage} />}
            <button
              onClick={async () => {
                const { data } = await client.query({
                  query: GET_DOG_PHOTO,
                  variables: { breed: "bulldog" }
                });
                this.onDogFetched(data.dog);
              }}
            >
              Click me!
            </button>
          </div>
        )}
      </ApolloConsumer>
    );
  }
}
```

以这种方式获取是非常冗长的，所以我们建议尽可能使用 `Query` 组件！
> 如果您想查看我们刚刚构建的应用程序的完整版本，您可以查看CodeSandbox [此处](https://codesandbox.io/s/n3jykqpxwm).

<h2 id="api">Query API 概览</h2>

如果您正在寻找所有 `Query` 组件接受及其渲染方法功能的概述，请不要再看了！ 大多数 `Query` 组件不需要所有这些配置选项，但知道它们存在是有用的。 如果您想通过使用示例更详细地了解 `Query` 组件 API，请访问我们的[参考指南](../api/react-apollo.html).

<h3 id="props">Props</h3>

Query 组件接受以下 props。 只有 `query` 和 `children`是必须的 **required**.

<dl>
  <dt>`query`: DocumentNode</dt>
  <dd>GraphQL查询文档通过`graphql-tag`解析为AST。 **Required**</dd>
  <dt>`children`: (result: QueryResult) => React.ReactNode</dt>
  <dd>一个函数，根据查询结果返回要渲染的 UI。**Required**</dd>
  <dt>`variables`: { [key: string]: any }</dt>
  <dd>包含执行查询需要的所有变量对象</dd>
  <dt>`pollInterval`: number</dt>
  <dd>指定组件轮询数据的时间间隔（以毫秒为单位）。 默认为 0（无轮询）。</dd>
  <dt>`notifyOnNetworkStatusChange`: boolean</dt>
  <dd>是否更新网络状态或网络错误应重新呈现组件。 默认为false。</dd>
  <dt>`fetchPolicy`: FetchPolicy</dt>
  <dd>您希望组件如何与 Apollo 缓存交互。 默认为 “cache-first”。</dd>
  <dt>`errorPolicy`: ErrorPolicy</dt>
  <dd>您希望组件如何处理网络和 GraphQL 错误。 默认为 “none”，这意味着我们将 GraphQL 错误视为运行时错误。</dd>
  <dt>`ssr`: boolean</dt>
  <dd>传入 false 以在服务端渲染期间跳过查询。</dd>
  <dt>`displayName`: string</dt>
  <dd>要在 React DevTools 中显示的组件的名称。 默认为 'Query'。</dd>
  <dt>`skip`: boolean</dt>
  <dd>如果 skip 为 true，则将完全跳过查询。</dd>
  <dt>`onCompleted`: (data: TData | {}) => void</dt>
  <dd>查询成功完成后执行的回调。</dd>
  <dt>`onError`: (error: ApolloError) => void</dt>
  <dd>发生错误时执行的回调。</dd>
  <dt>`context`: Record<string, any></dt>
  <dd>查询组件与网络接口（Apollo Link）之间的共享上下文。 用于从道具设置标题或将信息发送到 Apollo Boost 的 `request` 功能。</dd>
  <dt>`partialRefetch`: boolean</dt>
  <dd>如果是 `true`，如果查询结果被标记为部分，则执行查询 `refetch`，并且由 Apollo Client `QueryManager` 将返回的数据重置为空对象（由于高速缓存未命中）。 出于向后兼容性的原因，默认值为 “false”，但对于大多数实例，应将其更改为true。</dd>
</dl>

<h3 id="render-prop">Render prop function</h3>

使用具有以下属性的对象（`QueryResult`）调用传递给 `Query` 的 `children` prop 的 render prop 函数。 此对象包含您的查询结果，以及一些有用的函数，用于重新获取，动态轮询和分页。

<dl>
  <dt>`data`: TData</dt>
  <dd>包含 GraphQL 查询结果的对象。 默认为空对象。</dd>
  <dt>`loading`: boolean</dt>
  <dd>一个布尔值，指示请求是否在执行中</dd>
  <dt>`error`: ApolloError</dt>
  <dd>使用 `graphQLErrors` 和 `networkError` 属性的运行时错误</dd>
  <dt>`variables`: { [key: string]: any }</dt>
  <dd>包含调用查询的变量对象</dd>
  <dt>`networkStatus`: NetworkStatus</dt>
  <dd>1-8 之间的数字，对应网络请求的详细状态。 包括有关重新获取和轮询状态的信息。 与 `notifyOnNetworkStatusChange` 道具一起使用。
</dd>
  <dt>`refetch`: (variables?: TVariables) => Promise<ApolloQueryResult></dt>
  <dd>一个允许您重新获取查询并可选地传入新变量的函数</dd>
  <dt>`fetchMore`: ({ query?: DocumentNode, variables?: TVariables, updateQuery: Function}) => Promise<ApolloQueryResult></dt>
  <dd>为您的查询启用 [pagination](../features/pagination.html) 的函数</dd>
  <dt>`startPolling`: (interval: number) => void</dt>
  <dd>此函数以 ms 为单位设置间隔，并在每次指定的间隔通过时获取查询。</dd>
  <dt>`stopPolling`: () => void</dt>
  <dd>此函数停止查询轮询。</dd>
  <dt>`subscribeToMore`: (options: { document: DocumentNode, variables?: TVariables, updateQuery?: Function, onError?: Function}) => () => void</dt>
  <dd>设置 [subscription](../advanced/subscriptions.html) 的函数。 `subscribeToMore` 返回一个可用于取消订阅的函数。</dd>
  <dt>`updateQuery`: (previousResult: TData, options: { variables: TVariables }) => TData</dt>
  <dd>函数，允许您在获取，变异或订阅的上下文之外的缓存中更新查询的结果</dd>
  <dt>`client`: ApolloClient</dt>
  <dd>你的 'ApolloClient` 实例。 用于手动触发查询或将数据写入。</dd>
</dl>

<h2 id="next-steps">下一步</h2>

学习如何构建 `Query` 组件来获取数据是掌握 Apollo Client 开发的最重要技能之一。 既然您是获取数据的专家，为什么不尝试构建 `Mutation` 组件来更新您的数据？ 以下是我们认为可以帮助您提升技能的一些资源：

- [Mutations](./mutations.html): 了解如何使用突变更新数据以及何时需要更新 Apollo 缓存。 有关选项的完整列表，请查看 `Mutation` 组件的API参考。
- [Local state management](./local-state.html): 学习如何使用 `apollo-link-state` 查询本地数据。
- [Pagination](../features/pagination.html): 基于 Apollo Client 的 `fetchMore` 功能，构建列表从未如此简单。 在我们的分页教程中了解更多信息。
- [Query component video by Sara Vieira](https://youtu.be/YHJ2CaS0vpM): 如果您需要通过观看视频进行复习或学习，请查看 Sara 的 `Query` 组件的教程！
