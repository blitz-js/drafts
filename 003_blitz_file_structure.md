# [RFC] Blitz File Structure & Routing

**The purpose of this RFC is to gain consensus on the Blitz file structure and routing.**

**We welcome all feedback, whether good or bad! This is your chance to ensure Blitz meets the needs of your company or project.**

<hr/>


## File Structure

Blitz is defining a standard file structure to answer the age old question of "How should I organize my react app files?"

#### Guiding Principles

1. Files that change together should live together.
2. Minimal requirements, maximum flexibility


```
├── app/
│   ├── components/
│   │   ├── Button.js
│   │   ├── Image.js
│   │   ├── Input.js
│   │   ├── Link.js
│   │   └── Text.js
│   ├── marketing/
│   │   ├── components/
│   │   │   ├── FeatureSection.js
│   │   │   ├── Footer.js
│   │   │   ├── Header.js
│   │   │   ├── Screenshot.js
│   │   │   └── Testimonial.js
│   │   └── routes/
│   │       ├── about.js
│   │       ├── features.js
│   │       ├── index.js
│   │       └── pricing.js
│   ├── projects/
│   │   ├── components/
│   │   │   ├── Project.js
│   │   │   ├── ProjectForm.js
│   │   │   └── Projects.js
│   │   ├── mutations/
│   │   │   ├── createProject.js
│   │   │   ├── createProject.test.js
│   │   │   ├── deleteProject.js
│   │   │   ├── deleteProject.test.js
│   │   │   ├── updateProject.js
│   │   │   └── updateProject.test.js
│   │   ├── queries/
│   │   │   ├── getProject.js
│   │   │   └── getProjects.js
│   │   └── routes/
│   │       └── projects/
│   │           ├── [id]/
│   │           │   └── edit.js
│   │           ├── [id].js
│   │           ├── index.js
│   │           └── new.js
│   ├── routes/
│   │   ├── api/
│   │   │   └── stripe-webhook.js
│   │   ├── dashboard.js
│   │   ├── log-in.js
│   │   ├── settings.js
│   │   └── sign-up.js
│   ├── tasks/
│   │   ├── components/
│   │   │   ├── Task.js
│   │   │   ├── TaskForm.js
│   │   │   └── Tasks.js
│   │   ├── mutations/
│   │   │   ├── createTask.js
│   │   │   ├── deleteTask.js
│   │   │   └── updateTask.js
│   │   ├── queries/
│   │   │   ├── getTask.js
│   │   │   └── getTasks.js
│   │   └── routes/
│   │       └── projects/
│   │           └── [projectId]/
│   │               └── tasks/
│   │                   ├── [id]/
│   │                   │   └── edit.js
│   │                   ├── [id].js
│   │                   ├── index.js
│   │                   └── new.js
│   └── tests/
│       ├── userAuthentication.js
│       └── userOnboarding.js
├── blitz.config.js
├── db/
│   ├── index.js
│   ├── migrations/
│   └── schema.prisma
├── integrations/
├── jobs/
├── layouts/
│   ├── Authenticated.js
│   └── Public.js
├── public/
│   └── favicon.ico
├── routes/
└── utils/
```

#### `app`

Contains all your core application code.

The file structure nested inside `app` can be anything you want, except for a few special folders detailed below.

Typically you will have two types of "container" directories inside `app`: **entity** directories like `projects/` and `tasks/` and **context** directories like `marketing/` or `admin/`.

##### Special Folders

Special Blitz folders can exist at any level of the hierarchy inside `app`.

- `routes/` and `routes/api/` are for exposing a React component or API handler at a specific URL. Routing inside these folders is always from the application root. These folders follow the same semantics as the Next.js `pages/` directory, just with a better name.
- `queries/` and `mutations/` are for your Blitz queries and mutations. Each query and mutation is exposed at a URL corresponding to its file path.

#### `db`

Contains database configuration, schema, and migration files. `db/index.js` exports your initialized database client for easy use throughout your app.

#### `integrations`

Contains third-party integration code. Ex: `auth0.js`, `twilio.js`, etc. These files are a good place to instantiate a client with shared configuration.

#### `jobs`

Asynchronous background job processing is TBD, but processing logic will live here.

#### `layouts`

Contains top level layout components. These will typically define your app shell and top level navigation menus. Details surrounding these are TBD.

#### `routes`

This directory is optional. While you can place any route files here, we recommend placing route files inside `app`.

This has the same semantics as the Next.js `pages` folder, just with a better name. All files in here are mapped to the url corresponding to their file paths.

Files in `routes/api` are exposed as API endpoints.

#### `public`

All files in here are served statically from your app's root URL

#### `utils`

Contains all those pesky little files and functions every project accumulates

#### `blitz.config.js` 

A configuration file with the same format as `next.config.js`


### Other Notes

- All top level folders are automatically aliased. So for example you can import from `app/projects/queries/getProject` from anywhere in our app.
- The Blitz CLI will have a `routes` command that makes it easy to see a full aggregated view of all your app routes.


## Routing

Blitz uses the [file-system based router provided by Next.js](https://nextjs.org/docs/routing/introduction).

- All components in `routes/` are mapped to a URL.
- All http handlers in `routes/api` are mapped to a URL.
- Queries and mutations are automatically exposed as an API endpoint
  - The `app/projects/queries/getProjects.js` query will be exposed at `/api/projects/queries/getProjects`

### Conventions

We copied the conventions from Ruby on Rails, where it has stood the test of time. The Blitz CLI will use these conventions for code scaffolding. If you don't like them, you are free to deviate and do anything you want.

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
