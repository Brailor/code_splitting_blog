# Code Splitting

First of all what is code splitting and why should we care?

Modern Web Applications (especially SPAs) are everywhere, that means JavaScript is everywhere. As we add more features to our app the size of JS (and CSS) is growing too and the users of our app are paying the price of downloading, parsing, and executing this code.

![Image alt text](./js_bytes_3years.png '3 Years of JS')

> In the last 3 years the total size of JavaScript on an average page grew by 37%. On mobile it is even worse.

We as developers have the luxury of working with high end devices and fast internet connection, however our users may not have the same so we have to make an effort to deliver quality experience even when the network connection is not so great or the user is using an older and weaker mobile device.

![Image alt text](./desktop_vs_mobile.png 'Load time of JS')

> Load times of JS on different devices. Source: [Reduce JavaScript Payloads with Code Splitting](https://developers.google.com/web/fundamentals/performance/optimizing-javascript/code-splitting/)

### Code splitting aims to help with this by reducing the initial size of the JavaScript (and CSS) files our users have to download, parse and execute.

## Do we need to split our code?

Since we are not developing public facing applications I would say it is not necessary, but still the users of our app would only benefit from doing it and I think it is not a bad idea to use best practices where it would make sense.

## When should we split our code?

Before jumping in and refactoring the whole codebase to use code splitting, it makes sense to actually measure the performance of our app.

There are a couple of measurment metrics out there, the ones that we are interested are:

- **First Contentful Paint (FCP)** - measures the time from navigation to the time when the browser renders the first bit of content from the DOM

- **Time to Interactive (TTI)** - measures how long it takes the page to become interactive

We can measure the **TTI** and **FCP** (and many other things) of our app by using this tool called [Lighthouse](https://developers.google.com/web/tools/lighthouse/) by Google.

![Image alt text](./lightouse_report.png 'Lighthouse report')

> Lighthouse will give us a nice report and also it will even suggests hints to improve the performance of our app

---

## Once we decided that we want to split our code it actually makes sense to set some goals in terms of resource sizes and code coverage.

### JavaScript: <= ~200kB (uncompressed) Initial JS [total] (varies by app)

### CSS: <=100kB (uncompressed) Initial CSS [total]

### 80-90% Code Coverage (only 10-20% code unused)

![Image alt text](./code_coverage.png 'Code coverage of myidtravel')

> 5.3MB of content (after uncompressing) just to show a login screen, code splitting would definetely help here.

# Code splitting techniques

- **Vendor splitting**: by splitting up our third party dependencies from our app logic we can ensure that the user will have our vendor bundle cached on subsequent visits

- **Static splitting**: by using the [dynamic import proposal](https://github.com/tc39/proposal-dynamic-import) ([which is in Stage 3](https://github.com/tc39/proposals)) the code bundler (in our case Webpack) can split up our code during build time

- **Dynamic splitting**: we essentially tell the bundler to bundle the file contents of a specific directory and based on runtime conditions load the appropriate chunks (more on this later)

### **Vendor Splitting:**

By using Create React App the Webpack configuration is already set up for us to enable vendor splitting, but if we would like to write it manually we can do it like this:

```js
//webpack.config.js

module.exports = {
 ...
  optimization: {
    splitChunks: {
      cacheGroups: {
        // Split vendor code
        vendors: {
          test: /[\\/]node_modules[\\/]/i,
          chunks: "all",
          name: "vendors"
        },
        // Split code that is common to all other chunks
        commons: {
          name: "commons",
          chunks: "initial",
          minChunks: 2
        }
      },
      // Split the runtime as well, just like in CRA
      runtimeChunk: {
      name: "runtime"
    }
  ...
};
```

That is it, we don't have to do anything more, Webpack will create the 'vendors', 'commons', and the 'runtime' chunks for us.

There is another topic worth mentioning in this context is called **Tree Shaking**.

Tree shaking is a term commonly used in the JavaScript context for dead-code elimination (actually it is rather live code inclusion).

Lets say we want to import a specific function from a third party library, eg.: map function from Lodash.

```js
//some-file.js

import { map } from 'lodash';
```

The problem with this is webpack for various reasons (basically webpack doesn't know anything about the side effects that could occur in the module we are importing from) is going to include the whole library inside our 'vendors' chunk not just the function we actually import/use like we would expect.

> A "side effect" is defined as code that performs a special behavior when imported, other than exposing one or more exports. An example of this are polyfills, which affect the global scope and usually do not provide an export.

Of course we can change that. In our case we would need to use 'lodash-es' library which is basically Lodash split up to support ES Modules.

```js
//some-file.js

import { map } from 'lodash-es';
```

Now our bundle will only contain the 'map' function.

We can enable Tree Shaking with our own modules as well. If our modules doesn't contain any side effects we can straight up tell webpack about it by adding the 'sideEffects' proprety to our package.json.

```js
  //package.json
  {
    ...
    "sideEffects": false,
    ...
  }

```

If we have some file(s) with side effects we can specify them in an array instead:

```js
  //package.json
  {
    ...
    "sideEffects": [
      "./src/some-side-effectful-file.js"
    ],
    ...
  }
```

> Note: CommonJS imports are not Tree Shakeable.

### **Static splitting:**

The dynamic import function is a proposal currently at Stage 3 which means it will likely be forged into the language in the upcoming years, nevertheless we can use it today with Babel and apps created with CRA (Create React App) are supporting this feature out of the box.

Without CRA we have to add [babel-plugin-syntax-dynamic-import](https://babeljs.io/docs/en/babel-plugin-syntax-dynamic-import) plugin into our transpilation process.

Using static splitting based on routes is one of the easiest - and most common - way to introduce code splitting to our app.

```js
const loadingCmp = () => <div>Loading...</div>;

const Settings = asyncComponent(
  () =>
    import(// This tells webpack to name the chunk as 'Settings.chunk.js'

    /* webpackChunkName: 'Settings'*/

    // just like a normal import we have to specify the path to the file
    '../Settings/Settings'),

  // while we download the chunk show a loading indicator
  loadingCmp
);

const Duties = asyncComponent(() => import(/* webpackChunkName: 'Duties'*/ '../Duties/Duties'), loadingCmp);

function asyncComponent(importComponent, LoadingComponent) {
  class AsyncComponent extends Component {
    constructor(props) {
      super(props);
      this.state = {
        component: null
      };
    }

    async componentDidMount() {
      const { default: component } = await importComponent();

      this.setState({
        component: component
      });
    }

    render() {
      const Cmp = this.state.component;

      return Cmp ? <Cmp {...this.props} /> : <LoadingComponent />;
    }
  }

  return AsyncComponent;
}
```

> The asyncComponent function is just a higher order function which will return the imported component.
>
> Note: there are several libraries out there to achive the same behaviour, just to name one: [React Loadable](https://github.com/jamiebuilds/react-loadable)

### Then we can start using our async components in our router like so:

```js
<Router>
  // this is our async components
  <Route exact path="/" render={() => <Duties />} />
  <Route exact path="/settings" render={() => <Settings />} />
  // sync components
  <Route exact path="/summary" component={Summary} />
  <Route exact path="/calendar" component={Calendar} />
</Router>
```

Now we can build the project and inspect the bundles webpack generate.

First lets check the bundle without using any route based code splitting:

![Image alt text](./no_split.png 'No split bundle')

And after we used some route based splitting:

![Image alt text](./route_based_split.png 'Route based splitting')

> As we can see webpack added 4 more chunks to our bundle ( 2 JS - 2 CSS) named 'Settings' and 'Duties'.

![Image alt text](./route_based_split_visual.png 'Route based splitting visual')

> This is how our app would look like after the route based split.

After the route based split our main bundle will not contain the chunks for the routes until we navigate to the specific route.

So if we would change the url to '/settings' what would happen is Webpack is going to insert a `<script src="/static/js/Settings.chunk.js"></script>` tag to the head of the document and after that the browser will fetch that specific resource meanwhile showing our specified Loading Component. Once the chunk is downloaded, parsed and executed it will replace our Loading Component.

## **Static code splitting with React.Suspense and React.lazy**

From React version 16 we can achive the same behaviour as above without using any third party library or writing our own async component with `React.Suspense` and `React.lazy`.

### **Dynamic splitting**

Dynamic splitting is a technique which is used to split code based on not static paths but instead on runtime conditions and variables.

Let's say we have the following directory structure and a getTheme function:

![Image alt text](./dynamic_split_dir_struct.png 'Dynamic Splitting directory structure')

```js
function getTheme(theme) {
  return import(`./src/themes/${theme}`);
}
```

We are using the same dynamic import function as before but in this case we don't specify the static path to the file instead we get the name of the file from the argument.

Webpack then will look at the static part of the import `('./src/themes')` and create a chunk for every file in that directory.

![Image alt text](./dynamic_split_dir_struct_after.png 'Dynamic Splitting directory structure after build')

Then we can use our `getTheme('dark')` function to download the theme we are looking for during runtime.

Dynamic splitting is primarly used for theming and A/B testing.

# Conclusion

The main purpose of code splitting is to reduce the initial size of the files our users have to download, parse and execute thus providing a faster looking app and better overall user experience.

Introducing code splitting to our app is relatively easy thanks to modern tools like Create React App and Webpack.
