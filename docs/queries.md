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
2. First, we try to load the query result from the Apollo cache. If it's not in there, we send the request to the server.
3. Once the data comes back, we normalize it and store it in the Apollo cache. Since the `Query` component subscribes to the result, it updates with the data reactively.

To see Apollo Client's caching in action, let's build our `DogPhoto` component. `DogPhoto` accepts a prop called `breed` that reflects the current value of our form from the `Dogs` component above.

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

You'll notice there is a new configuration option on our `Query` component. The prop `variables` is an object containing the variables we want to pass to our GraphQL query. In this case, we want to pass the breed from the form into our query.

Try selecting "bulldog" from the list to see its photo show up. Then, switch to another breed and switch back to "bulldog". You'll notice that the bulldog photo loads instantaneously the second time around. This is the Apollo cache at work!

Next, let's learn some techniques for ensuring our data is fresh, such as polling and refetching.

<h2 id="refetching">Polling and refetching</h2>

It's awesome that Apollo Client caches your data for you, but what should we do when we want fresh data? Two solutions are polling and refetching.

Polling can help us achieve near real-time data by causing the query to refetch on a specified interval. To implement polling, simply pass a `pollInterval` prop to the `Query` component with the interval in ms. If you pass in 0, the query will not poll. You can also implement dynamic polling by using the `startPolling` and `stopPolling` functions on the result object passed to the render prop function.

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

By setting the `pollInterval` to 500, you should see a new dog image every .5 seconds. Polling is an excellent way to achieve near-realtime data without the complexity of setting up GraphQL subscriptions.

What if you want to reload the query in response to a user action instead of an interval? That's where the `refetch` function comes in! Here, we're adding a button to our `DogPhoto` component that will trigger a refetch when clicked. `refetch` takes variables, but if we don't pass in new variables, it will use the same ones from our previous query.

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

If you click the button, you'll notice that our UI updates with a new dog photo. Refetching is an excellent way to guarantee fresh data, but it introduces some added complexity with loading state. In the next section, you'll learn strategies to handle complex loading and error state.

<h2 id="loading">Loading and error state</h2>

We've already seen how Apollo Client exposes our query's loading and error state in the render prop function. These properties are helpful for when the query initially loads, but what happens to our loading state when we're refetching or polling?

Let's go back to our refetching example from the previous section. If you click on the refetch button, you'll see that the component doesn't re-render until the new data arrives. What if we want to indicate to the user that we're refetching the photo?

Luckily, Apollo Client provides fine-grained information about the status of our query via the `networkStatus` property on the result object in the render prop function. We also need to set the prop `notifyOnNetworkStatusChange` to true so our query component re-renders while a refetch is in flight.

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

The `networkStatus` property is an enum with number values from 1-8 representing a different loading state. 4 corresponds to a refetch, but there are also numbers for polling and pagination. For a full list of all the possible loading states, check out the [reference guide](../api/react-apollo.html#graphql-query-data-networkStatus).

While not as complex as loading state, responding to errors in your component is also customizable via the `errorPolicy` prop on the `Query` component. The default value for `errorPolicy` is "none" in which we treat all GraphQL errors as runtime errors. In the event of an error, Apollo Client will discard any data that came back with the request and set the `error` property in the render prop function to true. If you'd like to show any partial data along with any error information, set the `errorPolicy` to "all".

<h2 id="manual-query">Manually firing a query</h2>

When React mounts a `Query` component, Apollo Client automatically fires off your query. What if you wanted to delay firing your query until the user performs an action, such as clicking on a button? For this scenario, we want to use an `ApolloConsumer` component and directly call `client.query()` instead.

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

Fetching this way is quite verbose, so we recommend trying to use a `Query` component if at all possible!

> If you'd like to view a complete version of the app we just built, you can check out the CodeSandbox [here](https://codesandbox.io/s/n3jykqpxwm).

<h2 id="api">Query API overview</h2>

If you're looking for an overview of all the props `Query` accepts and its render prop function, look no further! Most `Query` components will not need all of these configuration options, but it's useful to know that they exist. If you'd like to learn about the `Query` component API in more detail with usage examples, visit our [reference guide](../api/react-apollo.html).

<h3 id="props">Props</h3>

The Query component accepts the following props. Only `query` and `children` are **required**.

<dl>
  <dt>`query`: DocumentNode</dt>
  <dd>A GraphQL query document parsed into an AST by `graphql-tag`. **Required**</dd>
  <dt>`children`: (result: QueryResult) => React.ReactNode</dt>
  <dd>A function returning the UI you want to render based on your query result. **Required**</dd>
  <dt>`variables`: { [key: string]: any }</dt>
  <dd>An object containing all of the variables your query needs to execute</dd>
  <dt>`pollInterval`: number</dt>
  <dd>Specifies the interval in ms at which you want your component to poll for data. Defaults to 0 (no polling).</dd>
  <dt>`notifyOnNetworkStatusChange`: boolean</dt>
  <dd>Whether updates to the network status or network error should re-render your component. Defaults to false.</dd>
  <dt>`fetchPolicy`: FetchPolicy</dt>
  <dd>How you want your component to interact with the Apollo cache. Defaults to "cache-first".</dd>
  <dt>`errorPolicy`: ErrorPolicy</dt>
  <dd>How you want your component to handle network and GraphQL errors. Defaults to "none", which means we treat GraphQL errors as runtime errors.</dd>
  <dt>`ssr`: boolean</dt>
  <dd>Pass in false to skip your query during server-side rendering.</dd>
  <dt>`displayName`: string</dt>
  <dd>The name of your component to be displayed in React DevTools. Defaults to 'Query'.</dd>
  <dt>`skip`: boolean</dt>
  <dd>If skip is true, the query will be skipped entirely.</dd>
  <dt>`onCompleted`: (data: TData | {}) => void</dt>
  <dd>A callback executed once your query successfully completes.</dd>
  <dt>`onError`: (error: ApolloError) => void</dt>
  <dd>A callback executed in the event of an error.</dd>
  <dt>`context`: Record<string, any></dt>
  <dd>Shared context between your Query component and your network interface (Apollo Link). Useful for setting headers from props or sending information to the `request` function of Apollo Boost.</dd>
  <dt>`partialRefetch`: boolean</dt>
  <dd>If `true`, perform a query `refetch` if the query result is marked as being partial, and the returned data is reset to an empty Object by the Apollo Client `QueryManager` (due to a cache miss). The default value is `false` for backwards-compatibility's sake, but should be changed to true for most use-cases.</dd>
</dl>

<h3 id="render-prop">Render prop function</h3>

The render prop function that you pass to the `children` prop of `Query` is called with an object (`QueryResult`) that has the following properties. This object contains your query result, plus some helpful functions for refetching, dynamic polling, and pagination.

<dl>
  <dt>`data`: TData</dt>
  <dd>An object containing the result of your GraphQL query. Defaults to an empty object.</dd>
  <dt>`loading`: boolean</dt>
  <dd>A boolean that indicates whether the request is in flight</dd>
  <dt>`error`: ApolloError</dt>
  <dd>A runtime error with `graphQLErrors` and `networkError` properties</dd>
  <dt>`variables`: { [key: string]: any }</dt>
  <dd>An object containing the variables the query was called with</dd>
  <dt>`networkStatus`: NetworkStatus</dt>
  <dd>A number from 1-8 corresponding to the detailed state of your network request. Includes information about refetching and polling status. Used in conjunction with the `notifyOnNetworkStatusChange` prop.</dd>
  <dt>`refetch`: (variables?: TVariables) => Promise<ApolloQueryResult></dt>
  <dd>A function that allows you to refetch the query and optionally pass in new variables</dd>
  <dt>`fetchMore`: ({ query?: DocumentNode, variables?: TVariables, updateQuery: Function}) => Promise<ApolloQueryResult></dt>
  <dd>A function that enables [pagination](../features/pagination.html) for your query</dd>
  <dt>`startPolling`: (interval: number) => void</dt>
  <dd>This function sets up an interval in ms and fetches the query each time the specified interval passes.</dd>
  <dt>`stopPolling`: () => void</dt>
  <dd>This function stops the query from polling.</dd>
  <dt>`subscribeToMore`: (options: { document: DocumentNode, variables?: TVariables, updateQuery?: Function, onError?: Function}) => () => void</dt>
  <dd>A function that sets up a [subscription](../advanced/subscriptions.html). `subscribeToMore` returns a function that you can use to unsubscribe.</dd>
  <dt>`updateQuery`: (previousResult: TData, options: { variables: TVariables }) => TData</dt>
  <dd>A function that allows you to update the query's result in the cache outside the context of a fetch, mutation, or subscription</dd>
  <dt>`client`: ApolloClient</dt>
  <dd>Your `ApolloClient` instance. Useful for manually firing queries or writing data to the cache.</dd>
</dl>

<h2 id="next-steps">Next steps</h2>

Learning how to build `Query` components to fetch data is one of the most important skills to mastering development with Apollo Client. Now that you're a pro at fetching data, why not try building `Mutation` components to update your data? Here are some resources we think will help you level up your skills:

- [Mutations](./mutations.html): Learn how to update data with mutations and when you'll need to update the Apollo cache. For a full list of options, check out the API reference for `Mutation` components.
- [Local state management](./local-state.html): Learn how to query local data with `apollo-link-state`.
- [Pagination](../features/pagination.html): Building lists has never been easier thanks to Apollo Client's `fetchMore` function. Learn more in our pagination tutorial.
- [Query component video by Sara Vieira](https://youtu.be/YHJ2CaS0vpM): If you need a refresher or learn best by watching videos, check out this tutorial on `Query` components by Sara!
