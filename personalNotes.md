Following tutorial: https://webrtc.org/getting-started/firebase-rtc-codelab

Notes:
* `sh npm -g install firebase-tools` doesn't work, omit `sh`, works.
* `firebase login` gives error: 'cannot run login from non-interactive mode.  See login:ci to generate a token for use in non-interactive environments
	* Am I supposed to be running these commands from within a terminal in Firebase?  Cuz if so, could NOT find a terminal if one exists...
	* ah...cool, becauase I'm using git-bash...it's 'non-interactive'. https://github.com/firebase/firebase-tools/issues/149 so we can force that with `--interactive`.
