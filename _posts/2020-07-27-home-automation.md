---
layout: post
title:  "Hey Siri, I wanna sleep!"
author: sandro
categories: [ Projects ]
image: assets/images/tradfri-gateway.jpg
tags: [featured]
---
Have you ever lain in your bed, already snuggled up and then you forgot you turn off a lamp, laptop or any other annoying device you cannot control from your bed? Well, we live in the 21st century, so we should be able to have a solution for that problem, don't we? 

Yeah, I thought that too and then I realized you have to buy one of those very pricy home automation systems from Apple. And then you realize - Oh we live in Switzerland with 'one-of-a-kind-power-plugs' that only exist in Switzerland. And guess what, a pretty small country and therefore a pretty small market. Well, luckily there are other brands that actually produce Swiss-smart-plug hardware but then they probably have a crappy app to control the whole thing. And your dream to say...
> Hey Siri, I wanna sleep! 

...is all gone again.

Well guess what, I am a developer and have some COVID-time to spend, so put up your sleeves and follow along if you wanna safe some bedtime. 

**DISCLAIMER** There is no solution for cleaning your teeth. You still have to do that yourself. You can stop reading now if that is what you want. 

## What you will need
Well there is some hardware involved. My research was not extraordinary, but I did search some cheap plugs with basically an ‘open’ or ‘hackable’ API and I got myself the following:
- [IKEA Trådfri Gateway - 40 CHF](https://www.ikea.com/ch/de/p/tradfri-gateway-weiss-40337806/)
- [IKEA Trådfri Plugs (Swiss) - 15 CHF](https://www.ikea.com/ch/de/p/tradfri-steckdose-funkgesteuert-00473650/)

If you are one of those IKEA heavy users, you might have some discount because of a family card or whatever. 

## Trådfri Setup
Once bought you will own approximately one hundred IKEA manuals/booklets with no useful information in it but translated in 6,500 languages or so. The only useful bit is to download an app from the App Store called 'Home smart'. After you've done that you have to setup your Gateway that goes something like this:
1. Unpack, plugin gateway to power and ethernet
2. Download and open 'Home smart' app and follow instructions
3. It will ask you to scan the QR-Code in the back of your gateway and you should be connected

### Connect plug without switch
Yeah, in IKEA's marketing departments there are working some clever guys. Because the Home smart app actually asks you what you want to connect, and the option 'Plug' is not available. What? Yeah, IKEA whant you to buy a remote to pair your devices. Of course there is no information on their website when you buy this, so you probably forgot this, as I did. So here is hack how you can connect your plug without a switch:

1. Plugin your plug nearby your gateway (20cm or so...)
2. Reset the plug by pushing a pin into the small opening next to the LED icon for 5 secs or so until the LED starts blinking
3. Push the pairing button on your gateway for a couple of secs
4. Go into your app and click the settings icon
5. Search for disconnected devices (you should now find your plug there)
6. Move plug into one of your rooms by drag and drop (if you don't have room create one)
7. Your plug should now be ready to use

> This should theoretically also work for lamps and other devices, but I did not test this. 

Ok, we can now control our plug with the very useful IKEA Home Smart app. So, is there an option to connect the smart home app to Siri? 
<div style="width:100%;height:0;padding-bottom:58%;position:relative;"><iframe src="https://giphy.com/embed/wYyTHMm50f4Dm" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>
Nooope for sure not. Would be a bit too easy, woulnd’t it?

## Build your own Home Control API
So, we have to build something on our own. More precisely will build a small backend that can communicate with the IKEA gateway and control our plugs based on some API endpoints.

**TL; TR;** Here is the code to the end result: [Github](https://github.com/saendu/homecontrol)

### Build Express API backend
So, in order to do what we need we are going to use [Express](https://expressjs.com/) and a library called [node-tradfri-client](https://www.npmjs.com/package/node-tradfri-client) that basically does the communication with the IKEA gateway for us.

Connecting to our gateway we do something like this (you need the 16-digit security code on the back of your gateway):
```
const tradfriLib = require("node-tradfri-client");
const TradfriClient = tradfriLib.TradfriClient;
const AccessoryTypes = tradfriLib.AccessoryTypes;
const tradfri = new TradfriClient(process.env.GATEWAY_IP, {watchConnection: true});

const stringify = require('json-stringify-safe');

const lightbulbs = {};
const plugs = {};
const devices = {};

const connect = async () => {
  try {
    const {identity, psk} = await tradfri.authenticate(process.env.SECURITYCODE);
    await tradfri.connect(identity, psk);
    tradfri
      .on("device updated", tradfri_deviceUpdated)
      .observeDevices();
  } catch (e) {
    console.log(e);
  }
}

function tradfri_deviceUpdated(device) {
  console.log(`*** DEVICE (${device.instanceId}) UPDATED ***`);
  devices[device.instanceId] = device;

  if (device.type === AccessoryTypes.lightbulb) {
    lightbulbs[device.instanceId] = device;
  } else if (device.type === AccessoryTypes.plug) {
    //const {...plug} = device; 
    device.client.psk = null;
    device.client.securityCode = null;
    plugs[device.instanceId] = device;
  }
}
```

**Notice** I’ve used an .env file to inject my secrets. 

Now we have a `plugs` object that contains all our plugs. We can now control our plug by doing this:
```
// TOGGLE 
router.put('/:plugId/toggle', async (req, res) => {
  const plugId = req.params['plugId'];
  const plug = plugs[plugId];
  const {client, ...safePlugInfo} = plug; 
  
  try {
    await plug.plugList[0].toggle();
  }
  catch(error) {
    console.log(error);
  }

  res.json(safePlugInfo);
});
```

Now we basically have an endpoint `https://yourdomain.com/api/<plugID>/toggle` to turn your plug on/off. 
In order nobody that read this article wants to play Joker with me and turns my devices on/off from the outside, we are going to use a bit of basic authentication and write a middleware:
```
const basicAuth = require('express-basic-auth');

const authenticate = (req, res, next) => {
  try {
    // check for basic auth header
      if (!req.headers.authorization || req.headers.authorization.indexOf('Basic ') === -1) {
        return res.status(401).json({ message: 'Missing Authorization Header' });
    }

    // verify auth credentials
    const base64Credentials =  req.headers.authorization.split(' ')[1];
    const credentials = Buffer.from(base64Credentials, 'base64').toString('ascii');
    const [username, password] = credentials.split(':');
    const userMatches = basicAuth.safeCompare(username, process.env.ADMIN_USER);
    const passwordMatches = basicAuth.safeCompare(password, process.env.ADMIN_SECRET);

    if(userMatches & passwordMatches) {
      next(); // only pass when successful
    }
    else {
      return res.status(403).json({ message: 'Access denied: wrong user or password' });
    }
  }
  catch(err) {
    console.log(`Authentication error: ${err}`);
    res.status(500).json({ message: `Internal error: ${err}` });
  }
}

module.exports = authenticate;
```

**NOTE** that we are using basicAuth.safeCompare() and not a '==' or '===' comparison that no hacker can perform a [Timing attack](https://en.wikipedia.org/wiki/Timing_attack).
<div style="width:100%;height:0;padding-bottom:67%;position:relative;"><iframe src="https://giphy.com/embed/jUEuQU7RGyYsdnXHvF" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>

Last thing we need to do is using the middleware in our route:
```
// API Routes
app.use(`/${APP_URL_PREFIX}/plugs`, authenticate, require('./routes/api/plugs'));
```

That's it. Now you can create a PUT call with your Basic <Token> to your endpoint and toggle your lamp on/off.

## Bring Siri to the game
The next thing we wanna do is create a Siri shortcut in order to execute that PUT call for us. For this we basically need two steps:
1. A ‘URL’ step
2. A ‘Get contents of URL’ step

![walking]({{ site.baseurl }}/assets/images/siri-shortcut.jpg)

Make sure that you replace the URL & the Authorization secret provided in your backend. 

That's it, now you can toggle your lamp by saying:
> Hey Siri, toggle lamp.

So, if your room now has multiple smart plugs accessible via your backend you can basically create some more steps in your Siri shortcut and say:
> Hey Siri, I wanna sleep!

And this time, it is not a dream anymore!  

Until next time . . .

#### Acknowledgements 
Here some articles that help get through all of this:
- [Homecontrol - Saendu Github](https://github.com/saendu/homecontrol)
- [node-tradfri-client](https://www.npmjs.com/package/node-tradfri-client)
- [NodeJS Basic Authentication Example](https://jasonwatmore.com/post/2018/09/24/nodejs-basic-authentication-tutorial-with-example-api)

:heart:
