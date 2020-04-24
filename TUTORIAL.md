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

## Write your first page

Now that your development environment—a “project”—is set up, you’re ready to start building out the app. First, we’ll create your first page.

Open the file `app/pages/index.tsx` and put the following code in it:

```tsx
export default () => (
  <div>
    <h1>Hello, world!</h1>
  </div>
)
```

This is the simplest page possible in Blitz. To look at it, go back to your browser and go to http://localhost:3000. You should see your text appear! Try editing the `index.tsx` file, and make it your own! When you’re ready, move on to the next section.


## Database setup
Now, we’ll setup the database and create your first model.

Open up `db/schema.prisma`. It’s a configuration file which our default database engine Prisma uses.

By default, the configuration uses PostgresQL. If you’re new to databases, or you’re just interested in trying Blitz, this is a great choice. Make sure you have it installed on your machine, and have created a database by this point. Do that with “`CREATE DATABASE database_name;`” within your database’s interactive prompt.

If you’re looking for an even easier option, you can try using SQLite. Do this by replacing:

```
datasource postgresql {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

with:

```
datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"
}
```

Please not that using SQLite is totally optional, and may bring some other problems with it. When starting your first real project, you will likely want to use a more scalable database like PostgresQL, to avoid database-switching nightmares down the road.

## Creating models

Now we’ll define your models—essentially, your database layout, with additional metadata.

In `schema.prisma`, we’ll create two models: `Question`, and `Choice`. A `Question` has a question and a publication date. A `Choice` has two fields: the text of the choice and a vote count. Each has an id, and each `Choice` is associated with a `Question`.

Edit the `schema.prisma` file so it looks like this:

```
// (datasource and generator)

...

model Question {
  id          Int      @default(autoincrement()) @id
  text        String
  publishedAt DateTime
  choices     Choice[]
}

model Choice {
  id         Int      @default(autoincrement()) @id
  text       String
  votes      Int      @default(0)
  question   Question @relation(fields: [questionId], references: [id])
  questionId Int
}
```

Now, we need to migrate our database. This is a way of telling it that you have edited your schema in some way. Run this, and enter something like `init db` when prompted for a migration name:

```sh
$ blitz db migrate
```


## Playing with the API

Now, let’s hop into the interactive Blitz shell and play around with the free API Blitz gives you. To invoke the Blitz shell, use this command:

```sh
$ blitz console
```

Once you’re in the console, explore the Database API:



```sh
# No questions are in the system yet.
⚡ > await db.question.findMany()
[]

# Create a new Question.
⚡ > q = await db.question.create({data: {text: 'What’s new?', publishedAt: new Date()}})
{ id: 1, text: 'What’s new?', publishedAt: 2020-04-24T22:08:17.307Z }

# Now it has an ID.
⚡ > q.id
1

# Access model field values via JavaScript attributes.
⚡ > q.text
"What’s new?"

⚡ > q.publishedAt
2020-04-24T22:08:17.307Z

# Change values by using the update function
⚡ > q = await db.question.update({where: {id: 1}, data: {text: 'What’s up?'}})

# db.questions.findMany() displays all the questions in the database.
⚡ > await db.questions.findMany()
[
  { id: 1, text: 'What’s up?', publishedAt: 2020-04-24T22:08:17.307Z }
]
```

## Writing more pages

Let’s create some more pages. 
