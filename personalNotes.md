Following tutorial: https://webrtc.org/getting-started/firebase-rtc-codelab

Notes:
* `sh npm -g install firebase-tools` doesn't work, omit `sh`, works.
* `firebase login` gives error: 'cannot run login from non-interactive mode.  See login:ci to generate a token for use in non-interactive environments
	* Am I supposed to be running these commands from within a terminal in Firebase?  Cuz if so, could NOT find a terminal if one exists...
	* ah...cool, becauase I'm using git-bash...it's 'non-interactive'. https://github.com/firebase/firebase-tools/issues/149 so we can force that with `--interactive`.
* Okay so just add `--interactive` after every command, and don't ever use `sh` if you're using git-bash on windows...
* Regarding the tutorial about Firebase: there's no longer a "developer" pane or what have you, you instead have to go to Build -> Cloud Firestore to do the create database step.

* okkkaaayyy so that tutorial is gloriously half-baked.  In comparing the 'solution' code to the starter code and the instructions in the tutorial...there's no way anyone following the tutorial will make it functional without just copying the solution branch app.js into their own in frustration.  Ugh.  Let's.....start this over, shall we?
