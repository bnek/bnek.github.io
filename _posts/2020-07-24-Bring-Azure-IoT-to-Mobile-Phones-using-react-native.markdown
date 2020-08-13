---
layout: post
title:  "Bring Azure IoT to mobile phones with react-native"
date:   2020-06-10 16:16:06 +1000
categories: code
---
tl;dr if you want to skip the boring blurbs, just skip straight to `steps to reproduce`.

## the goal
I'm personally interested in the IoT space and after a few experiments with Azure IoT Hub and some samples I wanted to run some code on a real device rather than simulating one from the console. I have a spare Android mobile phone and I thought it would be great to deploy and run some code on that as a PoC.

#### React-native
Time is precious, so I did not want to spend a lot of time on boiler plate / infrastructure set-up, so I chose `react-native` as a base as I'm already familiar with it and it is easy to set up and it pretty much provides you with a deployable app shell with a few commands on the CLI. Also, this would open up avenues to iOS devices later.

#### azure-iot-device sdk

For the same reason as above, I did not want to write any custom code to interact with the IoT Hub but rather make use of Microsoft's official [azure-iot-device sdk](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-sdks#azure-iot-hub-device-sdks) for javascript.

## the challenge
The `azure-iot-device sdk` needs a node-js runtime, which does not exists on Android. 

After some googlin' I found out there is a pretty active project called [nodejs-mobile](https://github.com/JaneaSystems/nodejs-mobile) and even better - they published a react-native plugin: [nodejs-mobile-react-native
](https://github.com/janeasystems/nodejs-mobile-react-native). It allows you to create a node-js aplication project and run it alongside your React-Native project. It provides an api for inter-process communication between the React-Native app and the node-js app (which is running in a different thread). Both, the node-js app and the node-js runtime are deployed alogside the React-Native application bundle (because they're part of it).

Now that we've established that we can run code that requires a node-js runtime alogside a react-native app, it should be possible to have an android device talking to Azure IoT Hub via javascript sdk with only writing a few lines of code, right? So let's try it out.

## steps to reproduce
* Worked for me on Windows 10 and node v12.18.0 x64
* Create the app shell React-Native project
  * Follow the instructions on '[getting started with react native](https://reactnative.dev/docs/0.61/getting-started)' - make sure you use the 'React Native CLI Quickstart', not Expo which is default.
  * I chose to use TypeScript: `npx react-native init ReactNativeAndIot --template react-native-template-typescript`
  * `cd ReactNativeAndIot`
  * Start the Android Emulator
  * Make sure you can run the app in the simulator after it has been generated: `npx react-native run-android`
* Install nodejs-mobile-react-native: `npm install --save nodejs-mobile-react-native`
* in the `nodejs-assets` folder, remove the `sample-` prefix from the files so that you have a `main.js` and a `package.json` file. The `main.js` file becomes the entry point for the node-js application running alongside the react-native app.
* replace the content of `App.tsx` with the following content (which will start the node-js app, listen for messages and display them in a `Text` component).

    ```tsx
    import React, { Component } from 'react';
    import {
        SafeAreaView,
        ScrollView,
        View,
        StatusBar,
        Text,
    } from 'react-native';
    import nodejs from 'nodejs-mobile-react-native';

    class App extends Component<{}, { logs: string[]; message: string }> {
    constructor(props: any) {
        super(props);
        this.state = { logs: ['starting'], message: '' };
    }

    componentWillMount() {
        nodejs.start('main.js');
        nodejs.channel.addListener(
        'message',
        (msg) => this.log(msg),
        this,
        );
        this.log('done, waiting for messages from channel');
    }

    public render() {
        return (
        <>
            <StatusBar barStyle="dark-content" />
            <SafeAreaView>
            <ScrollView contentInsetAdjustmentBehavior="automatic">
                <View>
                {this.state.logs.map((s, i) => (
                    <Text key={`log${i}`}>{s}</Text>
                ))}
                </View>
            </ScrollView>
            </SafeAreaView>
        </>
        );
    }

    private log(msg: string) {
        this.setState({ logs: [...this.state.logs, msg] });
    }
    }
    export default App;
    ```
    the resulting output should be as follows: 
    ![Screenshot 1]({{ site.url }}/assets/images/2020-07-24-iot-react-native/screen1.png =250x)