---
title: 与 React Native 集成
---

您可以像使用 React Web 一样使用 Apollo 和 React Native。

要将 Apollo 引入您的应用程序，请从 npm 安装 `react-apollo` 并在您的应用程序中使用它们，如 [安装指南](../essentials/get-started.html) 中所示:

```bash
npm install react-apollo --save
```

```js
import React from 'react';
import { AppRegistry } from 'react-native';
import { ApolloClient } from 'apollo-client';
import { ApolloProvider } from 'react-apollo';

// Create the client as outlined in the setup guide
const client = new ApolloClient();

const App = () => (
  <ApolloProvider client={client}>
    <MyRootComponent />
  </ApolloProvider>
);

AppRegistry.registerComponent('MyApplication', () => App);
```

如果您不熟悉 Apollo 和 React，您应该阅读 [React指南](../index.html)。

<h2 id="examples">示例</h2>

有一些用 React Native 编写的 Apollo 示例，您可能希望参考：

1. dev.apollodata.com上使用的 ["Hello World" example](https://github.com/apollographql/frontpage-react-native-app) 。
2. 构建一个 [GitHub API Example](https://github.com/apollographql/GitHub-GraphQL-API-Example) 以使用GitHub的新GraphQL API。

<h2 id="apollo-dev-tools">Apollo Dev Tools</h2>

[React Native Debugger](https://github.com/jhen0409/react-native-debugger) 支持 [Apollo Client Devtools](https://github.com/apollographql/apollo-client-devtools):

1. 安装 React Native Debugger 并打开它。
2. 在您的应用中启用 “远程调试JS”。
3. （可选）如果您没有看到 “开发人员工具” 面板或其中缺少 “Apollo” 选项卡，请通过右键单击任意位置并选择 “切换开发人员工具” 来切换开发人员工具。
