# RFC 002: Blitz Routing

## Summary

This is a proposal for the standard Blitz routing convention.

## Problem

Most web apps have very similar routing requirements. Currently, every React team comes up with their own solution. It's easier for developers to have a robust convention to follow. It's also good for the ecosystem because higher level abstractions can be built on top of standard conventions.

## Solution

This routing convention is easy to follow and, when needed, easy to break out of. We copied it from Ruby on Rails, where it has stood the test of time.

Blitz uses the [file-system based router provided by Next.js](https://nextjs.org/docs/routing/introduction).

### Pages

Say you have a model named `Project`. Your routes will be:

| HTTP | Controller Action        | Path                | File                        |
| ---- | ------------------------ | ------------------- | --------------------------- |
| GET  | ProjectsController.index | /projects           | pages/projects/index.js     |
| GET  |                          | /projects/new       | pages/projects/new.js       |
| GET  | ProjectsController.show  | /projects/[id]      | pages/projects/[id].js      |
| GET  | ProjectsController.show  | /projects/[id]/edit | pages/projects/[id]/edit.js |

And if you also have a `Task` models that belong to projects, your routes will be:

| HTTP | Controller Action        | Path                                  | File                                          |
| ---- | ------------------------ | ------------------------------------- | --------------------------------------------- |
| GET  | ProjectsController.index | /projects                             | pages/projects/index.js                       |
| GET  |                          | /projects/new                         | pages/projects/new.js                         |
| GET  | ProjectsController.show  | /projects/[id]                        | pages/projects/[id].js                        |
| GET  | ProjectsController.show  | /projects/[id]/edit                   | pages/projects/[id]/edit.js                   |
| GET  | TasksController.index    | /projects/[projectId]/tasks           | pages/projects/[projectId]/tasks/index.js     |
| GET  |                          | /projects/[projectId]/tasks/new       | pages/projects/[projectId]/tasks/new.js       |
| GET  | TasksController.show     | /projects/[projectId]/tasks/[id]      | pages/projects/[projectId]/tasks/[id].js      |
| GET  | TasksController.show     | /projects/[projectId]/tasks/[id]/edit | pages/projects/[projectId]/tasks/[id]/edit.js |

### API

Each model has a controller. Blitz turns each controller into an http request handler.

For `Project` and `Task` as started above, the API routes will be:

| HTTP   | Controller Action         | Path                                 | File                                            |
| ------ | ------------------------- | ------------------------------------ | ----------------------------------------------- |
| GET    | ProjectsController.index  | /api/projects                        | pages/api/projects/[...id].js                   |
| GET    | ProjectsController.show   | /api/projects/[id]                   | pages/api/projects/[...id].js                   |
| POST   | ProjectsController.create | /api/projects                        | pages/api/projects/[...id].js                   |
| PATCH  | ProjectsController.update | /api/projects/[id]                   | pages/api/projects/[...id].js                   |
| DELETE | ProjectsController.delete | /api/projects/[id]                   | pages/api/projects/[...id].js                   |
|        |                           |                                      |                                                 |
| GET    | TasksController.index     | /api/tasks                           | pages/api/tasks/[...id].js                      |
| GET    | TasksController.show      | /api/tasks/[id]                      | pages/api/tasks/[...id].js                      |
| POST   | TasksController.create    | /api/tasks                           | pages/api/tasks/[...id].js                      |
| PATCH  | TasksController.update    | /api/tasks/[id]                      | pages/api/tasks/[...id].js                      |
| DELETE | TasksController.delete    | /api/tasks/[id]                      | pages/api/tasks/[...id].js                      |
|        |                           |                                      |                                                 |
| GET    | TasksController.index     | /api/projects/[projectId]/tasks      | pages/api/projects/[projectId]/tasks/[...id].js |
| GET    | TasksController.show      | /api/projects/[projectId]/tasks/[id] | pages/api/projects/[projectId]/tasks/[...id].js |
| POST   | TasksController.create    | /api/projects/[projectId]/tasks      | pages/api/projects/[projectId]/tasks/[...id].js |
