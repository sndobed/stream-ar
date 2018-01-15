# Introduction
This guide will walk you through the high level architecture of our application, as well as how the code is organized.  This guide assumes at least a minimal understanding of electron.  If you are unfamiliar with electron, you should read through the quick start guide first, which can be found [here](https://electron.atom.io/docs/tutorial/quick-start/).

# High Level Architecture
For most of its lifetime, our application has 3 core processes:
- The main background process (electron main process)
- The main window (electron renderer process)
- The child window (electron renderer process)

## Main Background Process
This process is extremely simple, and exists mainly to orchestrate the creation of browser windows, and also to facilitate communication between windows.  It also hosts [obs-studio-node](https://github.com/stream-labs/obs-studio-node).  obs-studio-node (OSN) is a library primarily written in C++ that exposes OBS's underlying API as a simple Javascript API.  It is essentially an entire copy OBS running without the UI.  It lives in the main process because we want only a single copy of it for the entire app.  The main process does very little to interact with it directly.  Instead, the renderer processes communicate with node-obs over IPC.  In the (hopefully near) future, OSN will be moving to its own dedicated process using a custom IPC communication method.  This is expected to drastically improve stability and performance of the application.

## The Main and Child Windows
These can be described together because they are literally the exact same code.  These are the windows that you actually see on screen while using slobs.  The main window contains the editor/dashboard/live/library views.  The child window is the "popup" window that you see when editing settings or source properties.  In order to simplify things for the developer, these 2 windows execute identical copies of the code.  The main process communicates with each renderer process to tell it what top-level component should be mounted.  This is how each window can display completely different content, even when they are running identical code.

As mentioned before, these two windows are in completely separate processes.  This means they are completely separate execution contexts.  Although you shouldn't have to worry too much about this separation, there are a number of caveats.  For starters, any global variables set in one process won't be accessible from the other process.  Global variables are bad practice for the most part, and our app uses [Vuex](https://github.com/vuejs/vuex) to manage the state of the application.  Vuex is Vue's official state management solution inspired by the Flux architecture.  We have written some code that will keep the Vuex store in sync between the two processes.  This process is invisible to the developer, but it does come with the caveat that anything you put in the Vuex store needs to be serializable.  This is because each time you commit a mutation in the store, it is serialized to JSON and sent over IPC to the other process.  Another caveat is that your mutations cannot have any side effects.  This means that your mutations should not call into any other APIs, or do things like `new Date()`.  This is considered bad practice in mutations anyway, but it has slightly more disastrous effects in our code base than usual.

Our 2-window system also guarantees that methods in services always execute in the main window.  This further simplifies things, and allows services to manage their own state outside of the vuex store.  When a service is injected into a child window component, it instead receives a proxy object that sends requests over IPC to the main window.  This comes with the caveat that arguments passed to service methods must be serializable.

# Code Organization
Here is a quick description of how our files and folders are laid out.
```
slobs-client
|-- app - The source code for the renderer processes live here.  This is the bulk of the codebase
|   |-- components - All Vue components live in here
|   |   |-- mixins - Mixins for Vue components
|   |   |-- pages - Top level pages in the main window
|   |   |-- shared - Any components that are meant to be very generic, like form inputs
|   |   |-- source_properties - Deprecated, do not use
|   |   |-- windows - Top level components that are meant to mount at the top of a browser window
|   |
|   |-- services - All services live in here (see services section)
|   |-- store - Bootstrapping code for our vuex store.  New code shouldn't be added here.
|   |-- styles - Global CSS styles and helpers go in here
|   |-- util - For code that doesn't belong anywhere else.  Use sparingly.
|
|-- bin - Some JS scripts meant to be run from the command line
|-- media - Images and videos go in here
|-- node-libuiohook - This is a library we use for hotkey handling, added as a git submodule
|-- plugins - OBS plugins go here, and are extracted into node-obs.  We are trying to move away from this system, and install plugins directly from their respective repositories
|-- test - Our integration tests live here
|   |-- helpers - Re-usable helper functions for tests
|   |-- stress - Our stress testing system
|
|-- updater - This is the source for the auto-updater. It is kept separately and rarely changed to ensure its stability.
|-- vendor - Assets like fontawesome go in here.  We should not be adding things here regularly.
|-- index.html - The html that is loaded in every renderer process
|-- main.js - The source code for the main process, just 1 file
```

# Services
Services are a central part of our application logic.  If you have some application logic that doesn't belong in a Vue component, chances are that it belongs in a service.  Services are defined as classes that inherit from the `Service` class, either directly or through another class.  They use the singleton pattern, and expose useful methods to the rest of the application.  Services can be called into from anywhere in the application, and essentially expose an API that the UI layer can use to perform actions.  These actions can be things like performing operations in OBS, or loading data from a server, or even just performing a difficult calculation.  There are a number of abstract classes that inherit from `Service` that can be used to inject more advanced behavior into your service.  When you define a new service, it must be registered with the services manager (`app/services-manager.ts`) for it to be available to the rest of the app.

# Vuex
The high level description mentions that we use [Vuex](https://github.com/vuejs/vuex) for state management.  If you are unfamiliar with Vuex, I would recommend familiarizing yourself with the core concepts first.  Our use of Vuex is non-standard enough that it warrants an entire section describing it.

## Problems With Vanilla Vuex
Vuex uses a module system to allow scaling into a very large store.  The idea is that you write separate modules that define mutations, actions, and getters, which get mixed into the top level store.  There are a number of issues with this design:
- Mutation names need to be unique across all modules
- Modules are not classes, and as such it is difficult to factor the code in a way that encourages re-use
- Vuex's event-drive syntax is verbose and unclear

## Our Solution
In order to use Vuex in a more sustainable way, we have come up with a solution that allows you to integrate Vuex functionality directly in your service.  There is an abstract class called `StatefulService`, that provides this functionality.  Mutations can simply be defined as methods on your class using the `@mutation()` decorator.  They can access `this.state` directly, and take normal type-checked function arguments.  Under the hood, an actual Vuex mutation will be committed asynchronously whenever the method is called.  But since mutations can't have return values anyway, there is no need to expose this asynchronous behavior to the developer.  We consider mutations to be private to the service that defines them.

As for Actions and Getters, we don't use them.  Vuex actions are really no different from normal methods.  They can be synchronous or asynchronous, have side effects, return values, and can commit mutations.  As such, we forego them entirely, and just define functions on our service instead.  As mentioned previously, mutations are considered private to their service.  To expose that behavior to the rest of the app, you should simply define a public function on your service that commits the mutation.  Getters are equally useless now that we have class syntax.  You can simply define getters on your service that access `this.state`.  Because your getters are accessing `this.state`, which in turn is just a view into the Vuex store, all the built-in dependency management and reactivity that Vue/Vuex do under the hood will still work just fine.


