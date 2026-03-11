<div align="center">
  <h1>🎮 DiscordQuest</h1>
  <p><em>Automate and complete Discord Quests effortlessly!</em></p>
</div>

---

## ⚠️ Important Note
> **This does not work in the browser for quests that require you to play a game!**  
> Use the **Discord Desktop App** to complete those.

---

## 🚀 How to Use

Follow these simple steps:

1. **Accept** a quest under the **Quests** tab in Discord.
2. Press <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>I</kbd> to open **DevTools**.
3. Go to the **Console** tab.
4. *(Note: If you're unable to paste into the console, you might have to type `allow pasting` and hit Enter first)*
5. **Paste** the following code and hit Enter:

<details>
<summary><b>💻 Click to expand the Script</b></summary>

```javascript
delete window.$;
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getStreamerActiveStreamMetadata).exports.A;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getRunningGames).exports.Ay;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getQuest).exports.A;
let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getAllThreadsForParent).exports.A;
let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getSFWDefaultChannel).exports.Ay;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.h?.__proto__?.flushWaitQueue).exports.h;
let api = Object.values(wpRequire.c).find(x => x?.exports?.Bo?.get).exports.Bo;

const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"]
let quests = [...QuestsStore.quests.values()].filter(x => x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now() && supportedTasks.find(y => Object.keys((x.config.taskConfig ?? x.config.taskConfigV2).tasks).includes(y)))
let isApp = typeof DiscordNative !== "undefined"
if(quests.length === 0) {
	console.log("You don't have any uncompleted quests!")
} else {
	let doJob = function() {
		const quest = quests.pop()
		if(!quest) return

		const pid = Math.floor(Math.random() * 30000) + 1000
		
		const applicationId = quest.config.application.id
		const applicationName = quest.config.application.name
		const questName = quest.config.messages.questName
		const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2
		const taskName = supportedTasks.find(x => taskConfig.tasks[x] != null)
		const secondsNeeded = taskConfig.tasks[taskName].target
		let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0

		if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
			const maxFuture = 10, speed = 7, interval = 1
			const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime()
			let completed = false
			let fn = async () => {			
				while(true) {
					const maxAllowed = Math.floor((Date.now() - enrolledAt)/1000) + maxFuture
					const diff = maxAllowed - secondsDone
					const timestamp = secondsDone + speed
					if(diff >= speed) {
						const res = await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}})
						completed = res.body.completed_at != null
						secondsDone = Math.min(secondsNeeded, timestamp)
					}
					
					if(timestamp >= secondsNeeded) {
						break
					}
					await new Promise(resolve => setTimeout(resolve, interval * 1000))
				}
				if(!completed) {
					await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: secondsNeeded}})
				}
				console.log("Quest completed!")
				doJob()
			}
			fn()
			console.log(`Spoofing video for ${questName}.`)
		} else if(taskName === "PLAY_ON_DESKTOP") {
			if(!isApp) {
				console.log("This no longer works in browser for non-video quests. Use the discord desktop app to complete the", questName, "quest!")
			} else {
				api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
					const appData = res.body[0]
					const exeName = appData.executables?.find(x => x.os === "win32")?.name?.replace(">","") ?? appData.name.replace(/[\/\\:*?"<>|]/g, "")
					
					const fakeGame = {
						cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
						exeName,
						exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
						hidden: false,
						isLauncher: false,
						id: applicationId,
						name: appData.name,
						pid: pid,
						pidPath: [pid],
						processName: appData.name,
						start: Date.now(),
					}
					const realGames = RunningGameStore.getRunningGames()
					const fakeGames = [fakeGame]
					const realGetRunningGames = RunningGameStore.getRunningGames
					const realGetGameForPID = RunningGameStore.getGameForPID
					RunningGameStore.getRunningGames = () => fakeGames
					RunningGameStore.getGameForPID = (pid) => fakeGames.find(x => x.pid === pid)
					FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: fakeGames})
					
					let fn = data => {
						let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)
						console.log(`Quest progress: ${progress}/${secondsNeeded}`)
						
						if(progress >= secondsNeeded) {
							console.log("Quest completed!")
							
							RunningGameStore.getRunningGames = realGetRunningGames
							RunningGameStore.getGameForPID = realGetGameForPID
							FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []})
							FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
							
							doJob()
						}
					}
					FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
					
					console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
				})
			}
		} else if(taskName === "STREAM_ON_DESKTOP") {
			if(!isApp) {
				console.log("This no longer works in browser for non-video quests. Use the discord desktop app to complete the", questName, "quest!")
			} else {
				let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
					id: applicationId,
					pid,
					sourceName: null
				})
				
				let fn = data => {
					let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value)
					console.log(`Quest progress: ${progress}/${secondsNeeded}`)
					
					if(progress >= secondsNeeded) {
						console.log("Quest completed!")
						
						ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
						FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
						
						doJob()
					}
				}
				FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
				
				console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
				console.log("Remember that you need at least 1 other person to be in the vc!")
			}
		} else if(taskName === "PLAY_ACTIVITY") {
			const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id ?? Object.values(GuildChannelStore.getAllGuilds()).find(x => x != null && x.VOCAL.length > 0).VOCAL[0].channel.id
			const streamKey = `call:${channelId}:1`
			
			let fn = async () => {
				console.log("Completing quest", questName, "-", quest.config.messages.questName)
				
				while(true) {
					const res = await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: false}})
					const progress = res.body.progress.PLAY_ACTIVITY.value
					console.log(`Quest progress: ${progress}/${secondsNeeded}`)
					
					await new Promise(resolve => setTimeout(resolve, 20 * 1000))
					
					if(progress >= secondsNeeded) {
						await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: true}})
						break
					}
				}
				
				console.log("Quest completed!")
				doJob()
			}
			fn()
		}
	}
	doJob()
}
```
</details>

---

## 🎯 What to do next?

**Follow the printed instructions in the console depending on what type of quest you have:**
- 🎮 **Play/Watch Quests:** You can just wait and do nothing.
- 📺 **Stream Quests:** Join a Voice Channel (VC) with a friend (or an alt) and stream *any* window.
- ⏳ Wait a bit for it to complete the quest.
- 🎁 **Claim** your reward!
- 📊 Track progress by watching the `Quest progress:` prints in the Console tab, or checking the progress bar in the Discord Quests tab.

---

## ❓ Frequently Asked Questions (FAQ)

<details>
<summary><b>Running the script does nothing besides printing <code>undefined</code>, and makes chat messages not go through</b></summary>
This is a random bug with opening DevTools, where all HTTP requests break for a few minutes. It's not the script's fault. Either wait and try again, or restart Discord and try again.
</details>

<details>
<summary><b>Can I get banned for using this?</b></summary>
There is always a risk, though so far nobody has been banned for this or other similar things like client mods.
</details>

<details>
<summary><b>Ctrl + Shift + I doesn't work</b></summary>
Either download the PTB client, or look for a guide on how to enable DevTools on the stable client.
</details>

<details>
<summary><b>Ctrl + Shift + I takes a screenshot</b></summary>
Disable the keybind in your AMD Radeon app or graphics software.
</details>

<details>
<summary><b>I get a syntax error / unexpected token error</b></summary>
Make sure your browser isn't auto-translating the website before copying the script. Turn off any translator extensions and try again.
</details>

<details>
<summary><b>I'm on Vesktop but it tells me that I'm using a browser</b></summary>
Vesktop is not a true desktop client, it's a fancy browser wrapper. Download the actual official desktop app instead.
</details>

<details>
<summary><b>I get a different error</b></summary>
Make sure you're copy/pasting the script correctly and that you've followed all the steps properly.
</details>

<details>
<summary><b>Can I complete expired quests with this?</b></summary>
No, there is no way to do that.
</details>

<details>
<summary><b>Can you make the script auto accept the quest/reward?</b></summary>
No. Both of those actions may show a captcha, so automating them is not a good idea. Just do the two clicks yourself.
</details>

<details>
<summary><b>Can you make this a Vencord plugin?</b></summary>
No. The script sometimes requires immediate updates for Discord's changes, and Vencord's update cycle and code review would be too slow for that. There are some Vencord forks which have implemented this if you really want one.
</details>

<details>
<summary><b>Can you upload the standalone script to a repo and make this code a one line <code>fetch()</code>?</b></summary>
No. Doing that would put you at risk because anyone with account access could change the underlying code to be malicious at any time, then force-push it away later, and you'd never know.
</details>

---

## 📝 Remarks & License

> 🛑 **Comments on the original source have been disabled!**
> The majority were people posting AI slop edits, "fixed/improved" versions with stolen credits that broke things, outdated versions, or users failing to read instructions. This repository only focuses on retaining a tested and reliable copy of the script.

**If you're experiencing a bug:**
1. Make sure you're running the **latest version** of the script.
2. Read the **FAQ** above.
3. If all else fails, try your luck with **Equicord's Questify plugin** instead.

<p align="center">
  <b>License: <a href="https://choosealicense.com/licenses/gpl-3.0/">GPL-3.0</a></b>
</p>
