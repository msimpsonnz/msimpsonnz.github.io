---
layout: post
title: Using AWS Amplify to build a QR scanner
summary: In this post we look at a rapid prototype to build a simple app that can run on a smart phone and scan QR codes to improve data entry
tags: [aws, amplify]
---

### Simplify data entry

I was presented with a challenge recently on how we could simplify data entry for an integration solution that tracks shipments. The idea was that if we could match a shipment with a QR code then we would only need two pieces of information, the shipment ID and the QR code of the package. We could collect these two pieces of info and perform a lookup on the backend and record the entry. Previously, the data for the shipment would need to be collected, logged and then entered manually into the system. This was error prone, introduced a delay getting up to date info and there could be a backlog of several days of data during busy periods.

As a very quick prototype we wanted to get an app that could run on a smartphone that had a list of shipments and could link this to physical items. The list of shipments would be looked up from a central system and the app would scan a QR code on the physical item and update the records.

Enter [AWS Amplify](https://aws-amplify.github.io/) which is my go to for getting up and running quickly. I used the [Vue.js starter](https://github.com/aws-samples/aws-amplify-vue) which scaffolds out a 'ToDo' list app.
I borrowed the ToDo list model and adapted this to build a couple of screens which uses AppSync GraphQL to persist and query data in DynamoDB. Then used an nice Vue library that has a QR code reader called [Vue QR Code Reader](https://github.com/gruhn/vue-qrcode-reader). Which has a really nice stream feature that triggers once the camera finds a QR code.

```html
</h4>
    <qrcode-stream @decode="onDecode" @init="onInit" />
<h4>
```

See the sample code [here](https://github.com/msimpsonnz/aws-misc/tree/master/amp-mob-vue), this was a couple of hours work so it has some rough edges, but a perfect prototype to get in the users hands and start the feedback loop.

