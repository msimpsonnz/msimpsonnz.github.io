---
layout: post
title: Prototype with PowerApps
---

## Recycle or not

I have been working on a presentation for an upcoming Microsoft event in Auckland later this month. The FutureNow event is focused on data and AI, I have been asked to talk about building Cognitive Services into apps.

I really wanted to build something using the [Custom Vision](https://customvision.ai/) service to classify different types of waste products and if they could be recycled or not.

It is recycling awareness month here in NZ at the moment and there has been a big push for us all to be even more environmentally conscious.

Custom Vision allows you to bring a couple of images (minimum 5 for each label) and combine this with the Microsoft Computer Vision models to build your own custom classifier.

That was the easy part! I snapped some pics of the recycling before I took it outside over a few nights and uploaded them to the service. I setup my labels and had a working classifier in less than an hour, but the feedback loop was too slow, I had to take the photo on mobile and then use my laptop to upload them and really what I wanted was a mobile app that could do all of that.

## Enter PowerApps

[PowerApps](https://powerapps.microsoft.com/en-us/) is a business application builder! It provides a managed service to build applications using a drag and drop approach and built in connectors to support calling external services. It isn't even that new, previewed April 2016 and released later that same year.
The design canvas is web based and has loads of built in UI controls, navigation and APIs that expose device functionality. Like the camera API....perfect! It also supports running on Windows, Android and IOS....even better!!

As per above, there are 'connectors' and these are basically prebuilt code you can enable in your PowerApp which saves having to handcraft API calls to external services. All the Cognitive Services have REST endpoints but I didn't have to worry about that, used the [Custom Vision connector](https://docs.microsoft.com/en-us/connectors/cognitiveservicescustomvision/), walked through the auth flow in the browser and done. All I need is to provide my Customer Vision model id in the app and done.

So I spin up a new PowerApp, mobile format and then drag the camera control on to the design canvas and wire up the 'OnClick' event to my connector. No when I click the camera to take a picture, PowerApp will send the image off to Customer Vision and classify it.

<script src="https://gist.github.com/msimpsonnz/dd1645f77367981db26bba10613ea9f1.js"></script>

This how the PowerApp ended up looking

[<img src="{{ site.baseurl }}/images/2018-10-19-PowerApps/power1.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-10-19-PowerApps/power1.png)

Then a shot of it in action

[<img src="{{ site.baseurl }}/images/2018-10-19-PowerApps/power2.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-10-19-PowerApps/power2.png)

All up I was super impressed how easy it was to get going with this and no joke I had this all done in less that two hours. I have prior experience with both products so that's probably not the most useful statement but I honestly believe you could get something going in half a day from scratch.

Enjoy!