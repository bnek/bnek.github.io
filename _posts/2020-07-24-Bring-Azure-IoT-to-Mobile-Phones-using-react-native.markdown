---
layout: post
title:  "Bring Azure IoT to Mobile Phones with React Native"
date:   2020-06-10 16:16:06 +1000
categories: code
tags: azure iot hub device sdk react-native android ios mobile
---
TL;DR If you want to skip the blurbs, just go straight to [steps to reproduce](#steps-to-reproduce) or chceckout the [code on GitHub](https://github.com/bnek/react-native-and-iot).

## the goal
I'm personally interested in the IoT space and after a few experiments with Azure IoT Hub and some samples I wanted to run some code on a real device rather than simulating one from the console. I have a spare Android mobile phone and I thought it would be great to deploy and run some code on that as a PoC.

#### React Native
Time is precious, so I did not want to spend a lot of time on boiler plate / infrastructure set-up, so I chose React Native as a base as I'm already familiar with it and it is easy to set up and it pretty much provides you with a deployable app shell with a few commands on the CLI. Also, this would open up avenues to iOS devices later.

#### Azure IoT Hub device SDK for Node.js

For the same reason as above, I did not want to write any custom code to interact with the IoT Hub but rather make use of Microsoft's official [Azure IoT Hub device SDK](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-sdks#azure-iot-hub-device-sdks).

## the challenge
The Azure IoT Hub device SDK for Node.js needs a node-js runtime, which does not exists on Android. 

After some googlin' I found out there is a pretty active project called [nodejs-mobile](https://github.com/JaneaSystems/nodejs-mobile) and even better - they published a React Native plugin: [nodejs-mobile-react-native
](https://github.com/janeasystems/nodejs-mobile-react-native). It allows you to create a node-js aplication project and run it alongside your React Native project. It provides an api for inter-process communication between the React Native app and the node-js app (which is running in a different thread). Both, the node-js app and the node-js runtime are deployed alogside the React Native application bundle (because they're part of it).

Now that we've established that we can run code that requires a node-js runtime alogside a React Native app, it should be possible to have an android device talking to Azure IoT Hub via javascript sdk with only writing a few lines of code, right? So let's try it out.

## steps to reproduce
* Prerequisites
  * Windows 10 
  * [Node.js](https://nodejs.org/en/download/) v12.18.0 x64 (or compatible)
  * An [Azure IoT Hub / device created and a device connection string](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-through-portal) available to use to connect the app.
* Create the app shell React-Native project
  * Follow the instructions on '[getting started with react native](https://reactnative.dev/docs/0.61/getting-started)' - make sure you use the 'React Native CLI Quickstart', not Expo which is default.
  * I chose to use TypeScript: `npx react-native init ReactNativeAndIot --template react-native-template-typescript`
  * `cd ReactNativeAndIot`
  * Start the Android Emulator
  * Make sure you can run the app in the simulator after it has been generated: `npx react-native run-android`
* Install nodejs-mobile-react-native: `npm install --save nodejs-mobile-react-native`
* In the `nodejs-assets/nodejs-project` folder, remove the `sample-` prefix from the files so that you have a `main.js` and a `package.json` file. The `main.js` file becomes the entry point for the node-js application running alongside the react-native app.
* Replace the content of `App.tsx` with the following content (which will start the node-js app, listen for messages and display them in a `Text` component).

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
    ```
    starting
    done, waiting for messages from channel
    from node-js: Node was initialized.
    ```
* If you see an error in the metro server console relating to a `Duplicate module name` error, please follow the instructions [here](https://github.com/janeasystems/nodejs-mobile-react-native#duplicate-module-name).
* Now we can add the Azure IoT Hub device SDK:
* Change to the node-js project directory: `cd ./nodejs-assets/nodejs-project`
* Install Azure IoT Hub device SDK and HTTP transport package `npm install --save azure-iot-device azure-iot-device-http`
* We're now adding code to connect to IoT hub and to send/receive messages using the Azure IoT Hub device SDK using Http transport. Please note that you need your device connection string here to connect via `Client.fromConnectionString(...)` - **Please note that this is a PoC only and connection string should not be checked into source code. Also, for security considerations, I suggest to read "[Understand Azure IoT Hub security](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-security)" from the Microsoft Docs**
* In the `nodejs-assets/nodejs-project` folder replace the content of `main.js` with the following:
    ```js
    const Client = require('azure-iot-device').Client;
    const Protocol = require('azure-iot-device-http').Http;
    const Message = require('azure-iot-device').Message;
    var rn_bridge = require('rn-bridge');

    rn_bridge.channel.on('message', (msg) => {
        const message = generateMessage(msg);
        sendLog('Sending message: ' + message.getData());
        client.sendEvent(message);
    });

    function sendErr(theErr) {
        rn_bridge.channel.send('Err: ' + theErr);
    }

    function sendLog(theMsg) {
        rn_bridge.channel.send('Msg: ' + theMsg);
    }

    function messageHandler(msg) {
        sendLog('Received message: Id: ' + msg.messageId + ' Body: ' + msg.data);
        client.complete(msg);
    }

    function generateMessage(messageText) {
        const data = JSON.stringify({
            deviceId: 'iot-brick',
            text: messageText,
        });
        const message = new Message(data);
        return message;
    }

    const client = Client.fromConnectionString(
        '<YOUR DEVICE CONNECTION STRING>',
        Protocol,
    );

    client.on('connect', () => sendLog('Client connected'));
    client.on('error', (err) => sendErr(err.message));
    client.on('message', messageHandler);

    client.open().catch((err) => {
    sendErr('Could not connect: ' + err.message);
    });
    ```
* In the React Native app, we'll add a text field and button to send text-based messages to IoT Hub replace the content off the `App.tsx` with the following:
    ```tsx
    import React, { Component } from 'react';
    import {
        SafeAreaView,
        ScrollView,
        View,
        StatusBar,
        Text,
        Button,
        TextInput,
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
            (msg) => this.log('from node-js: ' + msg),
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
                <View>
                    <TextInput
                    placeholder="Type message here!"
                    onChangeText={(text: string) =>
                        this.setState({ ...this.state, message: text })
                    }
                    value={this.state.message}
                    />
                    <Button
                    title="send!"
                    onPress={() =>
                        nodejs.channel.send(this.state.message)
                    }
                    />
                </View>
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
* Now you should be able to send messages to the IoT Hub using the text field / button on the UI. Cloud-to-Device messages should be displayed in the log.
* I have borrowed heavily from the [azure-iot-sdk sample code](https://github.com/Azure/azure-iot-sdk-node/tree/master/device/samples).
* All code can be found in this [repo on GitHub](https://github.com/bnek/react-native-and-iot).

## conclusion
It's possible to run node-js alongside React Native applications and therefore make use of the Azure IoT Hub device SDK for Node.js on mobile devices.