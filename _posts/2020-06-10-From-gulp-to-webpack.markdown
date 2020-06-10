---
layout: post
title:  "From gulp to webpack"
date:   2020-06-10 16:16:06 +1000
categories: code
---
## Intro
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

## Step 2 - Successfully bootstrapping the angular app
I took a look at the [webpack docs](https://webpack.js.org/guides/getting-started/) to brush up my knowledge on how it basically works. Then I added.

* webpack.config.js
* a script within `package.json` to build the app
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

Now I was able to run the app inside the webpack dev server using the command `npm run dev`.