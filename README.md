# ElmTS Framework

ElmTS is a TypeScript framework inspired by Elm, designed for building scalable, maintainable web applications with predictable state management and excellent performance through code splitting.

## Core Concepts

1. **Single State Tree**: The entire application state is stored in a single, immutable object.
2. **Messages**: All state changes are triggered by messages.
3. **Update Function**: A pure function that takes the current state and a message, and returns a new state.
4. **Commands**: Side effects are handled through commands, keeping the update function pure.
5. **Subscriptions**: External inputs are managed through subscriptions.
6. **View**: The UI is a pure function of the application state.

## Application Structure

```
src/
├── main.ts
├── types.ts
├── modules/
│   ├── auth/
│   │   ├── types.ts
│   │   ├── model.ts
│   │   ├── update.ts
│   │   ├── view.tsx
│   │   └── api.ts
│   ├── chat/
│   │   ├── types.ts
│   │   ├── model.ts
│   │   ├── update.ts
│   │   ├── view.tsx
│   │   └── api.ts
│   └── landing/
│       ├── types.ts
│       ├── model.ts
│       ├── update.ts
│       └── view.tsx
└── lib/
    ├── elmts.ts
    └── socket.ts
```

## Core Types

```typescript
// src/types.ts

import { AuthModel, AuthMsg } from "./modules/auth/types";
import { ChatModel, ChatMsg } from "./modules/chat/types";
import { LandingModel, LandingMsg } from "./modules/landing/types";

export type Model = {
  page: Page;
  auth: AuthModel;
  chat?: ChatModel;
  landing?: LandingModel;
  isAuthenticated: boolean;
};

export type Page =
  | { type: "Loading"; route: string }
  | { type: "Auth" }
  | { type: "Chat" }
  | { type: "Landing" }
  | { type: "NotFound" };

export type Msg =
  | { type: "NavigateTo"; route: string }
  | { type: "ModuleLoaded"; module: any }
  | { type: "LoadError"; error: Error }
  | { type: "AuthMsg"; payload: AuthMsg }
  | { type: "ChatMsg"; payload: ChatMsg }
  | { type: "LandingMsg"; payload: LandingMsg }
  | { type: "SetAuthenticated"; value: boolean };

export type Cmd<M> = {
  execute: () => Promise<M>;
};

export type Sub<M> = {
  subscribe: (dispatch: (msg: M) => void) => void;
  unsubscribe: () => void;
};
```

## Main Application

```typescript
// src/main.ts

import { createElmApp, Cmd } from "./lib/elmts";
import { Model, Msg, Page } from "./types";
import { AuthModel, authInit } from "./modules/auth/model";
import { chatSubscriptions } from "./modules/chat/model";

const init = (): [Model, Cmd<Msg>] => {
  const [authModel, authCmd] = authInit();
  return [
    {
      page: { type: "Auth" },
      auth: authModel,
      isAuthenticated: false,
    },
    Cmd.map(authCmd, (msg): Msg => ({ type: "AuthMsg", payload: msg })),
  ];
};

const guardRoute = (route: string, model: Model): [Model, Cmd<Msg>] => {
  if (route === "/chat" && !model.isAuthenticated) {
    return [
      { ...model, page: { type: "Auth" } },
      Cmd.ofMsg({ type: "NavigateTo", route: "/auth" }),
    ];
  }
  return loadModule(route, model);
};

const update = (msg: Msg, model: Model): [Model, Cmd<Msg>] => {
  switch (msg.type) {
    case "NavigateTo":
      return guardRoute(msg.route, model);
    case "ModuleLoaded":
      return initializeLoadedModule(msg.module, model);
    case "LoadError":
      return [{ ...model, page: { type: "NotFound" } }, Cmd.none];
    case "SetAuthenticated":
      return [{ ...model, isAuthenticated: msg.value }, Cmd.none];
    case "AuthMsg":
      const [authModel, authCmd] = updateAuth(msg.payload, model.auth);
      let newModel = { ...model, auth: authModel };
      if (msg.payload.type === "SignInSuccess") {
        newModel.isAuthenticated = true;
      } else if (msg.payload.type === "SignOut") {
        newModel.isAuthenticated = false;
      }
      return [
        newModel,
        Cmd.map(authCmd, (msg) => ({ type: "AuthMsg", payload: msg })),
      ];

    case "ChatMsg":
      return updateChat(msg.payload, model);
    case "LandingMsg":
      return updateLanding(msg.payload, model);
  }
};

const loadModule = (route: string, model: Model): [Model, Cmd<Msg>] => {
  let modulePromise: Promise<any>;
  let pageType: Page["type"];

  switch (route) {
    case "/auth":
      return [{ ...model, page: { type: "Auth" } }, Cmd.none];
    case "/chat":
      modulePromise = import("./modules/chat/model");
      pageType = "Chat";
      break;
    case "/landing":
      modulePromise = import("./modules/landing/model");
      pageType = "Landing";
      break;
    default:
      return [{ ...model, page: { type: "NotFound" } }, Cmd.none];
  }

  return [
    { ...model, page: { type: "Loading", route } },
    Cmd.fromPromise(
      () => modulePromise,
      (module) => ({ type: "ModuleLoaded", module }),
      (error) => ({ type: "LoadError", error })
    ),
  ];
};

const initializeLoadedModule = (
  module: any,
  model: Model
): [Model, Cmd<Msg>] => {
  if (model.page.type !== "Loading") return [model, Cmd.none];

  switch (model.page.route) {
    case "/chat":
      const [chatModel, chatCmd] = module.chatInit();
      return [
        { ...model, page: { type: "Chat" }, chat: chatModel },
        Cmd.map(chatCmd, (msg) => ({ type: "ChatMsg", payload: msg })),
      ];
    case "/landing":
      const [landingModel, landingCmd] = module.landingInit();
      return [
        { ...model, page: { type: "Landing" }, landing: landingModel },
        Cmd.map(landingCmd, (msg) => ({ type: "LandingMsg", payload: msg })),
      ];
    default:
      return [{ ...model, page: { type: "NotFound" } }, Cmd.none];
  }
};

const updateAuth = (msg: AuthMsg, model: Model): [Model, Cmd<Msg>] => {
  const [authModel, authCmd] = authUpdate(msg, model.auth);
  return [
    { ...model, auth: authModel },
    Cmd.map(authCmd, (msg) => ({ type: "AuthMsg", payload: msg })),
  ];
};

// Similar updateChat and updateLanding functions...
const view = (model: Model, dispatch: (msg: Msg) => void) => {
  switch (model.page.type) {
    case "Loading":
      return <div>Loading {model.page.route}...</div>;
    case "Auth":
      return (
        <AuthView
          model={model.auth}
          dispatch={(msg) => dispatch({ type: "AuthMsg", payload: msg })}
        />
      );
    case "Chat":
      return model.isAuthenticated && model.chat ? (
        <ChatView
          model={model.chat}
          dispatch={(msg) => dispatch({ type: "ChatMsg", payload: msg })}
        />
      ) : (
        <div>Please sign in to access the chat.</div>
      );
    case "Landing":
      return model.landing ? (
        <LandingView
          model={model.landing}
          dispatch={(msg) => dispatch({ type: "LandingMsg", payload: msg })}
        />
      ) : (
        <div>Error: Landing not initialized</div>
      );
    case "NotFound":
      return <div>404 Not Found</div>;
  }
};

const subscriptions = (model: Model): Sub<Msg> => {
  return Sub.batch([
    // Global subscriptions...
    model.chat
      ? Sub.map(
          chatSubscriptions(),
          (msg) => ({ type: "ChatMsg", payload: msg } as Msg)
        )
      : Sub.none,
  ]);
};

const app = createElmApp({
  init,
  update,
  view,
  subscriptions,
});

app.start();
```

## Auth Module

```typescript
// src/modules/auth/types.ts

export type AuthModel = {
  user: User | null;
  form: {
    email: string;
    password: string;
  };
  error: string | null;
};

export type User = {
  id: string;
  email: string;
};

export type AuthMsg =
  | { type: "UpdateForm"; field: "email" | "password"; value: string }
  | { type: "AttemptSignIn" }
  | { type: "SignInSuccess"; user: User }
  | { type: "SignInFailure"; error: string }
  | { type: "SignOut" };
```

```typescript
// src/modules/auth/model.ts

import { AuthModel, AuthMsg } from "./types";
import { Cmd } from "../../lib/elmts";
import { signIn } from "./api";

export const authInit = (): [AuthModel, Cmd<AuthMsg>] => [
  {
    user: null,
    form: { email: "", password: "" },
    error: null,
  },
  Cmd.none,
];

export const authUpdate = (
  msg: AuthMsg,
  model: AuthModel
): [AuthModel, Cmd<AuthMsg>] => {
  switch (msg.type) {
    case "UpdateForm":
      return [
        { ...model, form: { ...model.form, [msg.field]: msg.value } },
        Cmd.none,
      ];
    case "AttemptSignIn":
      return [
        { ...model, error: null },
        Cmd.fromPromise(
          () => signIn(model.form),
          (user) => ({ type: "SignInSuccess", user }),
          (error) => ({ type: "SignInFailure", error: error.message })
        ),
      ];
    case "SignInSuccess":
      return [{ ...model, user: msg.user, error: null }, Cmd.none];
    case "SignInFailure":
      return [{ ...model, error: msg.error }, Cmd.none];
    case "SignOut":
      return [{ ...model, user: null }, Cmd.none];
  }
};
```

```typescript
// src/modules/auth/view.tsx

import { AuthModel, AuthMsg } from "./types";

export const AuthView = ({
  model,
  dispatch,
}: {
  model: AuthModel;
  dispatch: (msg: AuthMsg) => void;
}) => (
  <div>
    {model.user ? (
      <div>
        <p>Welcome, {model.user.email}!</p>
        <button onClick={() => dispatch({ type: "SignOut" })}>Sign Out</button>
        <button
          onClick={() => dispatch({ type: "NavigateTo", route: "/chat" })}
        >
          Go to Chat
        </button>
      </div>
    ) : (
      <form
        onSubmit={(e) => {
          e.preventDefault();
          dispatch({ type: "AttemptSignIn" });
        }}
      >
        <input
          type="email"
          value={model.form.email}
          onChange={(e) =>
            dispatch({
              type: "UpdateForm",
              field: "email",
              value: e.target.value,
            })
          }
        />
        <input
          type="password"
          value={model.form.password}
          onChange={(e) =>
            dispatch({
              type: "UpdateForm",
              field: "password",
              value: e.target.value,
            })
          }
        />
        <button type="submit">Sign In</button>
      </form>
    )}
    {model.error && <p>{model.error}</p>}
  </div>
);
```

```typescript
// src/modules/auth/api.ts

import { User } from "./types";

export const signIn = async (credentials: {
  email: string;
  password: string;
}): Promise<User> => {
  // Simulate API call
  await new Promise((resolve) => setTimeout(resolve, 1000));
  if (
    credentials.email === "user@example.com" &&
    credentials.password === "password"
  ) {
    return { id: "1", email: credentials.email };
  }
  throw new Error("Invalid credentials");
};
```

## Chat Module

```typescript
// src/modules/chat/types.ts

export type ChatModel = {
  messages: ChatMessage[];
  currentMessage: string;
};

export type ChatMessage = {
  id: string;
  user: string;
  text: string;
  timestamp: number;
};

export type ChatMsg =
  | { type: "ReceiveMessage"; message: ChatMessage }
  | { type: "UpdateCurrentMessage"; text: string }
  | { type: "SendMessage" }
  | { type: "SocketConnected" }
  | { type: "SocketDisconnected" };
```

```typescript
// src/modules/chat/model.ts

import { ChatModel, ChatMsg, ChatMessage } from "./types";
import { Cmd, Sub } from "../../lib/elmts";
import { sendMessage, io } from "./api";

export const chatInit = (): [ChatModel, Cmd<ChatMsg>] => [
  {
    messages: [],
    currentMessage: "",
  },
  Cmd.none,
];

export const chatUpdate = (
  msg: ChatMsg,
  model: ChatModel
): [ChatModel, Cmd<ChatMsg>] => {
  switch (msg.type) {
    case "ReceiveMessage":
      return [
        { ...model, messages: [...model.messages, msg.message] },
        Cmd.none,
      ];
    case "UpdateCurrentMessage":
      return [{ ...model, currentMessage: msg.text }, Cmd.none];
    case "SendMessage":
      return [
        { ...model, currentMessage: "" },
        Cmd.fromPromise(
          () => sendMessage(model.currentMessage),
          () => ({
            type: "ReceiveMessage",
            message: {
              id: Date.now().toString(),
              user: "Me",
              text: model.currentMessage,
              timestamp: Date.now(),
            },
          })
        ),
      ];
    case "SocketConnected":
    case "SocketDisconnected":
      // You might want to update the model to reflect the socket connection status
      return [model, Cmd.none];
  }
};

export const chatSubscriptions = (): Sub<ChatMsg> => {
  return Sub.batch([
    Sub.fromSocketEvent(io, "connect", () => ({ type: "SocketConnected" })),
    Sub.fromSocketEvent(io, "disconnect", () => ({
      type: "SocketDisconnected",
    })),
    Sub.fromSocketEvent<ChatMessage>(io, "chat message", (message) => ({
      type: "ReceiveMessage",
      message,
    })),
  ]);
};
```

```typescript
// src/modules/chat/view.tsx

import { ChatModel, ChatMsg } from "./types";

export const ChatView = ({
  model,
  dispatch,
}: {
  model: ChatModel;
  dispatch: (msg: ChatMsg) => void;
}) => (
  <div>
    <div className="chat-messages">
      {model.messages.map((msg) => (
        <div key={msg.id}>
          <strong>{msg.user}:</strong> {msg.text}
        </div>
      ))}
    </div>
    <form
      onSubmit={(e) => {
        e.preventDefault();
        dispatch({ type: "SendMessage" });
      }}
    >
      <input
        type="text"
        value={model.currentMessage}
        onChange={(e) =>
          dispatch({ type: "UpdateCurrentMessage", text: e.target.value })
        }
      />
      <button type="submit">Send</button>
    </form>
  </div>
);
```

```typescript
// src/modules/chat/api.ts

import { io } from "socket.io-client";

export const socket = io("http://localhost:3000");

export const sendMessage = async (text: string): Promise<void> => {
  socket.emit("chat message", text);
};
```

```typescript
// src/lib/socket.ts

import { io as ioClient } from "socket.io-client";

export const io = ioClient("http://localhost:3000");
```

```typescript
// src/lib/elmts.ts

import { Socket } from "socket.io-client";

export type Cmd<M> = {
  execute: () => Promise<M>;
};

export type Sub<M> = {
  subscribe: (dispatch: (msg: M) => void) => void;
  unsubscribe: () => void;
};

export const Cmd = {
  none: { execute: () => Promise.resolve() } as Cmd<never>,

  ofMsg: <M>(msg: M): Cmd<M> => ({
    execute: () => Promise.resolve(msg),
  }),

  fromPromise: <T, M>(
    promiseFn: () => Promise<T>,
    toMsg: (value: T) => M,
    onError: (error: Error) => M
  ): Cmd<M> => ({
    execute: () => promiseFn().then(toMsg).catch(onError),
  }),

  batch: <M>(cmds: Cmd<M>[]): Cmd<M> => ({
    execute: () =>
      Promise.all(cmds.map((cmd) => cmd.execute())).then(
        () => undefined as any
      ),
  }),

  map: <A, B>(cmd: Cmd<A>, fn: (a: A) => B): Cmd<B> => ({
    execute: () => cmd.execute().then(fn),
  }),
};

export const Sub = {
  none: {
    subscribe: () => {},
    unsubscribe: () => {},
  } as Sub<never>,

  batch: <M>(subs: Sub<M>[]): Sub<M> => ({
    subscribe: (dispatch) => subs.forEach((sub) => sub.subscribe(dispatch)),
    unsubscribe: () => subs.forEach((sub) => sub.unsubscribe()),
  }),

  fromSocketEvent: <T>(
    socket: Socket,
    eventName: string,
    toMsg: (data: T) => any
  ): Sub<any> => ({
    subscribe: (dispatch) => {
      socket.on(eventName, (data: T) => dispatch(toMsg(data)));
    },
    unsubscribe: () => {
      socket.off(eventName);
    },
  }),

  map: <A, B>(sub: Sub<A>, fn: (a: A) => B): Sub<B> => ({
    subscribe: (dispatch) => sub.subscribe((a) => dispatch(fn(a))),
    unsubscribe: () => sub.unsubscribe(),
  }),
};

type AppConfig<Model, Msg> = {
  init: () => [Model, Cmd<Msg>];
  update: (msg: Msg, model: Model) => [Model, Cmd<Msg>];
  view: (model: Model, dispatch: (msg: Msg) => void) => any;
  subscriptions?: (model: Model) => Sub<Msg>;
};

export function createElmApp<Model, Msg>(config: AppConfig<Model, Msg>) {
  let model: Model;
  let activeSub: Sub<Msg> | null = null;

  function dispatch(msg: Msg) {
    const [newModel, cmd] = config.update(msg, model);
    model = newModel;
    cmd.execute().then(dispatch);
    updateSubscriptions();
    render();
  }

  function updateSubscriptions() {
    if (activeSub) {
      activeSub.unsubscribe();
    }
    if (config.subscriptions) {
      activeSub = config.subscriptions(model);
      activeSub.subscribe(dispatch);
    }
  }

  function render() {
    const viewResult = config.view(model, dispatch);
    // Here you would update the DOM with the view result
    console.log("Render:", viewResult);
  }

  return {
    start: () => {
      const [initialModel, cmd] = config.init();
      model = initialModel;
      cmd.execute().then(dispatch);
      updateSubscriptions();
      render();
    },
  };
}
```

## Conclusion

This ElmTS framework provides a robust, scalable architecture for building web applications. It maintains the core principles of Elm while leveraging TypeScript and modern web development practices:

- **Predictable State Management**: All state changes are handled through a single update function, making the flow of data easy to understand and debug.
- **Code Splitting**: Modules are loaded asynchronously, improving initial load times and overall performance.
- **Type Safety**: TypeScript provides strong typing throughout the application, catching errors at compile-time.
- **Pure Functions**: The update and view functions are pure, making them easy to test and reason about.
- **Explicit Side Effects**: All side effects are handled through commands, keeping the core logic pure.
- **Scalability**: The modular structure allows the application to grow while maintaining clarity and organization.

By following this architecture, you can build complex, interactive web applications that are easy to maintain and extend over time.

```

This README provides a comprehensive overview of the ElmTS framework, including its core concepts, application structure, and detailed implementations of the authentication and chat modules. It demonstrates how to achieve code splitting, handle asynchronous operations, and manage real-time features like chat, all while maintaining the simplicity and predictability of Elm-style state management.
```
