# RFC 001: Blitz CLI

## Summary

The Blitz CLI is a single, unified interface to everything you need during development. It's an interface that's common across all blitz projects. This makes it easy to switch to another blitz app and immediately be productive.

## Problem

Modern React app development uses many disparate tools. Making them all work well together is often tedious. And even when they are working well, the interface for using them is often different for each project. I.e. "Does this project use `yarn start` or `yarn dev`...??"

## Solution

Blitz provides a nicely designed CLI that's heavily inspired by the Ruby on Rails CLI. For those who aren't on a first name basis with CLIs, Blitz will also provide a fully featured GUI.

### Installation & Usage

- You install `blitz` globally **or** you can use npx: `npx blitz [command]`
- A Blitz app will have `blitz` installed as a dependency
- When the blitz cli is used inside a blitz project, it will proxy commands to `node_modules/.bin/blitz`. This is critical so the blitz command is always tied to the version installed in the project, not the version of the global CLI package.

### Commands

#### `blitz new [project-name]`

Creates a new Blitz app with everything you need for app development. This goes way beyond `create-next-app` including file structure, installed packages, configuration etc.

It will prompt you to select from a few options, like Javascript vs Typescript and npm vs yarn. Eventually it'll support more options like Tailwind CSS or Theme UI.

#### `blitz start`

Starts the Next.js development server

#### `blitz db migrate`

Run any database migrations, if needed

#### `blitz console`

Start a Node.js REPL with your application code automatically loaded into scope. Just like `rails console`

#### `blitz generate`

Generate models, controllers, pages, etc.

For example, the following would generate the user model, user controller, all the standard user pages and components, and the user API route harnesses.

```
blitz generate resource user firstName:text email:text
```

#### `blitz test`

Run all the tests

#### `blitz test watch`

Run all the tests in watch mode to rerun tests anytime files change

#### `blitz run [script-name]`

Run the script at `scripts/[script-name].js` with the same context environment as `blitz console`. This is the equivalent to Rails rake tasks.
