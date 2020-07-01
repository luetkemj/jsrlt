# Part 0 - Setting Up

## Prior Knowledge

This tutorial assumes some basic familiarity with programming in general and Javascript in particular. There are many free resources online about learning programming and Javascript. I would recommend you spend some time getting a handle on the basics at minimum before proceeding. In addition to javascript you should be familiar with using the command line, git, and github.

That said - who am I to tell you what to do? Feel free to ignore that last paragraph and carry on! It's just typing right?

## Assumptions

You will need a github account and git installed on your machine.

[Install git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

If you don't already have node installed on your machine I recommend installing it with nvm. You'll thank yourself later, I promise. For this tutorial I will be using `node v12.13.1`.

[Install nvm and node](https://github.com/nvm-sh/nvm#installing-and-updating)

## Editors

Any text editor will work for Javacript. I personally use [Visual Studio Code](https://code.visualstudio.com/). In the past I have used Atom and Sublime among others. It really doesn't matter what editor you use so long as it helps you to be productive. Just pick one if you don't already have a daily driver.

## Making sure everything works

To begin you will need to clone the basic starter project. It includes a dev-server for local development, a preprod environment for testing your build, and a deploy script for github pages so you can host your game online for free.

```bash
git clone git@github.com:luetkemj/jsrlt-starter.git gobs-o-goblins
cd ./gobs-o-goblins
```

### Installation

There are bunch of devDependencies we need to install before we can run the app. None of these will end up in the production build so don't worry about bloat.

`npm install`

### Dev

This project uses a webpack development server for local development. Run `npm start` to boot up the server. After a moment you should be able to access your project at [http://localhost:8080/](http://localhost:8080/).

You should see "Hello World" and belo that "Build:" and some gibberish next to it. That gibberish is the current git hash. With that you can always pinpoint exactly what code is running. Very useful for debugging.

The development server comes with hot reloading and will reload the app automatically on save. To give it a try go ahead and make the following change in index.html:

```diff
- <p>Hello World</p>
+ <p>Gobs O' Goblins</p>
```

Save your changes and the browser will reload automatically. Cool!

Don't forget to commit your changes to git. Stop the server with `control + c`

```bash
git add .
git commit -m 'my first commit'
```

Start the server back up `npm start` and the git hash should have changed. Cool!

### Preprod

Sometimes there are subtle differences between the raw code and what gets compiled for production. "It works on my machine" is a common developer excuse but it doesn't do your fanbase any good. This project includes a preprod environment for testing your compiled code locally before deploying it to production.

Run `npm run preprod` to create a production build and serve it locally. After the build is complete your browser should automatically open a new tab running the app. Go ahead and try it!

### Deploy

Finally we will create a repo for our project on github, push everything and then deploy it to production.

To start, go to github and create a new repository. Name it `gobs-o-goblins` and click `Create repository`.

We can now push our local code to the new remote. Github has some code snippets for quick setup. We have an existing repository so copy the second snippet under `…or push an existing repository from the command line` it looks like this:

```bash
git remote add origin git@github.com:your-github-username/gobs-o-goblins.git
git push -u origin master
```

Don't forget to replace `your-github-username` with your actual github username :)

We will be using github pages to host our project. The deploy script is already setup for you so all you have to do now is run `npm run deploy`. This will generate a production build and deploy it to a gh-pages branch in your github repo.

If everything has worked so far you should be able to see your app running at `https://your-github-username.github.io/gobs-o-goblins/`

## About this tutorial

Code snippets will be presented in a way that tries to convey exactly what you should be adding to a file at what time. When you are expected to create a file scratch and enter code into it, it will be represented with standard Javascript code highlighting, like so:

```javascript
import { Component } from "geotic";

export default class Health extends Component {
  static properties = { current: 10 };
}
```

Most of the time, you’ll be editing a file and code that already exists. In such cases, the code will be displayed like this:

```diff
import { Component } from "geotic";

export default class Health extends Component {
-  static properties = { current: 10 };
+  static properties = { current: 10, max: 10 };
}
```

## Ready to go?

Once you’re set up and ready to go, you can proceed to [Part 1](https://github.com/luetkemj/jsrlt/blob/master/tutorial/part1.md).
