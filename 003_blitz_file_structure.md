# [RFC] Blitz File Structure & Routing Conventions

**The purpose of this RFC is to gain consensus on the Blitz file structure and routing.**

**We welcome all feedback, whether good or bad! This is your chance to ensure Blitz meets the needs of your company or project.**

<hr/>


## File Structure

```
├── app
│   ├── projects
│   │   ├── components
│   │   │   ├── Project.js
│   │   │   ├── ProjectForm.js
│   │   │   └── Projects.js
│   │   ├── mutations
│   │   │   ├── createProject.js
│   │   │   ├── deleteProject.js
│   │   │   └── updateProject.js
│   │   └── queries
│   │       ├── getProject.js
│   │       └── getProjects.js
│   └── tasks
│       ├── components
│       │   ├── Task.js
│       │   ├── TaskForm.js
│       │   └── Tasks.js
│       ├── mutations
│       │   ├── createTask.js
│       │   ├── deleteTask.js
│       │   └── updateTask.js
│       └── queries
│           ├── getTask.js
│           └── getTasks.js
├── blitz.config.js
├── db
│   ├── index.js
│   ├── migrations
│   └── schema.prisma
├── integrations
├── jobs
├── layouts
│   ├── Authenticated.js
│   └── Public.js
├── public
│   └── favicon.ico
├── routes
│   ├── about.js
│   ├── api
│   │   └── stripe-webhook.js
│   ├── features.js
│   ├── index.js
│   ├── log-in.js
│   ├── pricing.js
│   ├── projects
│   │   ├── [id]
│   │   │   └── edit.js
│   │   ├── [id].js
│   │   ├── [projectId]
│   │   │   └── tasks
│   │   │       ├── [id]
│   │   │       │   └── edit.js
│   │   │       ├── [id].js
│   │   │       ├── index.js
│   │   │       └── new.js
│   │   ├── index.js
│   │   └── new.js
│   └── sign-up.js
├── tests
└── utils

```

- All top level folders are automatically aliased. So for example you can import from `app/projects/queries/getProject` from anywhere in our app.

#### `app`

Contains all your core application code including queries, mutations, routes, and tests. Folders can be nested and organized any way you want. For example, `images/` and `videos/` could be nested inside `media/`.

#### `db`

Contains database configuration, schema, and migration files. `db/index.js` exports your initialized database client for easy use throughout your app.

#### `integrations`

Contains third-party integration code. Ex: `auth0.js`, `twilio.js`, etc. These files are a good place to instantiate a client with shared configuration.

#### `jobs`

Asynchronous background job processing is TBD, but processing logic will live here.

#### `layouts`

Contains top level layout components. These will typically define your app shell and top level navigation menus. Details surrounding these are TBD.

#### `routes`

Same semantics as the Next.js `pages` folder, just with a better name. All files and directories in here are mapped to the url corrosponding to their file paths.

Files in `routes/api` should expose an HTTP handler function.

#### `public`

All files in here are served statically from your app's root URL

#### `tests`

Contains integration and end-to-end tests

#### `utils`

Contains all those pesky little files and functions every project accumulates

#### `blitz.config.js` 

A configuration file with the same format as `next.config.js`

## Routing Conventions

Blitz uses the [file-system based router provided by Next.js](https://nextjs.org/docs/routing/introduction).

We copied this convention from Ruby on Rails, where it has stood the test of time. The Blitz CLI will use these conventions for code scaffolding. If you don't like them, you are free to deviate and do anything you want.

- Entity names are plural
- Each of the following have their own page: entity index, single entity show page, new entity page, and edit entity page
- `id` is used from the dynamic url slug
- `entityId` is used for dynamic url slug of parent entities

Example: You have a `Project` model and a `Task` model which belongs to a `Project`. Your routes will be:

| Path                                  | File                                          |
| ------------------------------------- | --------------------------------------------- |
| /projects                             | app/projects/routes/projects/index.js          |
| /projects/new                         | app/projects/routes/projects/new.js                         |
| /projects/[id]                        | app/projects/routes/projects/[id].js                        |
| /projects/[id]/edit                   | app/projects/routes/projects/[id]/edit.js                   |
|                                       |                                               |
| /projects/[projectId]/tasks           | app/tasks/routes/projects/[projectId]/tasks/index.js     |
| /projects/[projectId]/tasks/new       | app/tasks/routes/projects/[projectId]/tasks/new.js       |
| /projects/[projectId]/tasks/[id]      | app/tasks/routes/projects/[projectId]/tasks/[id].js      |
| /projects/[projectId]/tasks/[id]/edit | app/tasks/routes/projects/[projectId]/tasks/[id]/edit.js |
