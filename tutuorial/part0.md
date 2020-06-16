# Part 0 - Setting Up

## Prior Knowledge

This tutorial assumes some basic familiarity with programming in general and Javascript in particular. There are many free resources online about learning programming and Javascript. I would recommend you spend some time getting a handle on the basics at least before proceeding. In additional to javascript you should be familiar with using the command line, git, and github.

That said - who am I to tell you what to do? Feel free to ignore that last paragraph and carry on!

## Installing Node

If you already have node installed on your machine, you're good to go. If not I recommend installing it with nvm. You'll thank yourself later, I promise.

For this tutorial I will be using node v12.13.1.

[Installation instructions for nvm and node are available here.](https://github.com/nvm-sh/nvm#installing-and-updating)

## Editors

Any text editor will work for Javacript. I personally use Visual Studio Code. In the past I have used Atom and Sublime. It really doesn't matter what editor you use so long as it helps you to be productive. So just pick one if you don't already have a daily driver.

## Initialize The Repo

Let's create a folder for our game and initialize the repo with npm.

`mkdir ./gobs-o-goblins && cd ./gobs-o-goblins`

`npm init`

Go ahead and accept all the defaults for now.

## Quality of Life Dev Tools

We'll be using npm to install our dependencies as our project grows. I like to pin the dependencies to an exact version. This ensures we always know exactly what third party libraries are installed in our project. To do this we just need to add simple npm settings file.

Create the file `.npmrc` at project root and paste the following:

```
save-exact=true
```

Next, I like to use prettier to format my code. It's super opinionated about code style so I don't have to be. I use the default settings so all we need to do after installing it is tell it what directories not to manage.

First let's install prettier as a development dependency:

`npm install --save-dev prettier`

Next lets add an ignore file to let prettier know what directories it can leave alone.

Create the file `.prettierignore` at project root and paste the following:

```
/dist
/node_modules
```

Git can ignore those directories too so we'll also add a gitignore file with the same contents.

Create the file `.gitignore` at project root and paste the following:

```
/dist
/node_modules
```

Ok! Now is a good time to save our progress. Let's initialize our git repo and make our first commit.

```
git init
git add .
git commit -m 'init'
```

## Webpack

OK. This next bit is a necessary evil. Modern web development requires build tools. You can try using config-free build tools but they aren't really config-free - just config-hidden - often an abstraction layer on top of webpack. The moment you have to do that one thing that's just a bit different from the "config-free" tool you're using... you have my pity.

We're going to rip off the band-aid and just write our own webpack config. You're welcome to read through the entire file and make any changes you want or need as your project grows but for now we're just gonna install some dev dependencies, paste some code to a new file and get on with it.

Ok - go ahead and install this list dev-dependencies:

```

```

```javascript
const path = require("path");
const { CleanWebpackPlugin } = require("clean-webpack-plugin");
const HtmlWebpackPlugin = require("html-webpack-plugin");
const GitRevisionPlugin = require("git-revision-webpack-plugin");
const TerserPlugin = require("terser-webpack-plugin");

const gitRevisionPlugin = new GitRevisionPlugin();

const mode = () => {
  if (process.env.NODE_ENV === "development") {
    return { mode: "development" };
  }

  if (process.env.NODE_ENV === "production") {
    return { mode: "production" };
  }

  return {};
};

const devtool = () => {
  if (process.env.NODE_ENV === "development") {
    return { devtool: "inline-source-map" };
  }

  if (process.env.NODE_ENV === "production") {
    return { devtool: "source-map" };
  }

  return {};
};

const devServer = () => {
  if (process.env.NODE_ENV === "development") {
    return {
      devServer: {
        contentBase: "./dist",
        open: false,
      },
    };
  }

  return {};
};

module.exports = {
  ...mode(),
  ...devtool(),
  ...devServer(),

  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          // because it's jacking up my property names and breaking code && mangle.properties = false don't do shit
          // probably related to babel class properties...
          mangle: false,
        },
      }),
    ],
  },

  entry: "./src/index.js",

  plugins: [
    new CleanWebpackPlugin({ cleanStaleWebpackAssets: false }),
    new HtmlWebpackPlugin({
      title: "Snail 6",
      template: "index.html",
      version: gitRevisionPlugin.commithash().slice(0, 7),
    }),
  ],
  output: {
    filename: "[name].[contenthash].js",
    path: path.resolve(__dirname, "dist"),
  },

  module: {
    rules: [
      {
        test: /\.(png|jpe?g|gif)$/i,
        use: [
          {
            loader: "file-loader",
          },
        ],
      },
      {
        test: /\.m?js$/,
        exclude: /(node_modules)\/(?!geotic)/,
        use: {
          loader: "babel-loader",
          options: {
            presets: ["@babel/preset-env"],
            plugins: [
              "@babel/plugin-proposal-class-properties",
              "@babel/plugin-proposal-private-methods",
            ],
          },
        },
      },
      {
        test: /\.s[ac]ss$/i,
        use: [
          // Creates `style` nodes from JS strings
          "style-loader",
          // Translates CSS into CommonJS
          "css-loader",
          // Compiles Sass to CSS
          "sass-loader",
        ],
      },
    ],
  },
};
```
