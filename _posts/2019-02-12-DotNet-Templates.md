---
layout: post
title: Templates for .NET Console App
summary: A quick guide on how to create a template project for .NET applications, focusing on .NET Console App and Generic Host service
---

### Every day I'm ~~hustling~~ consoling

This is a bit of a "you're basic" post but I thought I would share all the same.

I end up writing a LOT of .NET Console Apps...a quick test here, a demo there and some automation every now and again.

Every single time I do `"File => New Project"` or `dotnet new console`...well NOT TODAY!

I've been really trying to share most of what I do through GitHub and I have been making a conscious effort to this using some basic principals...no hard coded secrets in code.

There is actually more of a story here, a couple of years back I would use Azure DevOps (VSTS back then) and found that private repos actually made me a lazy dev, nothing in config everything hard coded!! This is a shame as I there is a few project in there that would have been useful to make public but I ran out of time and enthusiasm to kill all the git history and migrate them before they got stale.

It wasn't like I was writing anything that made it close to production but I wanted to start treating my code like this so I could better relate to what others were having to deal with.

Ok...I'll get to the point soon, stick with me this is good for the soul.

So I need to build a test or in this case a repro for a problem that we are seeing, right `dotnet new console`

Then I would spend time reading about how to do configuration in .NET Core again and reading the same [article](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.2) and scaffolding up all the things...it was Groundhog Day every time!

After finding out about [.NET Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-2.2) which was introduced in 2.1, I decided it was time to build a project template that I could reuse.

[Custom Templates](https://docs.microsoft.com/en-us/dotnet/core/tutorials/create-custom-template) are super easy, I'm mad I didn't spend time on this before as I was only a couple of hours.

So I create my Console App with all the setup I need and just a `.template.config` folder and `template.json` file and i can run `dotnet new --install <<Path to Project folder>>`

Couple of things I found out:
* Make sure you get rid of any build folders `obj` and `bin` need to be gone otherwise you just get compiled `dll` files when you create a new project from the template
* If you are installing the template from the local machine, drop the trailing `\` on the folder path otherwise the install will fail
* Use `"preferNameDirectory": "true"` in `template.json` so the project gets named after the parent directory or --name parameter
* Use `"sourceName": "<<Your Namespace>>"` in `template.json` to rename the namespace after the parent directory or --name parameter

That's it, hope this helps someone, but will probably just end up be a reference for me when I come back and update this to .NET Core 3.0!

Link to the GitHub repo is [here](https://github.com/msimpsonnz/dotnet-templates)
Link to Nuget package is [here](https://www.nuget.org/packages/MSimpson.ConsoleTemplate.CSharp/1.0.0)