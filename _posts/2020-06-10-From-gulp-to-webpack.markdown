---
layout: post
title:  "From gulp to webpack"
date:   2020-06-10 16:16:06 +1000
categories: code
---
## Intro
`webpack` brought much convenience into web development. Together with `npm` or `yarn`, `webpack` enables us to be productive on a new project from day 1. However, there are still heaps of projects around that were built before webpack came along relying on ageing tools like i.e. `gulp` or `grunt` sometimes even using the outdated package manager `bower`.

Over time it becomes increasingly difficult to maintain a working build/deployment pipeline, updating dependencies i.e. to address exposed vulnerabilities, or simply adding new functionality to the website using latest dependencies which i.e. might not be available through `bower`.

In this post I'll run  through the steps necessary to migrate an angularjs (1.7) web app from gulp/bower to webpack. Since the migration is expected to be ongoing while development is happening, the emphasis is also on trying to migrate with minimal changes to the source code to avoid merge conflicts when rebasing onto newer versions.

## Step 1 - Getting rid of bower / gulp and adding webpack
This project used both, bower and npm to manage dependencies. The first step is to bring all dependencies into npm. 
for each dependency in `bower.js` run `npm install --save <dependencyname>`.

Some packages were not available in the required version anymore on `npm` so I had to install them using the github URL guidance can be found [here](https://docs.npmjs.com/cli/install).

Then I removed gulp from `package.json` and added webpack 
```
npm install --save-dev webpack webpack-cli
```
------

## Step 2 - Successfully bootstrapping the angular app
I took a look at the [webpack docs](https://webpack.js.org/guides/getting-started/) to brush up my knowledge on how it basically works - but basically it comes down to including all sources that have been used by the previous bundler (gulp) and have webpack emitting them in a bundle. After all it's a working app, so all we want to do is make it work the same way it worked before (just with a different bundler). 

------



### 2.1. Getting webpack to run
* add a [configuration](https://webpack.js.org/guides/getting-started/#using-a-configuration): webpack.config.js
* add a script within `package.json` to build the app
    ``` json
    {
        //...
        "scripts" {
            //...
            "dev": "webpack-dev-server --watch --config webpack.config.js"
        }
        //...
    }
    ```
* add an `index.js` entry point for webpack
* add an `index.html` template to inject the emitted bundles into (optional) (I used the old gulp template because it had links to external scripts and other required code).

### 2.1.1 Running the dev server
This project's development configuration relied on the single page app (SPA) being served with the backend from one server (rather than having a separate dev server running for the frontend like it's popular these days).

The proxy function of the webpack dev server comes in handy as it can be configured to transparently route requests to a certain path to another host/port. This way, the webpack dev server can be used separately and the configuration of the SPA does not need to be changed.

```js
config.devServer = {
    contentBase: `./${conf.paths.dist}`,
    port: 5000,
    proxy: {
        // api requests to the path '/client' will be proxied to a different host
        '/client': {
            target: 'https://localhost:44300',
            secure: false,
            changeOrigin: true
        },
    },
}
```

------



### 2.2 Including all the dependencies 
Now that I can run webpack using `npm run dev`, the next step is to add the whole app to `index.js`. To do that I looked at an emitted `index.html` from the old gulp pipeline and added a `require('path/to/resource')` for each resource that was included in the template.
``` html
<!-- css libraries //-->
<link rel="stylesheet" href="bower_components/Hover/css/hover.css" />
...
<!-- local css files (built from less) //-->
<link rel="stylesheet" href="./app/tmp/site.css">
  ...
<!-- dependencies //-->
<script src="bower_components/angular/angular.js"></script>
  ...
<!-- local js files //-->
<script src="./app/app.js"></script>
```
becomes
```js
// css libraries
require('Hover/css/hover.css');
  ...
// local less files
require('./content/site.less');
  ...
// dependencies
require("angular");
  ...
// local js files
require("./app/app.js");
```
------



### 2.3 Getting webpack to load all the files

Below is the configuration for each of the [loaders](https://webpack.js.org/concepts/loaders/) to successfully 

* _**JS Files**_  - have been loaded with `eslint-loader` and a modified version of [angularjs-template-loader](https://github.com/dgsmith2/angularjs-template-loader/blob/master/index.js) that replaces angular template URLs with a `require` call to load the template inline. Basically it replaces `templateUrl: 'path/to/foo.html'` with `template: 'require('path/to/foo.html')` to prevent angular from wanting to load the template with a separate request. 
  
  Alternatively, this can be done by hand or 'search and replace' as well. However, if development goes on during the migration, merge conflicts become more likely when changing the sources too much.
  ```js
  const angularJsTemplateLoader = {
      loader: path.resolve(__dirname, "<path/to>/angular-template-loader.js"),
      options: {
          relativeTo: './'
      }
  };

  // ...

  {
      test: /\.js$/,
      exclude: [/node_modules/, /submodules/],
      loader: ['eslint-loader', angularJsTemplateLoader],
  },
  ```
* _**CSS / Less Files**_ loader config:
    ```js
    const MiniCssExtractPlugin = require('mini-css-extract-plugin');
    ...

    {
      test: /\.css$/,
      use: [
          'style-loader',
          {
              loader: MiniCssExtractPlugin.loader,
          },
          {
              loader: 'css-loader',
              options: {
                  sourceMap: true, importLoaders: 1
              }
          },
          {
              loader: 'postcss-loader', options: {
                  ident: 'postcss', sourceMap: true,
                  plugins: () => [require('autoprefixer')]
              }
          }
      ]
  },
  {
      test: /\.less$/,
      use: [
          'style-loader',
          {
              loader: MiniCssExtractPlugin.loader,
          },
          {
              loader: 'css-loader',
              options: {
                  sourceMap: true,
                  importLoaders: 2
              },
          },
          {
              loader: 'postcss-loader',
              options: {
                  ident: 'postcss',
                  sourceMap: true,
                  plugins: () => [require('autoprefixer')]
              }
          },
          {
              loader: 'less-loader',
              options: {
                  sourceMap: true,
              },
          },
      ]
  }
    ```
    
* _**Static resources / images (required in less files)**_ - Some of the images would be 'required' by the less files, in which case they'd be picked up by webpack through the `file-loader` as configured below.
    ```javascript
  {
      test: /\.(png|jpg|jpeg|gif|svg|woff|woff2|ttf|eot)$/,
      use: {
          loader: 'file-loader',
          options: {
              esModule: false,
              name: '[name].[ext]',
              outputPath: (url, resourcePath, context) => {
                  const relativePath = path.relative(context, resourcePath);
                  return relativePath.replace(`${conf.paths.src}\\`, '');
              },
              publicPath: (url, resourcePath, context) => {
                  const relativePath = path.relative(context, resourcePath);
                  return relativePath.replace(`${conf.paths.src}\\`, '').replace(/\\/g, '/');
              },
          }
      },
  },
    ```
* _**Static resources / images (NOT in less files)**_ - There are plenty of references to resources (i.e. images references  from the template, etc) that were not being picked up by webpack. I made those available by using the CopyPlugin. To match the original file structure, I had to copy the files one directory up of the output directory.
    ```javascript
  config.plugins = [
    ...
        new CopyPlugin([{
            from: `./${conf.paths.src}/**/*.{ico,json,png,jpg,jpeg,gif,svg,woff,woff2,ttf,eot}`,
            to: './',
            ignore: [/*'ignore.me'*/],
            transformPath(targetPath, absolutePath) {
                const srcPrefix = conf.paths.src.startsWith('./') ? 
                  conf.paths.src.substring(2) : conf.paths.src;
                return targetPath.replace(`${srcPrefix}`, '..');
        },},]),
      ...
    ]
    ```
* _**Custom HTML template**_ - using [HtmlWebpackPlugin](https://webpack.js.org/plugins/html-webpack-plugin/), the original template could be used to inject the bundle. I also had to place this one directory above the actual output of webpack to make it work with the original configuration.
    ```javascript
  config.plugins = [
    ...
        new HtmlWebpackPlugin({
            template: path.join(conf.paths.src, './index.html.tpl'),
            inject: true,
            filename: '../index.html'
        }),
      ...
    ]
    ```
* _**Expected global References (like jQuery)**_ - The [ProvidePlugin](https://webpack.js.org/plugins/provide-plugin/) can be used to automatically load modules when a variable from the module is accessed to prevent the need to `require` the respective module everywhere.
    ```javascript
  config.plugins = [
    ...
      new webpack.ProvidePlugin({
          'window.moment': 'moment',
          $: "jquery",
          'window.$': "jquery",
          jQuery: "jquery",
          'window.jQuery': "jquery"
      })
      ...
    ]
    ```
------

## Conclusion
Migrating an angularjs app from gulp / bower to webpack is manageable as it can be done alongside development with minimal changes to existing source code.