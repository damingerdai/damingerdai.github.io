# 记录一次迁移Apollo Server V3的过程

## 前言

[Apollo Server V3](https://www.apollographql.com/docs/apollo-server/)出来也快半年了，是时候把[express-postgres-ts-starter](https://github.com/damingerdai/express-postgres-ts-starter)的graphql升级了。

## 使用dependabot帮助更新版本

[dependabot](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates)是一个github的工具(似乎也支持gitlab,但是我不确定)，用于检测repo依赖安全性，同时也可以帮助我定期更新repo的依赖版本。

这是我的dependabot的配置文件：

```
version: 2
updates:
 - package-ecosystem: npm
   directory: '/'
   schedule:
    interval: weekly
   open-pull-requests-limit: 10
```

## 升级Apollo Servier所需要依赖

### nodejs

`Apollo Servier 3`仅仅支持nodejsv12以上版本(`Apollo Servier 2`则只需要nodejsv6以上的支持)。 因此需要升级到nodejs12，推荐使用node14和node16。

> 我十分推荐使用Linux/MAC用户使用[nvm](https://github.com/nvm-sh/nvm),Windows用户使用[nvm-windows](https://github.com/coreybutler/nvm-windows)安装、升级node版本。

### graphql

`Apollo Servier`有一个可选依赖[graphql](https://www.npmjs.com/package/graphql)(GraphQL JS的核心实现),`Apollo Servier 3`需要`graphql` v15.3.0以上的支持。

## 问题记录

### GraphQL Playground

`Apollo Server 2`是默认支持[GraphQL Playground](https://github.com/graphql/graphql-playground)，我们只需要在构造函数里配置好*playground*这个字段就好了，但是`Apollo Server 3`删除了对GraphQL Playgroun的默认支持，转而推荐在非生产环境中使用[Apollo Sandbox](https://www.apollographql.com/blog/announcement/platform/apollo-sandbox-an-open-graphql-ide-for-local-development/)。

不过我们还是可以重新配置`GraphQL Playground`的。

如果之前是使用`new ApolloServer({playground: boolean})`的类似方式配置`GraphQL Playground`，那么可以

```typescript
import { ApolloServerPluginLandingPageGraphQLPlayground,
         ApolloServerPluginLandingPageDisabled } from 'apollo-server-core';
new ApolloServer({
  plugins: [
    process.env.NODE_ENV === 'production'
      ? ApolloServerPluginLandingPageDisabled()
      : ApolloServerPluginLandingPageGraphQLPlayground(),
  ],
});

```

如果之前是使用`new ApolloServer({playground: playgroundOptions})`的类似方式配置`GraphQL Playground`，那么可以使用:

```typescript
import { ApolloServerPluginLandingPageGraphQLPlayground } from 'apollo-server-core';

const playgroundOptions = {
    // 仅做参考
    settings: {
        'editor.theme': 'dark',
        'editor.cursorShape': 'line'
    }
}

new ApolloServer({
  plugins: [
    ApolloServerPluginLandingPageGraphQLPlayground(playgroundOptions),
  ],
});
```

### tracing

在`Apollo Server 2`中，构造函数的参数提供*tracing*布尔字段，用于开启基于(apollo-tracing)[https://www.npmjs.com/package/apollo-tracing]跟踪机制，但是很遗憾，在`Apollo Server 3`中，*tracing*已经被删除了，，，`apollo-tracing`也已经被废弃了，如果一定要使用，可以：

```typescript
new ApolloServer({
  plugins: [
    require('apollo-tracing').plugin()
  ]
});

```

> 不过值得注意的是，该解决方案没有经过严格测试，可能存在bug。

### You must `await server.start()` before calling `server.applyMiddleware()`

在`Apollo Server v2.22`中提供了_server.start()_的方法，其目的是为了方便集成非serverless的框架(Express、Fastify、Hapi、Koa、Micro 和 Cloudflare)。因此这些框架的使用者使用在创建`ApolloServer`对象之后立刻启动graphql服务。

```javascript
const app = express();
const server = new ApolloServer({...});
await server.start();
server.applyMiddleware({ app });
```

## 结束

现在可以在浏览器打开`GraphQL Playground`, 以[express-postgres-ts-starter](https://github.com/damingerdai/express-postgres-ts-starter)为例，使用`http://127.0.0.1:3000/graphql`就可以看到效果了。