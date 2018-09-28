---
layout: post
title:  "Debuggning Probot with Visual Studio Code"
date:   2018-09-28 13:00:00
categories: probot github dev
---

One of the [Zalando Open Source Teams](https://opensource.zalando.com/) responsibilities is to ensure that our use of Github is Compliant and accesible to our engineering teams.
One way to ensure this is to build tooling and automation which supports our compliance rules. 

[Probot](https://probot.github.io/) is an excellent framework for building Github bots, but it's been challenging to figure our how to monitor activities in the bot and debug it with VS codes breakpoints. Out of the box Probot uses nodemon, which gives me a decent output to the terminal, but it feels like going back in time to `console.log` based development

## Configuring VSCode for debugging
As I'm writing [a new bot in typescript](https://github.com/perploug/4eyesbot/) I want to have 2 things automated, compiling the Typescript to javascript, and then attaching the debugger to my project so I can add breakpoints and see what is going on when a Probot webhook is triggered. 

Luckily the probot [typescript template](https://github.com/probot/create-probot-app) comes with a good starting point, after going through the wizard I configured a new Typescript compile task in `.vscode/tasks.json` :

{% highlight json %}
"tasks": [
    { 
      "identifier": "typescript",
      "type": "typescript",
      "tsconfig": "tsconfig.json",
      "problemMatcher": [
        "$tsc"
      ]
    }
  ]
{% endhighlight %}

I then used this task with an entry in the `.vscode/launch.json` config to ensure Typescript is compiled before the debugger is attached:

{% highlight json %}
"configurations": [
    {
      "type": "node",
      "request": "launch",
      "preLaunchTask": "typescript",
      "name": "Launch Probot",
      "program": "${workspaceFolder}/node_modules/probot/bin/probot-run.js",
      "args": ["./lib/index.js"],
      "console": "integratedTerminal",
      "cwd": "${workspaceRoot}/",
      "outFiles": [],
      "sourceMaps": true,
      "env": {}
    }
  ]
{% endhighlight %}

Notice the prelaunch task is my Typescript compiler and that I use `probot-run.js` from the Probot module to run `./lib/index.js` the index.js is the output of my typescript compile, based on the include .tsconfig file. 

## Debugging with breakpoints

With the combination of a compile task and a launch setting I can now simply hit F5 and the probot will start up locally, receive webhooks from Github through smee.io and any breakpoints set in VS Code will be hit with access to variables and all the other debugging goodness.


