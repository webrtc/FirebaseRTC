# Firebase + WebRTC Codelab


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

1) Create and set up a Firebase project

* Create a Firebase project
In the Firebase console, click Add project, then name the Firebase project FirebaseRTC.
Remember the Project ID for your Firebase project.

Note: If you go to the home page of your project, you can see it in Settings > Project Settings (or look at the URL!)
Click Create project.
The application that you're going to build uses two Firebase services available on the web:

Cloud Firestore to save structured data on the Cloud and get instant notification when the data is updated
Firebase Hosting to host and serve your static assets
For this specific codelab, you've already configured Firebase Hosting in the project you'll be cloning. However, for Cloud Firestore, we'll walk you through the configuration and enabling of the services using the Firebase console.

Enable Cloud Firestore
The app uses Cloud Firestore to save the chat messages and receive new chat messages.

You'll need to enable Cloud Firestore:

In the Firebase console menu's Develop section, click Database.
Click Create database in the Cloud Firestore pane.
Select the Start in test mode option, then click Enable after reading the disclaimer about the security rules.
Test mode ensures that you can freely write to the database during development. We'll make our database more secure later on in this codelab.

1) Get the sample code
Clone the GitHub repository from the command line:


git clone https://github.com/webrtc/FirebaseRTC
The sample code should have been cloned into the FirebaseRTC directory. Make sure your command line is run from this directory from now on:


cd FirebaseRTC
Import the starter app
Open the files in FirebaseRTC in your editor and change them according to the instructions below. This directory contains the starting code for the codelab which consists of a not-yet functional WebRTC app. We'll make it functional throughout this codelab.

1) Install the Firebase Command Line Interface
The Firebase Command Line Interface (CLI) allows you to serve your web app locally and deploy your web app to Firebase Hosting.

Note: To install the CLI, you need to install npm which typically comes with Node.js.
Install the CLI by running the following npm command: sh npm -g install firebase-tools
Note: Doesn't work? You may need to run the command using sudo instead.
Verify that the CLI has been installed correctly by running the following command: sh firebase --version
Make sure the version of the Firebase CLI is v6.7.1 or later.

Authorize the Firebase CLI by running the following command: sh firebase login
You've set up the web app template to pull your app's configuration for Firebase Hosting from your app's local directory and files. But to do this, you need to associate your app with your Firebase project.

Associate your app with your Firebase project by running the following command: sh firebase use --add

When prompted, select your Project ID, then give your Firebase project an alias.

An alias is useful if you have multiple environments (production, staging, etc). However, for this codelab, let's just use the alias of default.

Follow the remaining instructions in your command line.

1) Run the local server
You're ready to actually start work on our app! Let's run the app locally!

Run the following Firebase CLI command: sh firebase serve --only hosting

Your command line should display the following response: hosting: Local server: http://localhost:5000

We're using the Firebase Hosting emulator to serve our app locally. The web app should now be available from http://localhost:5000.

Open your app at http://localhost:5000.
You should see your copy of FirebaseRTC which has been connected to your Firebase project.

The app has automatically connected to your Firebase project.

1) Creating a new room
In this application, each video chat session is called a room. A user can create a new room by clicking a button in their application. This will generate an ID that the remote party can use to join the same room. The ID is used as the key in Cloud Firestore for each room.

Each room will contain the RTCSessionDescriptions for both the offer and the answer, as well as two separate collections with ICE candidates from each party.

Your first task is to implement the missing code for creating a new room with the initial offer from the caller. Open public/app.js and find the comment // Add code for creating a room here and add the following code:


const offer = await peerConnection.createOffer();
await peerConnection.setLocalDescription(offer);

const roomWithOffer = {
    offer: {
        type: offer.type,
        sdp: offer.sdp
    }
}
const roomRef = await db.collection('rooms').add(roomWithOffer);
const roomId = roomRef.id;
document.querySelector('#currentRoom').innerText = `Current room is ${roomId} - You are the caller!`
The first line creates an RTCSessionDescription that will represent the offer from the caller. This is then set as the local description, and finally written to the new room object in Cloud Firestore.

Next, we will listen for changes to the database and detect when an answer from the callee has been added.


roomRef.onSnapshot(async snapshot -> {
    console.log('Got updated room:', snapshot.data());
    const data = snapshot.data();
    if (!peerConnection.currentRemoteDescription && data.answer) {
        console.log('Set remote description: ', data.answer);
        const answer = new RTCSessionDescription(data.answer)
        await peerConnection.setRemoteDescription(answer);
    }
});
This will wait until the callee writes the RTCSessionDescription for the answer, and set that as the remote description on the caller RTCPeerConnection.

1) Joining a room
The next step is to implement the logic for joining an existing room. The user does this by clicking the Join room button and entering the ID for the room to join. Your task here is to implement the creation of the RTCSessionDescription for the answer and update the room in the database accordingly.


const offer = roomSnapshot.data().offer;
await peerConnection.setRemoteDescription(offer);
const answer = await peerConnection.createAnswer();
await peerConnection.setLocalDescription(answer);

const roomWithAnswer = {
    answer: {
        type: answer.type,
        sdp: answer.sdp
    }
}
await roomRef.update(roomWithAnswer);
In the code above, we start by extracting the offer from the caller and creating a RTCSessionDescription that we set as the remote description. Next, we create the answer, set it as the local description, and update the database. The update of the database will trigger the onSnapshot callback on the caller side, which in turn will set the remote description based on the answer from the callee. This completes the exchange of the RTCSessionDescription objects between the caller and the callee.

1) Collect ICE candidates
Before the caller and callee can connect to each other, they also need to exchange ICE candidates that tell WebRTC how to connect to the remote peer. Your next task is to implement the code that listens for ICE candidates and adds them to a collection in the database. Find the function collectIceCandidates and add the following code:

```
async function collectIceCandidates(roomRef, peerConnection,
                                    localName, remoteName) {
    const candidatesCollection = roomRef.collection(localName);

    peerConnection.addEventListener('icecandidate', event -> {
        if (event.candidate) {
            const json = event.candidate.toJSON();
            candidatesCollection.add(json);
        }
    });

    roomRef.collection(remoteName).onSnapshot(snapshot -> {
        snapshot.docChanges().forEach(change -> {
            if (change.type === "added") {
                const candidate = new RTCIceCandidate(change.doc.data());
                peerConneciton.addIceCandidate(candidate);
            }
        });
    })
}
```
This function does two things. It collects ICE candidates from the WebRTC API and adds them to the database, and listens for added ICE candidates from the remote peer and adds them to its RTCPeerConnection instance. It is important when listening to database changes to filter out anything that isn't a new addition, since we otherwise would have added the same set of ICE candidates over and over again.

1) Conclusion
In this codelab you learned how to implement signaling for WebRTC using Cloud Firestore, as well as how to use that for creating a simple video chat application.

To learn more, visit the following resources:

FirebaseRTC Source Code
WebRTC samples
Cloud Firestore
