---
title: 认证
---

除非您加载的所有数据都是完全公开的，否则您的应用程序应该拥有用户，帐户和权限系统。 如果不同的用户在您的应用程序中具有不同的权限，那么您需要一种方法来告诉服务器哪个用户与每个请求相关联。

Apollo客户端使用超灵活的 [Apollo Link](/docs/link)，其中包括几个身份验证选项。


## Cookie

如果您的应用程序是基于浏览器的，并且您使用 cookie 进行后端登录和会话管理，则很容易告诉您的网络接口，发送 cookie 到每个请求。 您只需要传递凭证选项。 例如 `credentials：'same-origin'` 如果你的后端服务器是同一个域，或者如果你的后端是一个不同的域，则为 `credentials：'include'`。

```js
const link = createHttpLink({
  uri: '/graphql',
  credentials: 'same-origin'
});

const client = new ApolloClient({
  cache: new InMemoryCache(),
  link,
});
```

这个选项简单地传递给 HttpLink 在发送查询时使用的 [`fetch` implementation](https://github.com/github/fetch)

注意：后端还必须允许来自请求来源的凭据。 例如 如果在 node.js 中使用来自 npm 的流行 'cors' 包，则以下设置将与上述 apollo 客户端设置协同工作，
```js
// enable cors
var corsOptions = {
  origin: '<insert uri of front-end domain>',
  credentials: true // <-- REQUIRED backend setting
};
app.use(cors(corsOptions));
```
## Header

使用 HTTP 时识别自己的另一种常用方法是基于 header 发送认证。通过将 Apollo Links 结合在一起，可以轻松地为每个 HTTP 请求添加一个 `authorization` 标头。 在这个例子中，每次发送请求时我们都会从 `localStorage` 中提取登录令牌：

```js
import { ApolloClient } from 'apollo-client';
import { createHttpLink } from 'apollo-link-http';
import { setContext } from 'apollo-link-context';
import { InMemoryCache } from 'apollo-cache-inmemory';

const httpLink = createHttpLink({
  uri: '/graphql',
});

const authLink = setContext((_, { headers }) => {
  // get the authentication token from local storage if it exists
  const token = localStorage.getItem('token');
  // return the headers to the context so httpLink can read them
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
    }
  }
});

const client = new ApolloClient({
  link: authLink.concat(httpLink),
  cache: new InMemoryCache()
});
```

请注意，上面的示例使用了 `apollo-client` 包中的 `ApolloClient`。 虽然可以使用 `apollo-boost` 包中的 `ApolloClient` 修改 Headers，但由于 `apollo-boost` 不允许修改它所使用的 `HttpLink` 实例，所以 Headers 必须作为配置参数传入。 有关详细信息，请参阅 Apollo Boost [配置选项](../essentials/get-started.html#configuration) 部分。

服务器可以使用 header 信息对用户进行身份验证并将其附加到 GraphQL 执行上下文，因此解析器可以根据用户的角色和权限修改其行为。

<h2 id="login-logout">注销时重置存储</h2>

由于 Apollo 会缓存您的所有查询结果，因此在登录状态更改时删除它们非常重要。

确保 UI 和存储状态反映的是当前用户权限的最简单方法是在登录或注销过程完成后调用 `client.resetStore()`。 这将导致清除存储并重新获取所有查询。 如果您只想清除存储并且不想重新获取查询，请使用 `client.clearStore()`。 另一种选择是重新加载页面，产生类似的效果。


```js
const PROFILE_QUERY = gql`
  query CurrentUserForLayout {
    currentUser {
      login
      avatar_url
    }
  }
`;

const Profile = () => (
  <Query query={PROFILE_QUERY} fetchPolicy="network-only">
    {({ client, loading, data: { currentUser } }) => {
      if (loading) {
        return <p className="navbar-text navbar-right">Loading...</p>;
      }
      if (currentUser) {
        return (
          <span>
            <p className="navbar-text navbar-right">
              {currentUser.login}
              &nbsp;
              <button
                onClick={() => {
                  // call your auth logout code then reset store
                  App.logout().then(() => client.resetStore());
                }}
              >
                Log out
              </button>
            </p>
          </span>
        );
      }
      return (
        <p className="navbar-text navbar-right">
          <a href="/login/github">Log in with GitHub</a>
        </p>
      );
    }}
  </Query>
);
```
