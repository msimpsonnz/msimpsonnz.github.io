---
layout: post
title: Web Assembly as the next UI Framework
summary: A look a up and coming projects using Web Assembly
---

## Using the browser as defacto client

[Web Assembly](https://webassembly.org) (WASM) was first announced way back in 2015, since then it has continued to gain traction and more recently 1.0 has shipped with support for all 4 of the major browsers - Chrome, Edge, Firefox and WebKit

Why do I care? This allows us to use a language that we might be more familiar with to compile down to a binary that we can ship to the browser to run.

I see a lot of divided opinions on this - browser is not the place for all applications and we don't need another tool to generate HTML, CSS and JS!

But this is happening, there are numerous projects happening around the place:
*[AutoCAD running in Chrome](https://www.youtube.com/watch?v=BnYq7JapeDA)
*[Blazor](https://github.com/aspnet/Blazor)
*[Ooui](https://github.com/praeclarum/Ooui)
*[Uno](http://platform.uno/)
*[Mono](https://www.mono-project.com/news/2018/01/16/mono-static-webassembly-compilation/)

I personally like the Xamarin Forms approach from Ooui, I know this is like inception with another layer using Forms but I think being able to go cross plat including web is really attractive.

I have played with both Blazor and Ooui and whilst it is still early days it feels like this is finally heading in the right direction.