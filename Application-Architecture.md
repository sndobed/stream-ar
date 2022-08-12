# Introduction
This guide will walk you through the high level architecture of our application, as well as how the code is organized.  This guide assumes at least a minimal understanding of electron.  If you are unfamiliar with electron, you should read through the quick start guide first, which can be found [here](https://electron.atom.io/docs/tutorial/quick-start/).

# High Level Architecture
Electron uses a multi-process architecture. There is a main electron process, and then each application window is also a separate process. There are additional processes that are spawned in certain contexts for additional windows or embedded web content.

For most of its lifetime, our application has 3 core processes:
- The main background process (electron main process)
- The worker window (electron renderer process)
- The main window (electron renderer process)
- The child window (electron renderer process)

## Main Background Process
This process is extremely simple, and exists mainly to orchestrate the creation of browser windows, and also to facilitate communication between windows.  The code that runs in this process can be found [here](https://github.com/stream-labs/desktop/blob/master/main.js). This represents a very small subset of the code in our application.

## The Worker, Main, and Child Windows
These processes are all electron "renderer" processes, which means they represent an application window.  In our application all of these processes execute the exact same code.  They run our main javascript bundle, which can be found in the `app/` folder in our desktop repository.  While these windows run the same code, they behave differently due some initial bootstrapping code that changes the behavior based on which window it is executing in. We will go through each of these windows in detail.

## The Worker Window
This window is actually hidden.  It will never mount any React or Vue components, and its window will never be shown to the user.  Its purpose is to execute performance intensive tasks on behalf of the rest of the frontend.

The OBS backend client lives in this process.  OBS can *only* be accessed directly from this process.  All of our Frontend services also live in this process.  The worker window contains the canonical version of the Vuex state.  All mutations to this state also happen within the worker window.  Other windows interact with this window via an IPC connection.

## The Main Window
This represents the "main" window that user sees when using the app.  This window is always visible to the user as long as the application is running.  The main window has a *copy* of the Vuex store that is synced from the canonical version kept in the worker window.  The Vuex store in the main window should be considered *read only* and is used to preserve reactivity of Vue and React components in the main window.

The main window interacts with the worker window primarily by interacting with services.  When a service is injected in the main window, it actually receives a thin proxy copy of the service it requested.  Any calls into that service will actually be proxied to the worker window for execution, and if applicable, the result of the operation will be sent back to the main window.  The semantics of inter-process proxying depend on the situation and how it is called, and is beyond the scope of this document.  However, the proxy system allows for both synchronous and asynchronous proxying, as well as handling of return values.  This does have the limitation of requiring arguments and returns values to be serializable via the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm).

## The Child Window
The child window has all the same semantics as the main window, *except* that it is usually hidden and running in the background until needed.  When you click on something in Streamlabs Desktop that opens up a sub-window, it is usually using the child window.  This window sits in the background until it is needed.  Electron windows can take a second or more to initialize and load a large javascript bundle, so for a smoother UX we keep this window always running but hidden, so it's ready to pop up at a moments notice.

Other than its purpose, the child window behaves exactly like the main window.  It has it's own copy of the Vuex store, and renders its UI with React and Vue components.  And it relies on proxying into the worker window to do the heavy lifting.

# Services
Services are a central part of our application logic.  If you have some application logic that doesn't belong in a Vue or React component, chances are that it belongs in a service.  Services are defined as classes that inherit from the `Service` class, either directly or through another class.  They use the singleton pattern, and expose useful methods to the rest of the application.  Services can be called into from anywhere in the application, and essentially expose an API that the UI layer can use to perform actions.  These actions can be things like performing operations in OBS, or loading data from a server, or even just performing a difficult calculation.  There are a number of abstract classes that inherit from `Service` that can be used to inject more advanced behavior into your service.  When you define a new service, it must be registered with the services manager (`app/services-manager.ts`) for it to be available to the rest of the app.

# Vuex
The high level description mentions that we use [Vuex](https://github.com/vuejs/vuex) for state management.  If you are unfamiliar with Vuex, I would recommend familiarizing yourself with the core concepts first.  Our use of Vuex is non-standard enough that it warrants an entire section describing it.

We have continued to use Vuex even as we have migrated to React.  We will eventually migrate to something different, but for now we have written an adapter to leverage Vuex reactivity inside React components.

## Our Solution
In order to use Vuex in a more sustainable way, we have come up with a solution that allows you to integrate Vuex functionality directly in your service.  There is an abstract class called `StatefulService`, that provides this functionality.  Mutations can simply be defined as methods on your class using the `@mutation()` decorator.  They can access `this.state` directly, and take normal type-checked function arguments.  Under the hood, an actual Vuex mutation will be committed asynchronously whenever the method is called.  But since mutations can't have return values anyway, there is no need to expose this asynchronous behavior to the developer.  We consider mutations to be private to the service that defines them.

As for Actions and Getters, we don't use them.  Vuex actions are really no different from normal methods.  They can be synchronous or asynchronous, have side effects, return values, and can commit mutations.  As such, we forego them entirely, and just define functions on our service instead.  As mentioned previously, mutations are considered private to their service.  To expose that behavior to the rest of the app, you should simply define a public function on your service that commits the mutation.  Getters are equally useless now that we have class syntax.  You can simply define getters on your service that access `this.state`.  Because your getters are accessing `this.state`, which in turn is just a view into the Vuex store, all the built-in dependency management and reactivity that Vue/Vuex do under the hood will still work just fine.


