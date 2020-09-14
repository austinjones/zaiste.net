+++
date = 2020-09-14T00:00:00.000Z
title = "Lightweight, Modern React.js Setup for GraphQL using Vite and urql"
[taxonomies]
topics = ["React.js", "GraphQL", "JavaScript", "TypeScript"]
+++

In this tutorial, we will build a React.js application that interacts with a GraphQL endpoint. This method of project setup is modern and lightweight: it uses Hooks, ES Modules and has a small amount of dependencies. We will use [Vite](https://github.com/vitejs/vite) to create the project structure, [pnpm](https://pnpm.js.org/) to manage the dependencies, [urql](https://formidable.com/open-source/urql/) for GraphQL, and finally [OneGraph](https://www.onegraph.com/) as the GraphQL gateway to various APIs. Our goal is to create an application for listing GitHub repositories of a specific user.

## Install `pnpm`

Let’s start with `pnpm`, a package manager for JavaScript that is faster and more efficient than `npm` or `yarn`. `pnpm` uses a content-addressable filesystem to store your project dependencies. This way files inside `node_modules` are linked from a single place on your disk. Thus, you install each dependency only once and this dependency occupies the space on your disk only once. In other words, libraries are not copied over for each new project. This way, on top of being faster than alternatives, `pnpm` provides huge disk space gains.

```
npm i -g pnpm
```

`pnpm` comes with two command-line tools: `pnpm` for installing dependencies and `pnpx` for invoking commands (a `npx` equivalent).

## Create the project

Let’s use Vite to create the project structure for our React.js project. We need to initialize the project using the `vite-app` generator with the React as the template. The template must be set explicitly using the `--template` parameter with `react` as its value. Finally, `gh-explorer` is a custom name we are giving to this project.

```
pnpm init vite-app gh-explorer --template react
```

Vite is a build tool for web projects. It serves the code in development using ECMAScript Module imports. In production, vite bundles the code using Rollup. Vite is a lightweight solution that can be **100-150x times** faster than alternatives such as Webpack or Parcel. This enormous speed gain is possible thanks to [esbuild](https://github.com/evanw/esbuild), a new TypeScript/JavaScript bundler written using the Go programming language.

![esbuild benchmark](https://user-images.githubusercontent.com/200613/93084500-664aeb00-f694-11ea-8cff-372f5c0057b4.png)

Go inside the `gh-explorer` directory and install the necessary dependencies using the `pnpm install` command. Then, start the development server with `pnpm dev` and head to the `localhost:5000` in your browser. You should see a React.js logo along with a counter and a button.

![A React.js project generated by Vite](https://user-images.githubusercontent.com/200613/93084551-85497d00-f694-11ea-9285-ba2716582f7a.png)

## Integrate with OneGraph

When interacting with external APIs, we need to learn the specifics for each new API we are dealing with. This is especially visible at the level of authentication. The methods of authentication are slightly different between one API and another. Even though those APIs are provided either as REST or GraphQL endpoints, it takes time and often much effort to learn how to use them. Luckly, there is OneGraph. The project provides a layer of unification for various GraphQL APIs. Using OneGraph, we can just access one endpoint and gain access to various GraphQL APIs at once. Think, a catalog of APIs. This simplifies and speeds up the development. We will use OneGraph to interact with the GitHub API.

Let’s create an application in OneGraph:

![Creating an application in OneGraph](https://user-images.githubusercontent.com/200613/93085161-70211e00-f695-11ea-9c94-9306b2f358ca.gif)

Then, we can use OneGraph's Explorer to test our GraphQL queries for GitHub before we integrate them with our React.js application. On the left side of the Explorer I have a list of all available APIs. It goes from Airtable, Box to Shopify, Stripe, Zendesk and much more. This catalog is quite impressive on its own.

![OneGraph's Explorer](https://user-images.githubusercontent.com/200613/93085373-c5f5c600-f695-11ea-85be-44561245af2a.png)

## Construct the GraphQL Query

Our goal is to list the repositories of a specific user. I start by selecting the GitHub API. Then, I select the`user` branch. I enter the handle of a specific user, e.g. `zaiste` - in this case, it’s my own username. I go further down the GitHub API tree by selecting the `repositories` branch. I want to list only the public repositories that are not forks and ordered by the number of stars. For each repository, I want to return its `id`, `name` and the number of stars.

![OneGraph Explorer for GitHub in action](https://user-images.githubusercontent.com/200613/93086730-da3ac280-f697-11ea-8079-2ea256c9e36a.gif)

Just by clicking the fields in the OneGraph Explorer I end up with the following GraphQL query:

```graphql
query GitHubRepositories {
  gitHub {
    user(login: "zaiste") {
      repositories(
        first: 10
        orderBy: { field: STARGAZERS, direction: DESC }
        privacy: PUBLIC
        isFork: false
        affiliations: OWNER
      ) {
        nodes {
          id
          name
          stargazers(
            first: 10
            orderBy: {
              field: STARRED_AT
              direction: DESC
            }
          ) {
            totalCount
          }
        }
      }
    }
  }
}
```

## Integrate with urql

We can now execute this query from our React.js application. We will use urql, a versatile GraphQL client for React.js, Preact and Svelte. The project is lightweight and highly customizable compared to alternatives such as Apollo or Relay. Its API is simple and the library aims to be easy to use. We need to add `urql` along with the `graphql` as dependencies for our project.

```
pnpm add urql graphql
```

`urql` provides the `useQuery` hook. This function takes the GraphQL query as input, and returns the data along with errors and the fetching status as the result. We will name our component `RepositoryList`. You can use the regular `.jsx` extension, or `.tsx` if you plan to integrate with TypeScript - it will work out-of-the-box with Vite. There is no need for additional TypeScript configuration.

```tsx
export const RepositoryList = () => {
  const [result] = useQuery({ query });

  const { data, fetching, error } = result;

  if (fetching) return <p>Loading...</p>;
  if (error) return <p>Errored!</p>;

  const repos = data.gitHub.user.repositories.nodes;

  return (
    <ul>
      {repos.map(repo => (
        <li key={repo.id}>{repo.name} <small>({repo.stargazers.totalCount})</small></li>
      ))}
    </ul>
  );
}
```

Next, in `main.jsx` let’s configure our GraphQL client. We need the `Provider` component along with the `createClient` function from `urql`, and an instance of `OneGraphAuth`. For the latter, we need another dependency, i.e. `onegraph-auth`.

```
pnpm add onegraph-auth
```

Let’s create an instance of `OneGraphAuth` with the `appId` of the application we created using the OneGraph dashboard. Then, we create a GraphQL client with the OneGraph endpoint as the `url` parameter. Finally, we nest the `<App/>` component inside the `<Provider/>`.

```tsx
import React from 'react'
import { render } from 'react-dom'
import { createClient, Provider } from 'urql';
import OneGraphAuth from 'onegraph-auth';

import './index.css'
import App from './App'

const appId = "<Your APP_ID from OneGraph goes here>";

export const auth = new OneGraphAuth({ appId });

const client = createClient({
  url: 'https://serve.onegraph.com/dynamic?app_id=' + appId,
  fetchOptions: () => ({ headers: auth.authHeaders() })
});

render(
  <React.StrictMode>
    <Provider value={client}>
      <App />
    </Provider>
  </React.StrictMode>,
  document.getElementById('root')
)
```

## Authenticate with OneGraph

We are almost finished. The last step is to authenticate the user against the OneGraph endpoint. It’s a unified approach for any API from the OneGraph catalog. We will use the `.login` method from the `onegraph-auth` with `github` as the value. Once the user logs in, we will adjust the state accordingly by displaying the `<RepositoryList/>` component.

```tsx
import React, { useState, useEffect } from 'react'

import './App.css'
import { auth } from './main';
import { RepositoryList } from './RepositoryList';

function App() {
  const [isLoggedIn, setIsLoggedIn] = useState(false)

  const login = async () => {
    await auth.login('github');
    const isLoggedIn = await auth.isLoggedIn('github');

    setIsLoggedIn(isLoggedIn);
  }

  return (
    <div className="App">
      <header className="App-header">
        <p>GitHub Projects via OneGraph</p>
        <p>
          {isLoggedIn ? (
            <RepositoryList/>
          ) : (
            <button style={{fontSize: 18}} onClick={() => login()}>
              Login with YouTube
            </button>
          )}
        </p>
      </header>
    </div>
  )
}

export default App
```

## The Result

That’s all. Here’s the final result. You may need to adjust the stylesheets for the same visual effect.

![Final Result: List of GitHub repositories](https://user-images.githubusercontent.com/200613/93087084-5d5c1880-f698-11ea-90d2-46629c473450.png)

We created a React.js application using **Hooks**. The project has a **minimal set of dependencies**. It uses the modern **ECMASCript Modules** approach. It is **efficient in disk space** as it uses pnpm as the package manager. The JavaScript/TypeScript transpilation is **100-150x faster** than Webpack or Parcel. We use a simple and versatile GraphQL client called **urql**. Finally, we access the GitHub API via **OneGraph**, a meta API that provides an impressive catalog of GraphQL APIs with the unified access method. The end result is lightweight and modern.

I hope you will use some of those elements in your future React.js applications. If you liked the article, [follow me on Twitter](https://twitter.com/zaiste) for more.
