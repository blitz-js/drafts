# RFC 002: Blitz Routing

## Summary

This is a proposal for the standard Blitz routing convention.

## Problem

Most web apps have very similar routing requirements. Currently, every React team comes up with their own solution. It's easier for developers to have a robust convention to follow. It's also good for the ecosystem because higher level abstractions can be built on top of standard conventions.

## Solution

This routing convention is easy to follow and, when needed, easy to break out of. We almost directly copied it from Ruby on Rails, where it has stood the test of time.

### Pages

Say you have a model named `Project`. Your routes will be:

| Path                | File                        |
| ------------------- | --------------------------- |
| /projects           | pages/projects/index.js     |
| /projects/new       | pages/projects/new.js       |
| /projects/[id]      | pages/projects/[id].js      |
| /projects/[id]/edit | pages/projects/[id]/edit.js |

And if you also have `Task` models that belong to projects, your routes will be:

| Path                           | File                                   |
| ------------------------------ | -------------------------------------- |
| /projects                      | pages/projects/index.js                |
| /projects/new                  | pages/projects/new.js                  |
| /projects/[id]                 | pages/projects/[id].js                 |
| /projects/[id]/edit            | pages/projects/[id]/edit.js            |
| /projects/[id]/tasks           | pages/projects/[id]/tasks/index.js     |
| /projects/[id]/tasks/new       | pages/projects/[id]/tasks/new.js       |
| /projects/[id]/tasks/[id]      | pages/projects/[id]/tasks/[id].js      |
| /projects/[id]/tasks/[id]/edit | pages/projects/[id]/tasks/[id]/edit.js |

### API

Each model has a controller. Blitz turns each controller into an http request handler.

For a `ProjectsController`, the API routes will be:

| HTTP Verb | Controller Action | Path               | File                          |
| --------- | ----------------- | ------------------ | ----------------------------- |
| GET       | index             | /api/projects      | pages/api/projects/[...id].js |
| GET       | show              | /api/projects/[id] | pages/api/projects/[...id].js |
| POST      | create            | /api/projects      | pages/api/projects/[...id].js |
| PATCH     | update            | /api/projects/[id] | pages/api/projects/[...id].js |
| DELETE    | delete            | /api/projects/[id] | pages/api/projects/[...id].js |
