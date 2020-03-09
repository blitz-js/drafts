# The Blitz.js Manifesto

## Background

Technology follows a repeating cycle of bundling and unbundling. Created in 2005, Ruby on Rails became a major bundling force, making web application development easier and more accessible than ever before. This benefited everyone, from those learning programming to seniors building production systems.

A major unbundling happened in 2013 with the release of React, a hyper focused rendering layer without any opinions for things like styling, state management, and routing. As React grew in popularity, so did the choices for all the other parts of an app, leaving developers with hundreds of decisions to make when starting a new app. While this has contributed to JavaScript Fatigue, it's been a powerful driving force for rapid frontend innovation.

Now, in 2020, is the perfect time for another major bundling. Developers are yearning for an easier, simpler way to build web applications. Beginners want a guiding hand for building a robust app, and seniors want a framework that removes mundane tasks while providing a highly scalable architecture with all the right escape hatches.

Hence the creation of Blitz.

## What is Blitz For?

Blitz is for building tiny to large database-backed web applications (and in the future, mobile apps). It's not for building extremely large web apps, like Facebook.com. It's not for building content sites, although you can easily add fully static pages to a Blitz app so you don't need a separate host for your marketing site.

## Foundational Principles

1. Fullstack & Monolithic
2. API Optional
3. Convention over Configuration
4. Loose Opinions
5. Easy to Start, Easy to Scale
6. Stability
7. Community over Code

### 1. Fullstack & Monolithic

A fullstack, monolithic application is simpler than an application where frontend and backend are developed and deployed separately. Monolithic doesn't mean it will be slow or hard to scale to large teams. Monolithic doesn't mean there isn't separation of concerns. Monolithic means you can reason about your app as a single entity.

### 2. API Optional

Choosing React for your view layer should not force you to build a backend API. Usually the only reason you need an API is to get data to your frontend, unless you need a public API or a mobile app (Blitz will later add no-API support for mobile). It's extremely expensive to have an API only for your frontend because it adds a lot of unnecessary complexity, making development slower, maintenance harder, and deployment more complex.

If at some point you actually do need an API, you can easily add a GraphQL API with auto generated resolvers. Or if REST is your jam, you can add that to your Blitz app instead.

### 3. Convention over Configuration

When starting a new app, you should be able to immediately start developing core app features instead of spending days to configure eslint, prettier, husky git hooks, jest, cypress, typescript, deciding on a file structure, setting up a database, adding authentication and authorization, setting up a router, defining routing conventions, setting up your styling library, and all the other menial tasks for initial app setup.

Blitz will make as many decisions and do as much work for you as possible. This makes it lightning fast to start real development. It also greatly benefits the community. Common project structure and architectural patterns make it easy to move from Blitz app to Blitz app and immediately feel at home.

Convention over configuration doesn't mean no configuration. It means configuration is optional. Blitz will provide all the escape hatches and configuration options you need for bespoke customization.

### 4. Loose Opinions

Blitz is opinionated. The out-of-the-box experience guides you on a path perfect for most applications. However, Blitz isn't arrogant. It totally understands there are very good reasons for deviating from convention, and it allows you to do so easily. For example, Blitz has a conventional file structure, but, with a few exceptions, doesn't _enforce_ it.

And when there's not consensus among the React community for a thing, `blitz new` will prompt you to choose the approach you want, such as Tailwind CSS, Theme UI, or styled-components.

### 5. Easy to Start, Easy to Scale

A framework that's only easy for one end of an application lifecycle is not a good framework. Both starting and scaling must be easy.

Easy to start includes being easy for beginners and being easy to migrate existing Next.js apps to Blitz.

Scaling is important in all forms: lines of code, number of people working in the codebase, and code execution in production.

### 6. Stability

In the fast-paced world of Javascript, a stable, predictable release cycle is a breath of fresh air. [Ember is the model citizen](https://emberjs.com/releases/) in this regard, and Blitz will follow their lead. Ember strictly follows SemVer with stable releases every 6 weeks and LTS releases every 6 months.

A stable release cycle ensures minimal breaking changes and ensures you know exactly what and when a breaking change will occur. It also minimizes bugs in stable releases by ensuring features are in beta for a minimum amount of time.

Blitz will follow a public RFC (request for comments) process so all users and companies can participate in proposing and evaluating new features.

If a Blitz API needs to be removed, it will be deprecated in a minor release. Major releases will simply remove APIs already deprecated in a previous release.

### 7. Community over Code

The Blitz community is the most important aspect of the framework, by far.
We have a comprehensive [Code of Conduct](https://github.com/blitz-js/blitz/blob/canary/CODE_OF_CONDUCT.md). LGBTQ+, women, and minorities are especially welcome.

We are all in this together, from the youngest to the oldest. We are all more similar than we are different. We can and should solve problems together. We should learn from other communities, not compete against them.