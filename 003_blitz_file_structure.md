# RFC 003: Blitz App File Structure

## Summary

This is a proposal for the file structure of a standard Blitz app.

## Problem

The file structure of a project is something that adds zero value to your app. Therefore time spent debating or managing this is truly a waste of time. Prettier eliminated time wasted on file formatting. Blitz is eliminating time wasted on file structure for React apps. Other frameworks like Ruby on Rails and Ember have always had ile structure conventions to great success. One of the biggest benefits is uniformity among projects. Any Blitz app will feel familiar, and you'll know eactly where to look when you need to find something.

## Solution

```
├── app/
│   └── users/
│       ├── components/
│       │   └── Form.js
│       ├── controller.js
│       ├── model.js
│       └── tests/
├── db/
│   ├── migrations/
│   └── schema.prisma
├── integrations/
├── jobs/
├── layouts/
│   ├── Authenticated.js
│   └── Public.js
├── next.config.js
├── pages/
│   └── api/
├── public/
│   └── favicon.ico
├── tests/
└── utils/
```

### `app`

Contains all domain models along with their model, controller, components, and tests. Models can be organized into groups. For example, `images/` and `videos/` could be nested inside `media/`.

### `db`

Contains database configuration, schema, and migration files

### `integrations`

Contains third-party integration code. Ex: `auth0.js`, `twilio.js`, etc. These files are a good place to instantiate a client with shared configuration.

### `jobs`

Async job processing is TBD, but processing logic will live here

### `layouts`

Contains top level layout components. These will typically define your app shell and top level navigation menus. How these are used is TBD. How these will get dyamic data, like current user name and current company name is also TBD.

### `pages`

Contains all your top level app pages

### `pages/api`

Contains all API handlers. For many apps, you will not write any custom code in here. All the files will be just a few lines of auto generated code for Blitz to automatically harness controllers to an HTTP handler.

However, you could of course add `graphql.js` and use Nexus to generate your resolvers.

### `public`

All files in here are served statically by Next.js from the root URL

### `tests`

Contains integration and end-to-end tests

### `utils`

Contains all those pesky little files and functions every project accumulates
