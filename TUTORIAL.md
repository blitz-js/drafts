# Blitz Tutorial

← [Back to Alpha Guide](https://github.com/blitz-js/blitz/blob/canary/USER_GUIDE.md)

Thanks for trying out Blitz! In this tutorial, we’ll walk you through the creation of a basic voting application.

We’ll assume that you have [Blitz installed](https://github.com/blitz-js/blitz/blob/canary/USER_GUIDE.md#blitz-app-development) already. You can tell if Blitz is installed, and which version you have by running the following command in your terminal:

```sh
$ blitz -v
```

If Blitz is installed, you should see the version of your installation. If it isn’t, you’ll get an error saying something like “command not found: blitz”.

## Creating a project

If this is your first time using Blitz, you’ll have to begin with some initial setup. We provide a command which takes care of all this for you, generating the configuration and code you need to get started.

From the command line, `cd` into the directory where you’d like to store your code, and then run the following command:

```sh
blitz new mysite
```

This should create a `mysite` directory in your current directory.

Let’s look at what `blitz new` created:

```
mysite/
  app/
    components/
      ErrorBoundary.tsx
    layouts/
    pages/
      _app.tsx
      _document.tsx
      index.tsx
  db/
  integrations/
  jobs/
  node_modules/
  public/
  utils/
  .babelrc.js
  .env
  .eslintrc.js
  .gitignore
  .npmrc
  .prettierignore
  blitz.config.js
  package.json
  README.md
  tsconfig.json
```

**Explanation of each file**

## The development server

Let’s check that your Blitz project works. Make sure you are in the `mysite` directory, if you haven’t already, and run the following command:

```sh
$ blitz start
```

You’ll see the following output on the command line:

```sh
✔ Prepped for launch
[ wait ]  starting the development server ...
[ info ]  waiting on http://localhost:3000 ...
[ info ]  bundled successfully, waiting for typecheck results...
[ wait ]  compiling ...
[ info ]  bundled successfully, waiting for typecheck results...
[ ready ] compiled successfully - ready on http://localhost:3000
```

Now that the server’s running, visit http://localhost:3000/ with your Web browser. You’ll see a welcome page, with the Blitz logo. It worked!

## Creating the Voting app

Now that your development environment—a “project” is set up, you’re ready to start building out the app.
