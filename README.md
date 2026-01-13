<details>
	<summary>Click to expand</summary>
  '''
  delete window.$;

let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

// --- Module caching helper ---
const cachedModules = {};
function getModule(findFn) {
    if (!cachedModules[findFn]) {
        cachedModules[findFn] = Object.values(wpRequire.c).find(findFn)?.exports;
    }
    return cachedModules[findFn];
}

// --- Stores ---
const QuestsStore = getModule(x => x?.Z?.__proto__?.getQuest).Z;
const RunningGameStore = getModule(x => x?.ZP?.getRunningGames).ZP;
const ApplicationStreamingStore = getModule(x => x?.Z?.__proto__?.getStreamerActiveStreamMetadata).Z;
const ChannelStore = getModule(x => x?.Z?.__proto__?.getAllThreadsForParent).Z;
const GuildChannelStore = getModule(x => x?.ZP?.getSFWDefaultChannel).ZP;
const FluxDispatcher = getModule(x => x?.Z?.__proto__?.flushWaitQueue).Z;
const api = getModule(x => x?.tn?.get).tn;

// --- Config ---
const supportedTasks = [
    "WATCH_VIDEO",
    "PLAY_ON_DESKTOP",
    "STREAM_ON_DESKTOP",
    "PLAY_ACTIVITY",
    "WATCH_VIDEO_ON_MOBILE"
];

const isApp = typeof DiscordNative !== "undefined";

// --- Filter active quests ---
const quests = [...QuestsStore.quests.values()].filter(q => 
    q.userStatus?.enrolledAt &&
    !q.userStatus?.completedAt &&
    new Date(q.config.expiresAt).getTime() > Date.now() &&
    supportedTasks.some(task => Object.keys((q.config.taskConfig ?? q.config.taskConfigV2).tasks).includes(task))
);

if (!quests.length) {
    console.log("You don't have any uncompleted quests!");
}

// --- Utility ---
const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));

function getQuestTask(quest) {
    const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
    return supportedTasks.find(t => taskConfig.tasks[t] != null);
}

// --- Main handler ---
async function handleQuest(quest) {
    const taskName = getQuestTask(quest);
    const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
    const secondsNeeded = taskConfig.tasks[taskName].target;
    let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;
    const applicationId = quest.config.application?.id;
    const applicationName = quest.config.application?.name;
    const questName = quest.config.messages?.questName;

    switch (taskName) {
        // --- Video tasks ---
        case "WATCH_VIDEO":
        case "WATCH_VIDEO_ON_MOBILE": {
            const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime();
            const maxFuture = 10, speed = 7, interval = 1000;

            console.log(`Spoofing video for ${questName}.`);
            
            while (secondsDone < secondsNeeded) {
                const maxAllowed = Math.floor((Date.now() - enrolledAt) / 1000) + maxFuture;
                const delta = Math.min(speed, maxAllowed - secondsDone);
                if (delta > 0) {
                    secondsDone = Math.min(secondsDone + delta, secondsNeeded);
                    const res = await api.post({
                        url: `/quests/${quest.id}/video-progress`,
                        body: { timestamp: secondsDone }
                    });
                    if (res.body.completed_at) break;
                }
                await sleep(interval);
            }

            // Ensure complete
            if (secondsDone < secondsNeeded) {
                await api.post({
                    url: `/quests/${quest.id}/video-progress`,
                    body: { timestamp: secondsNeeded }
                });
            }

            console.log("Quest completed!");
            break;
        }

        // --- Desktop game task ---
        case "PLAY_ON_DESKTOP": {
            if (!isApp) {
                console.log(`Use Discord Desktop App to complete the ${questName} quest!`);
                break;
            }

            const res = await api.get({ url: `/applications/public?application_ids=${applicationId}` });
            const appData = res.body[0];
            const exeData = appData.executables.find(x => x.os === "win32");
            const exeName = exeData?.name.replace(">", "");
            const pid = Math.floor(Math.random() * 30000) + 1000;

            const fakeGame = {
                cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
                exeName,
                exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
                hidden: false,
                isLauncher: false,
                id: applicationId,
                name: appData.name,
                pid,
                pidPath: [pid],
                processName: appData.name,
                start: Date.now()
            };

            const realGames = RunningGameStore.getRunningGames();
            const originalGetGames = RunningGameStore.getRunningGames;
            const originalGetGameForPID = RunningGameStore.getGameForPID;

            RunningGameStore.getRunningGames = () => [fakeGame];
            RunningGameStore.getGameForPID = pid => (pid === fakeGame.pid ? fakeGame : null);

            FluxDispatcher.dispatch({ type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: [fakeGame] });

            const progressListener = data => {
                const progress = quest.config.configVersion === 1
                    ? data.userStatus.streamProgressSeconds
                    : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);

                console.log(`Quest progress: ${progress}/${secondsNeeded}`);

                if (progress >= secondsNeeded) {
                    console.log("Quest completed!");
                    RunningGameStore.getRunningGames = originalGetGames;
                    RunningGameStore.getGameForPID = originalGetGameForPID;
                    FluxDispatcher.dispatch({ type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: [] });
                    FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", progressListener);
                }
            };

            FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", progressListener);

            console.log(`Spoofed your game to ${applicationName}. Wait for ~${Math.ceil((secondsNeeded - secondsDone) / 60)} minutes.`);
            break;
        }

        // --- Stream task ---
        case "STREAM_ON_DESKTOP": {
            if (!isApp) {
                console.log(`Use Discord Desktop App to complete the ${questName} quest!`);
                break;
            }

            const pid = Math.floor(Math.random() * 30000) + 1000;
            const originalFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;

            ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({ id: applicationId, pid, sourceName: null });

            const progressListener = data => {
                const progress = quest.config.configVersion === 1
                    ? data.userStatus.streamProgressSeconds
                    : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value);

                console.log(`Quest progress: ${progress}/${secondsNeeded}`);

                if (progress >= secondsNeeded) {
                    console.log("Quest completed!");
                    ApplicationStreamingStore.getStreamerActiveStreamMetadata = originalFunc;
                    FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", progressListener);
                }
            };

            FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", progressListener);
            console.log(`Spoofed your stream to ${applicationName}.`);
            break;
        }

        // --- Play Activity task ---
        case "PLAY_ACTIVITY": {
            const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id
                ?? Object.values(GuildChannelStore.getAllGuilds()).find(g => g?.VOCAL?.length > 0)?.VOCAL[0]?.channel?.id;

            if (!channelId) break;

            const streamKey = `call:${channelId}:1`;
            console.log(`Starting activity quest: ${questName}`);

            while (true) {
                const res = await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, terminal: false } });
                const progress = res.body.progress.PLAY_ACTIVITY.value;
                console.log(`Quest progress: ${progress}/${secondsNeeded}`);
                if (progress >= secondsNeeded) {
                    await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, terminal: true } });
                    break;
                }
                await sleep(20_000);
            }

            console.log("Quest completed!");
            break;
        }
    }
}

// --- Run all quests sequentially ---
(async () => {
    for (const quest of quests) {
        try {
            await handleQuest(quest);
        } catch (err) {
            console.error("Error handling quest:", quest.config.messages?.questName, err);
        }
    }
})();

'''
</details>
