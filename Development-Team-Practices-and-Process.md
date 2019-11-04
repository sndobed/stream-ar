# Coding Style
We use [TSLint](https://palantir.github.io/tslint/) to handle formatting of Typescript.  We use the "Prettier" base configuration for TSLint, which enforces our coding style and provides a automatic fixer for most violations.  This makes it extremely easy to adhere to the style guide and let the developer focus on the important stuff.  It is highly recommended that you install TSLint into your editor.  The CI build will fail if you submit a PR with TSLint violations.

# Code Policies
Here are some policies for adding new code to this repo:
- All new code *must* be reviewed before going into the `staging` branch
- All JS code should be written as TypeScript.  Vanilla JavaScript is never allowed.
- All new Vue code should be written as single file TSX components.
- Styling for Vue components should be handled with CSS modules `m.less`.
- Vue components should be as minimal as possible. Anything that isn't specifically involved with rendering HTML or handling user input should be in a service class instead.  Keep UI components lightweight and re-usable.
- ICI *must* be passing before merging into `staging`
- All new *major* features should be accompanied by an integration test (or a few)
- jQuery is banned from this repo.  There are a lot of existing libraries written as jQuery plugins that are tempting to use, but introducing jQuery into your Vue components and mutating the DOM outside of template rendering leads to difficult to diagnose bugs.
- This project supports multiple languages. All strings that are going to be shown to users must be added to i18n dictionaries. See the "Internationalization" section.

# Code Guidelines
These are less rigid than policies, but should be adhered to unless there is a good reason not to:
- Any single file in the repo should not be larger than 500 lines unless there is a good reason.  Try splitting up your functionality into more logical chunks.
- When adding a new library, think about whether we have an existing library that does something similar.  Also think about whether you can easily accomplish the same thing without a library.

# Workflow
These are some guidelines on our workflow, mostly surrounding git and github:
- If you are a member of the Streamlabs team, do your work on a branch on the main repo.  Do not work on a fork.  Forks can make it more complicated to collaborate on changes.  If you are not a member of the Streamlabs team, you can ignore this.
- Do each logical piece of work on a new branch, and open a separate PR for each new change coming into `staging`
- Use the "Squash & Merge" option on github.  This keeps the commit history clean.

# Tools
First off, everything in this section is completely optional.  We firmly believe you should use the tools you are most comfortable with.  That said, these tools offer a good experience with the technologies we use in this project.  This should be taken as a recommendation but definitely not a requirement.

## Editor
Visual Studio Code is a great editor for this project.  It's a modern cross-platform editor.  It is built on Electron and TypeScript (just like slobs), so it has great out-of-the-box support for these technologies.

If you choose to use VSCode, here are some extensions we highly recommend:
- Vetur - This is an essential plugin that provides a much better experience when editing .vue files
- TSLint - Linting for TypeScript
- Trailing Spaces - Highlights trailing whitespace

JetBrains IDEs like webstorm or phpstorm are also a good choice if you are used to it. All code linters are already included.
Only the Vue.js plugin is required.

It is also possible to use Sublime and Atom (or any other editor) for our project, but it those instructions are beyond the scope of this guide.

# Internationalization
If you commit has the strings that supposed to be translated, you have to add these strings it into one of the JSON dictionaries in `i18n/en-US` folder. We use [vue-i18n](https://github.com/kazupon/vue-i18n) plugin for templating. You can run the util `node build-utils/search-untranslated.js` that find the all wrapped into the template function `$t()` stings that don't present in the dictionaries.

# Final Comments
If you don't understand one of these rules, or why it has been implemented, please ask.  If you think something is dumb, please say so.  Anything on this page can be changed if the team agrees on it.