# Blitz API

## First principles

The minimum conceptual things a developer needs to define an application are:

- State - Data structure for the app and it’s constraints
- View - How that data is presented
- Computation - How to manipulate the data as it travels between View | State | View. Also chron jobs etc.
- Orchestration - How these things are all connected

That leaves Computation and Orchestration

These are the key decisions our framework needs to make for people.

## View

An application's view takes application data and presents the application's data to the user. It also collects the user's feedback and forwards that feedback to the computation layer. The view is predominantly run on the client although it may prerender on the server.

View has the ability to hold certain types of state. View specific state includes the location of the user within the application the user session information as well as control state cache for ephemeral form information.

Next.js/React is an extremely simple way to manage _View_

## State

An application's state is persisted somewhere - those options might be varied and could theoretically include things like indexDB, localStorage or PouchDB on the client or the file system on the server however in most cases state will be persisted to a database on a server somewhere.

State management would comprise of any system that would abstract away access to persisted state including systems such as an ORM, Computed properties, database constraints system.

Prisma is becoming an increasingly promising and simple way to manage _State_

## Interactions

An Interaction is effectively a function with controlled side effects who's job it is to connect the caller to the data store and glue the application together.

Interactions can be classified into four kinds:

- Pure Function
- Pure Query
- Command/Mutation
- Pipe

A Pipe is an orchestration management structure it designates how functions should be sequenced. This is mainly useful for managing selecting data and returning data from commands.

An Interaction may be designed to execute on the server or the client and should mark itself as such as it is dependent on it's context. If an Interaction marks itself as both client and server then the caller should be able to stipulate where it wants the command to run or it should be able to override this behaviour with configuration which may help with offline enabled apps.

We need to differentiate the different types of computation because they have different execution requirements.

- Pure Functions are universal(client/server) and can be memoized with an infinite ttl.
- Pure Querys can be optimised automatically by the system based on load or explicitly by the user varying the ttl on a query by query basis.
- Commands will need to be executed in an asynchronous queue sequentially via a special command route in order to avoid data race conditions.

The design for commands needs to have some caveats in order to allow for good write performance:

- You can call Queries from Queries.
- You cannot call Commands from queries as that turns a query into a command.
- Commands can call queries to select the current state of data and perform a computation on it.
- Commands only return that they were accepted.

### Alternatives to functions

Controller classes could be considered an alternative to using functions however they have issues:

- Developer is required to organise the computations into namespaces - some may not be clear
- Classes as namespacing mechanisms tempt the developer to share state write state between methods - better to simply provide read state to a function.
- Classes provide a false mental model of how an application is executing as methods are used within the context of the class
- Added ceremony due to prescriptive namespacing

They have some benefits too

- Fewer files

## Orchestration

### View <-> Computation

Using React as the view layer the most idiomatic way to provide bidirectional "state" layer connectivity is to utilise a hooks based API. This has been achieved in many data-fetching contexts including Redux and Apollo. By doing this we can then leverage several advantages:

- Developers will instantly be familiar with the API.
- The developer's entrypoint to the framework is highly composable which makes blitz more interoperable.
- Blitz becomes much more portable.
- Hooks lead to simpler JSX.

Tradeoffs

- Hooks don't support legacy class based components.
- There is a slight SSR performance penalty with hooks as we need to render the react tree to string twice - we may be able to cache this at next build time which would remove this penalty.

### Computation <-> State

The Pipe Interactions should manage the orchestration of Interactionss. Pipe is a simplistic analogy for a Unix pipe and allows the output of one function to be piped to the input of the other or diverted to an error handler. Under the hood this allows us to know how to manage the functions we are given and becomes an abstraction point for an underlying event driven architecture.

Later we can add decision points to split control flow following a model such as that provided by RxJS partition perhaps, however initially we should be able to use error handling as the only escape/skip mechanism. This is especially as it is possible to manage orchestration in the client so long as we use a hooks API interface.

## Syntax candidates

### Example view

```tsx
// app/counter/pages/the-counter.tsx
import GetCounter from 'app/counter/interactions/GetCounter'
import Increment from 'app/counter/interactions/Increment'
import Decrement from 'app/counter/interactions/Decrement'
import {useBlitz} from 'blitzjs/view'

export default function Counter(props: { id: number }) {
  const counter = useBlitz(GetCounter, props.id); // This result is computed during SSR
  const increment = useBlitz(Increment); // No argument prop means it is not computed during SSR
  const decrement = useBlitz(Decrement);

  return (
    <div>
      <div>{count}<div>
      <button onClick={() => increment(props.id)}>+</button>
      <button onClick={() => decrement(props.id)}>-</button>
    </div>
  )
}
```

### Example State

```prisma
// prisma/schema.prisma

model Counter {
  id          Int     @id
  count       Int
}
```

### Example Simple Interaction

GetCount

```ts
// app/counter/interactions/GetCounter.ts
import { defineQuery } from "blitzjs/interactions";
import { BlitzDB } from "blitzjs/db";

function GetCounter(db: BlitzDB) {
  return async (id: number) => {
    // no try catch as is managed by the framework errors are bubbled up to the client.
    return await db.counter.findOne({ where: { id } });
  };
}

export default defineQuery(GetCounter);
```

Increment

```ts
// app/counter/interactions/Increment.ts
import { defineCommand } from "blitzjs/interactions";
import { BlitzDB } from "blitzjs/db";
import GetCounter from "./GetCounter";

function Increment(db: BlitzDB) {
  return async (id: number) => {
    const counter = await GetCounter(id); // Not optimal because it is run every time this command is run

    await db.counter.update(
      { data: { count: counter.count + 1 } },
      { where: { id: counter.id } }
    );
  };
}

export default defineCommand(Increment);
```

Decrement

```ts
// app/counter/interactions/Decrement.ts
import { defineCommand } from "blitzjs/interactions";
import { BlitzDB } from "blitzjs/db";
import GetCounter from "./GetCounter";

function Decrement(db: BlitzDB) {
  return async (id: number) => {
    const counter = await GetCounter(id);

    await db.counter.update(
      { data: { count: counter.count - 1 } },
      { where: { id: counter.id } }
    );
  };
}

export default defineCommand(Decrement);
```

### Example of splitting Commands and Queries with Pipe (later)

Pipe allows us to split commands and queries which means we can manage performance better by running commands in an asynchronous queue, caching queries and doing optimistic lock analysis. This could be introduced at a later point when solving race conditions.

#### Decrement Interaction

Pipe orchestration: First get the counter then apply the decrement.

```ts
// app/counter/interactions/Decrement.ts
import { pipe, defineCommand } from "blitzjs/interactions";
import GetCounter from "./GetCounter";
import ApplyDecrement from "./ApplyDecrement";

export default pipe(GetCounter, ApplyDecrement);
```

Command

```ts
// app/counter/interactions/ApplyDecrement.ts
import { pipe, defineCommand } from "blitzjs/interactions";
import { BlitzDB } from "blitzjs/db";

function ApplyDecrement(db: BlitzDB) {
  return async (counter: Counter) => {
    await db.counter.update(
      { data: { count: counter.count - 1 } },
      { where: { id: counter.id } }
    );
  };
}

export default defineCommand(ApplyDecrement);
```

#### Increment Interaction

```ts
// app/counter/interactions/Increment.ts
import { pipe, defineCommand } from "blitzjs/interactions";
import GetCounter from "./GetCounter";
import ApplyDecrement from "./ApplyDecrement";

export default pipe(GetCounter, ApplyIncrement);
```

```ts
// app/counter/interactions/ApplyIncrement.ts
import { pipe, defineCommand } from "blitzjs/interactions";
import { BlitzDB } from "blitzjs/db";

function ApplyIncrement(db: BlitzDB) {
  return async (counter: Counter) => {
    await db.counter.update(
      { data: { count: counter.count + 1 } },
      { where: { id: counter.id } }
    );
  };
}

export default defineCommand(ApplyIncrement);
```
