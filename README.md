# Ionic 5, React, Capacitor and Appflow Live Deploy
### Introduction
I have been building hybrid apps for the past 5 years, mainly with cordova and phonegap. January 2020 I got the opportunity to make a deep dive into Ionic, Capacitor and Appflow. It was mostly a good experience, but the lack of up-to-date documentation and a currently sparse developer community, I soon realized I was spending way to much time bugfixing the Ionic and Appflow glitches and oddities rather than just building cool hybrid apps. 

In order to save you from the endless headaches I have compiled this guide on how to build a solid hybrid app foundation using the latest technologies; Ionic 5, React, Capacitor and Appflow. 

### Prerequisites
In order to get Live Deploy working you need to have an active subscription on Appflow; [sign up here](https://www.ionicframework.com/) #notSponsored.

For more information about Appflow, and the benefits of Live Deploy, please checkout the [Ionic Deploy documentation](https://ionicframework.com/docs/appflow/quickstart/deploy).

You also need the Ionic CLI and [Git](https://github.com/git-guides/install-git) for source control. You can use the Ionic source control, but I would strongly suggest you go the extra mile and setup Git and the [Github](https://www.github.com/) integration from the start.

If you dont already have a github account, you should create one right now and set up a fresh repository for this project. And **write down the github repository URL**. Go on... I'll wait. It's important. 

Okay, to install the ionic CLI run this command in your terminal:
`npm install -g @ionic/cli`

### Getting started
First, you need to login to your ionic account from the terminal: 
`ionic login`
You will be redirected to the browser to verify your credentials. 

To link your local app with ionic hub, first create a new app in Ionic Hub and **write down the App Id**.

Then to fire up a fresh Ionic app with Capacitor and React support simply run this terminal command from within your myApp directory, substituting YOUR_APPID with your App Id you wrote down before:

`ionic start myApp tabs --capacitor --type react --id YOUR_APPID`

In order for ionic to link to your Github repository, you need to follow the instructions from the Terminal, and authorize Ionic to access your github repositories. 

Finally, to hook up your Github repo as a remote repository, so that Ionic can actually listen for updates to the code base, you need to connect your repo to the ionic git project:
`git remote add origin YOUR_GITHUB_REPO_URL`
To just see that it's working run ionic serve and check out the tabs in the browser. 

> ⚠️ NOTE: For the time of this writing (september 2020), you also need to install additional types, for typescript to work and the app to render: 
`npm install -D @types/request-promise-native @types/object-assign @types/long @types/lodash @types/caseless`

### Adding iOS and Android support

To add support for iOS and Android you simply run:

`ionic capacitor add ios`

and for Android:

`ionic capacitor add android`

⚠️ In order for Capacitor to actually get your Ionic app to load inside the native Android webview you also need to add the following attribute to the `<application>`-tag in the `AndroidManifest.xml`:

```xml
<application
    android:usesCleartextTraffic="true"
    ...
</application>
```

### Adding app icons and splash screens
We're gonna use the `cordova-res` package to handle the packaging, resizing and saving of our app icon and splash screen into the many different sizes and formats that both iOS and Android needs:
`npm install cordova-res --save-dev`

cordova-res expects a project structure such as:
```
resources/
  ├─ icon.png (at least 1024×1024px)
  └─ splash.png (at least 2732×2732px)
```

To use cordova-res with Capacitor, it is recommended to use --skip-config (which skips reading & writing to Cordova's config.xml file) and --copy (copies generated resources into native projects).

So to generate icons and splash screens for iOS and Android in Capacitor, run:
```bash
cordova-res ios --skip-config --copy
cordova-res android --skip-config --copy
```

### Adding Live Deploy
To link up the Appflow Live Deploy service, you first need to build the web assets. So run this command:
`ionic build`

And once completed, you can now hook up Appflow Live Deploy, like this (again substituting APPID with your App Id from Ionic Hub):
`ionic deploy add --app-id="APPID" --channel-name="master" --update-method="auto"`

### Testing the app on iOS and Android
In order for Live Deploy to actually work on both iOS and Android you need to access the app from a device or emulator. 
`ionic capacitor open ios`

And then in Xcode choose the device or emulator you want the app to be executed on (and wait for the indexing to complete * sigh*). 

Same story with Android:
`ionic capacitor open android`

And then in Android Studio select the device or emulator (in the AVD Manager) to execute on (and wait for gradle build and sync to complete *sigh*).

### Making sure Live Deploy works
Now, the moment you've all been waiting for. How do we now hook everything up to Live Deploy, so we can - within the iOS and Android apps we now have open - perform live updates to the app, which changes will reflect on the next boot of the app? 

Well, first off, you need to have a Web build assigned to a Live Deploy channel named "Master" in Ionic Hub. 

And then you need to make sure that the app, that you have now packaged (ionic build) and served locally within the native build, is visually different than the version that is available on the Live Deploy channel (so you can make a distinction whether or not the app presented is from local copy or fetched live from ionic Hub. 

Let's first get a build up on Ionic Hub. For that, we use Github. 

```bash
git add . 
git commit -m "first commit"
git push origin master
```

Then in Ionic Hub select the commit and pick "Create Build" and select "Web" as Target Platform. This will cause a cloud build/web build of your committed code. 

And make sure to also tick "Live Update" as destination, so we can actually perform live deploys to our apps. And select "master" as channel. 

Also, you need to add additional parameters to the info.plist (for IOS):
```xml
<key>IonMaxVersions</key>
<string>2</string>
<key>IonMinBackgroundDuration</key>
<string>20</string>
<key>IonApi</key>
<string>https://api.ionicjs.com</string>
```

and AndroidManifest.xml (for Android):

```xml
<string name="ionic_max_versions" >2</string>
<string name="ionic_update_api">https://api.ionicjs.com</string>
<string name="ionic_min_background_duration">20</string>  
 ```
Credits to [dotorimook](https://dev.to/dotorimook/important-to-know-for-ionic-appflow-s-live-update-28hc) for pointing this out.

Now, Live Deploy should work on both Android and iOS. 

### Adding A Splash Screen

The last thing we need now, is to add a nice splash screen, while the version check and download is taking place. [See the full documentation from Capacitor](https://capacitorjs.com/docs/apis/splash-screen).

We need to tell Capacitor, that we want to handle the Splash screen ourselves, and not rely on a static timer/delay. So add this to the root `capacitor.config.json` file:

```js
  "plugins": {
    "SplashScreen": {
      "launchAutoHide": false
    }
  }
```

To hide the splash screen simply add this code to your `App.tsx` inside your App component:
```js
import { Plugins } from "@capacitor/core";
import { Deploy } from "cordova-plugin-ionic";
    //...
  React.useEffect(() => {
    (async () => {
      const update = await Deploy.checkForUpdate();
      if (!update.available) Plugins.SplashScreen.hide();
    })();
  }, []);
```
> ⚠️ NOTE: For the time of this writing (september 2020), you need to make a manuel check of available updates, because of a bug in Ionic where the update method "auto" is ignored. 

And then just run `ionic capacitor sync` to build out the web app and populate it to the iOS and Android native builds.

To test things out you can run `ionic capacitor open ios` to launch the native iOS project in Xcode or `ionic capacitor open android` to open native Android project in Android Studio. 

The app should now hide the splash screen as soon as the app is loaded, while also showing the splash screen while any Live Deploy updates download and installs. Woop woop! :) 

### Next steps

So there you have it. A solid tech stack to build a hybrid app anno 2020. 

From here you can take it wherever you want to go. You could try playing around with Appflows automation tools, set up a CI/CD workflow or even building out the Live Deploy to support e.g. a opt-in beta channel for users. The sky's the limit - and now you have a strong foundation to start your hybrid app adventure. 

Show me what you've got... 
