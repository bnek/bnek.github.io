---
layout: post
title:  "From gulp to webpack"
date:   2020-06-10 16:16:06 +1000
categories: code
---
## Intro bla
`webpack` brought much convenience into web development. Together with `npm` or `yarn`, `webpack` enables us to be productive on a new project from day 1. However, there are still heaps of projects around that were built before webpack came along relying on ageing tools like i.e. `gulp` or `grunt` sometimes even using the outdated package manager `bower`.

Over time it becomes increasingly difficult to maintain a working build/deployment pipeline, updating dependencies i.e. to address exposed vulnerabilities, or simply adding new functionality to the website using latest dependencies which i.e. might not be available though `bower`.

In this post I'll run  through the steps necessary to migrate an angularjs (1.7) web app from gulp/bower to webpack.

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

webpack can be provided with one or more `loaders` for each file type used in the project. 

* _JS Files_  - have been loaded with `eslint-loader` and a modified version of [angular-template-loader](https://github.com/dgsmith2/angularjs-template-loader/blob/master/index.js) that replaces angular template URLs with a `require` call to load the templte inline. Basically it replaces `templateUrl: 'path/to/foo.html'` with `template: 'require('path/to/foo.html')` to prevent angular from wanting to load the template with a separate request.
  ```js
  {
      test: /\.js$/,
      exclude: [/node_modules/, /submodules/],
      loader: ['eslint-loader', angularJsTemplateLoader],
  },
  ```
* CSS / Less Files
I have used the below config to load css / less sources
  ```js
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



------

### Running the dev server
This project's development configuration relied on the single page app (SPA) being served with the backend from one server (rather than having a separate dev server running for the frontend like it's popular these days).

The proxy function of the webpack dev server comes in handy as it can be configured to transparently route requests to a certain path to another host/port. This way, the webpack dev server can be used separrately and the configuration of the SPA does not need to be changed.

```js
config.devServer = {
    contentBase: `./${conf.paths.dist}`,
    // dev server runs on port 5000
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