# Firebase + WebRTC Codelab

### Full code solution can be found under the branch: _solution_
  This is the GitHub repo for the FirebaseRTC codelab. This will teach you how to use Firebase Cloud Firestore for signalling in a WebRTC video chat application.
  The solution to this codelab can be seen in the _solution_ branch.

  See http://webrtc.org for details.
  
Note: the following instructions were copied and updated/corrected from https://webrtc.org/getting-started/firebase-rtc-codelab.

## Introduction

In this codelab, you'll learn how to build a simple video chat application using the WebRTC API in your browser and Cloud Firestore for signaling. The application is called FirebaseRTC and works as a simple example that will teach you the basics of building WebRTC enabled applications.

> Note: Another option for signaling could be Firebase Cloud Messaging. However, that is currently only supported in Chrome and in this codelab we will focus on a solution that works across all browsers supporting WebRTC.

### What you'll learn
* Initiating a video call in a web application using WebRTC
* Signaling to the remote party using Cloud Firestore

### What you'll need
Before starting this codelab, make sure that you've installed:
* npm which typically comes with Node.js - Node LTS is recommended

## Instructions

1. __Create and set up a Firebase project__
    1. Create a Firebase project
        - [In the Firebase console](https://console.firebase.google.com/), click __Create a project__, then name the Firebase project "FirebaseRTC".
        - Remember the Project ID for your Firebase project.
            >Note: If you go to the home page of your project, you can see it in Settings > Project Settings (or look at the URL!)
        - Disable Google Analytics
        - Click Create project.
        - Creation takes a moment.  When your project is created, click Continue.

    The application that you're going to build uses two Firebase services available on the web:
    * Cloud Firestore
        - to save structured data on the Cloud and get instant notification when the data is updated
    * Firebase Hosting
        - to host and serve your static assets

    For this specific codelab, you've already configured Firebase Hosting in the project you'll be cloning. However, for Cloud Firestore, we'll walk you through the configuration and enabling of the services using the Firebase console.

1. __Enable Cloud Firestore__
    
    The app uses Cloud Firestore to save the chat messages and receive new chat messages.
    1. In the Firebase sidebar, navigate to Build -> Cloud Firestore.
    1. Click __Create database__ in the Cloud Firestore pane.
    1. Select the __Start in test mode__ option, read the warning about database security, and click __Next__. Then click __Enable__ after reading the warning about the security rules.
    Test mode ensures that you can freely write to the database during development. 

1. __Get the sample code__
    1. On your local machine, clone the codelab GitHub repository from the command line:
        `git clone https://github.com/webrtc/FirebaseRTC` \[__DELETE in PR:__ for now, use `git clone https://github.com/meldaravaniel/FirebaseRTC`\]
        - The sample code should have been cloned into the FirebaseRTC directory.
    1. Make sure your command line is run from this directory from now on by executing 
        `cd FirebaseRTC`

    As you work through the tutorial, open the files in `FirebaseRTC` in your editor and change them according to the instructions below. This directory contains the starting code for the codelab which consists of a not-yet functional WebRTC app. We'll make it functional throughout this codelab.
    
1. __Install the Firebase Command Line Interface__
    
    The Firebase Command Line Interface (CLI) allows you to serve your web app locally and deploy your web app to Firebase Hosting.
    > Note: To install the CLI, you need to install npm which typically comes with Node.js.
    1. Install the CLI by running the following npm command: `npm -g install firebase-tools` 
        - On unix and doesn't work? You may need to run the command using sudo instead.
    1. Verify that the CLI has been installed correctly by running the following command: `firebase --version`
        - Make sure the version of the Firebase CLI is v6.7.1 or later.
    1. Authorize the Firebase CLI by running the following command: `firebase login`
        - if the terminal you use is 'non-interactive' (eg. git-bash) you may need to append `--interactive` to the above command.
    
    You've set up the web app template to pull your app's configuration for Firebase Hosting from your app's local directory and files. But to do this, you need to associate your app with your Firebase project.
    1. Associate your app with your Firebase project by running the following command: `firebase use --add`
        - you may need to append `--interactive` to the above command.
        - When prompted, select your Project ID (from the Create Firestore Project step), then give your Firebase project an alias.  An alias is useful if you have multiple environments (production, staging, etc). However, for this codelab, let's just use the alias of default.
    1. Follow the remaining instructions in your command line.
    
1. __Run the local server__
    
    You're ready to actually start work on our app! Let's run the app locally!
    1. Run the following Firebase CLI command: `firebase serve --only hosting`
        - you may need to append `--interactive` to the above command.
        - Your command line should display the following response: 
            > hosting: Local server: http://localhost:5000
        - We're using the Firebase Hosting emulator to serve our app locally. The web app should now be available from http://localhost:5000.
    1. Open your app at http://localhost:5000
        You should see your copy of FirebaseRTC which has been connected to your Firebase project.
    The app has automatically connected to your Firebase project.
    > Note: if you want to access the server from other devices on your local network, you need to add `-o 0.0.0.0` at the end of the `firebase serve` command.
    
1. __Creating a new room__
    
    In this application, each video chat session is called a __room__. A user can create a new room by clicking a button in their application. This will generate an ID that the remote party can use to join the same room. The ID is used as the key in Cloud Firestore for each room.
    
    Each room will contain the `RTCSessionDescriptions` for both the offer and the answer, as well as two separate collections with [ICE candidates](https://webrtcglossary.com/ice/#:~:text=ICE%20stands%20for%20Interactive%20Connectivity,NAT%20traversal%20used%20in%20WebRTC.) from each party.

    Your first task is to implement the missing code for creating a new room with the initial offer from the caller. Open public/app.js, find the comment `// Add code for creating a room below`, and add the following code. (You should add the code before the corresponding `// Add code for creating a room above`; the same goes for the rest of the code blocks.)

    ```
    const offer = await peerConnection.createOffer();
    await peerConnection.setLocalDescription(offer);
    console.log('Created offer:', offer);
    const roomWithOffer = {
        'offer': {
            type: offer.type,
            sdp: offer.sdp
        }
    }
    await roomRef.set(roomWithOffer);
    roomId = roomRef.id;
    console.log('New room created with SDP offer. Room ID: ${roomRef.id}');
    document.querySelector('#currentRoom').innerText = `Current room is ${roomId} - You are the caller!`
    ```
    The first line creates an `RTCSessionDescription` that will represent the offer from the caller. This is then set as the local description and, finally, written to the new room object in Cloud Firestore.

    Next, we will listen for changes to the database and detect when an answer from the callee has been added.  Find the comment `// Listening for remote session description below` and add the following code:

    ```
    roomRef.onSnapshot(async snapshot => {
    const data = snapshot.data();
    if (!peerConnection.currentRemoteDescription && data && data.answer) {
      console.log('Got remote description: ', data.answer);
      const rtcSessionDescription = new RTCSessionDescription(data.answer);
      await peerConnection.setRemoteDescription(rtcSessionDescription);
    }
    });
    ```
    This will wait until the callee writes the `RTCSessionDescription` for the answer, and set that as the remote description on the caller `RTCPeerConnection`.

1. __Joining a room__
    
    The next step is to implement the logic for joining an existing room. The user does this by clicking the "Join room" button and entering the ID for the room to join. Your task here is to implement the creation of the `RTCSessionDescription` for the answer and update the room in the database accordingly.  Find the comment `// Code for creating SDP answer below` in the `joinRoomById(roomId)` method and add the following code:

    ```
    const offer = roomSnapshot.data().offer;
    console.log('Got offer:', offer);
    await peerConnection.setRemoteDescription(new RTCSessionDescription(offer));
    const answer = await peerConnection.createAnswer();
    console.log('Created answer:', answer);
    await peerConnection.setLocalDescription(answer);

    const roomWithAnswer = {
      answer: {
        type: answer.type,
        sdp: answer.sdp,
      },
    };
    await roomRef.update(roomWithAnswer);
    ```
    In the code above, we start by extracting the offer from the caller and creating a `RTCSessionDescription` that we set as the remote description. Next, we create the answer, set it as the local description, and update the database. The update of the database will trigger the `onSnapshot` callback that we added in the previous step on the caller side. That callback will set the remote description based on the answer from the callee. This completes the exchange of the `RTCSessionDescription` objects between the caller and the callee.

1. __Collect ICE candidates__
    
    Before the caller and callee can connect to each other, they also need to exchange ICE candidates that tell WebRTC how to connect to the remote peer. Your next task is to implement the code that listens for ICE candidates and adds them to a collection in the database. Find the comment `// collect ICE Candidates function below` and add the following function:

    ```
    async function collectIceCandidates(roomRef, peerConnection,
                                        localName, remoteName) {
        const candidatesCollection = roomRef.collection(localName);

        peerConnection.addEventListener('icecandidate', event => {
            if (event.candidate) {
                const json = event.candidate.toJSON();
                candidatesCollection.add(json);
            }
        });

        roomRef.collection(remoteName).onSnapshot(snapshot => {
            snapshot.docChanges().forEach(change => {
                if (change.type === "added") {
                    const candidate = new RTCIceCandidate(change.doc.data());
                    peerConnection.addIceCandidate(candidate);
                }
            });
        })
    }
    ```
    This function does two main things. 
      * collects ICE candidates from the WebRTC API and adds them to the database
      * listens for added ICE candidates from the remote peer and adds them to its `RTCPeerConnection` instance. 
    
    It is important when listening to database changes to filter out anything that isn't a new addition, since we otherwise would have added the same set of ICE candidates over and over again.
    
    Complete this step by uncommenting the calls to this function in both the `joinRoomById` and `createRoom` methods; you can find them after `// Uncomment to collect ICE candidates below`.

1. __Try it out__

    1. Open a new browswer tab at http://localhost:5000, or refresh the tab you opened above.
    1. Mute your speakers to avoid loud piercing feedback!
    1. Click on __Open camera & microphone__; give permission to the app to use them, if requested.
    1. Click on __Create room__. The app will display the room ID.
    1. Open another browswer tab, click __Open camera & microphone__, and click __Join room__. Paste in the ID from the previous step. The two instances should connect.
    
    To connect with a different device on your local network:
    
    1. Make sure that your start the server with `-o 0.0.0.0`. 
    1. Find the IP address of your server. Let's say it's `192.168.1.5`.
    1. On your remote device, you need to enable the camera for "insecure orgins," since you are connecting to the server via http, not https. For Chrome, you can do this by navigating to `chrome://flags/#unsafely-treat-insecure-origin-as-secure`, and adding `http://192.168.1.5:5000/` to the list of origins.
    1. Connect as above, except that you will have to enter the room ID manually.


1. __Conclusion__
    
    In this codelab you learned how to implement signaling for WebRTC using Cloud Firestore, as well as how to use that for creating a simple video chat application.

    To learn more, visit the following resources:

    * [FirebaseRTC Source Code](https://github.com/webrtc/FirebaseRTC)
    * [WebRTC samples](https://webrtc.github.io/samples)
    * [Cloud Firestore](https://firebase.google.com/docs/firestore/)
