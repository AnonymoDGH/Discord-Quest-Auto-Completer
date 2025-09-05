# Discord Quest Auto-Completer

Instantly completes Discord quests with GUI interface.

## Features

- Auto-completes all quest types (Video, Desktop, Stream, Activity)
- Draggable GUI with real-time logs
- Works in browser and desktop app
- Auto-refresh quest detection

## Installation

1. Open Discord in browser/desktop
2. Open Developer Console (F12)
3. Paste script and press Enter

## Usage

- GUI appears top-right after injection
- Click "Complete Quest" to auto-complete active quest
- Drag panel to reposition
- View logs for status updates

## Quest Types Supported

- **WATCH_VIDEO**: Instant completion
- **PLAY_ON_DESKTOP**: Game spoofing + heartbeat
- **STREAM_ON_DESKTOP**: Stream metadata spoofing
- **PLAY_ACTIVITY**: Voice channel activity simulation

## Script

```javascript
(function() {
    let gui = document.createElement('div');
    gui.id = 'quest-gui';
    gui.innerHTML = `
        <div style="position:fixed;top:20px;right:20px;width:300px;background:#2c2f33;color:#fff;border-radius:8px;padding:15px;z-index:9999;font-family:monospace;font-size:12px;box-shadow:0 4px 12px rgba(0,0,0,0.5);cursor:move;" id="quest-panel">
            <div style="display:flex;justify-content:space-between;margin-bottom:10px;">
                <span>Discord Quest Bot</span>
                <span style="cursor:pointer;color:#ff4444;" onclick="document.getElementById('quest-gui').remove()">×</span>
            </div>
            <div id="quest-status">Loading...</div>
            <button id="complete-quest" style="width:100%;margin:10px 0;padding:8px;background:#5865f2;color:#fff;border:none;border-radius:4px;cursor:pointer;" onclick="executeQuest()">Complete Quest</button>
            <div id="quest-log" style="height:150px;overflow-y:scroll;background:#1e2124;padding:8px;border-radius:4px;font-size:11px;"></div>
        </div>
    `;
    document.body.appendChild(gui);

    let panel = document.getElementById('quest-panel');
    let isDragging = false, startX, startY, initialLeft, initialTop;

    panel.addEventListener('mousedown', (e) => {
        if(e.target.tagName === 'BUTTON' || e.target.innerHTML === '×') return;
        isDragging = true;
        startX = e.clientX;
        startY = e.clientY;
        initialLeft = parseInt(window.getComputedStyle(panel).left, 10);
        initialTop = parseInt(window.getComputedStyle(panel).top, 10);
    });

    document.addEventListener('mousemove', (e) => {
        if(!isDragging) return;
        panel.style.left = (initialLeft + e.clientX - startX) + 'px';
        panel.style.top = (initialTop + e.clientY - startY) + 'px';
    });

    document.addEventListener('mouseup', () => isDragging = false);

    function log(msg) {
        let logEl = document.getElementById('quest-log');
        logEl.innerHTML += `<div>${new Date().toLocaleTimeString()}: ${msg}</div>`;
        logEl.scrollTop = logEl.scrollHeight;
    }

    function checkQuests() {
        try {
            delete window.$;
            let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
            webpackChunkdiscord_app.pop();

            let stores = {};
            for(let m of Object.values(wpRequire.c)) {
                if(m?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata) stores.streaming = m.exports.Z;
                if(m?.exports?.ZP?.getRunningGames) stores.games = m.exports.ZP;
                if(m?.exports?.Z?.__proto__?.getQuest) stores.quests = m.exports.Z;
                if(m?.exports?.Z?.__proto__?.getAllThreadsForParent) stores.channels = m.exports.Z;
                if(m?.exports?.ZP?.getSFWDefaultChannel) stores.guildChannels = m.exports.ZP;
                if(m?.exports?.Z?.__proto__?.flushWaitQueue) stores.dispatcher = m.exports.Z;
                if(m?.exports?.tn?.get) stores.api = m.exports.tn;
            }

            window.questStores = stores;
            let quest = [...stores.quests.quests.values()].find(x => 
                x.id !== "1248385850622869556" && 
                x.userStatus?.enrolledAt && 
                !x.userStatus?.completedAt && 
                new Date(x.config.expiresAt).getTime() > Date.now()
            );

            if(!quest) {
                document.getElementById('quest-status').innerHTML = '<span style="color:#ff4444;">No active quests</span>';
            } else {
                let taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
                let taskName = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"].find(x => taskConfig.tasks[x]);
                document.getElementById('quest-status').innerHTML = `<span style="color:#57f287;">${quest.config.messages.questName}</span><br>Type: ${taskName}`;
                window.currentQuest = quest;
            }
        } catch(e) {
            log(`Error: ${e.message}`);
        }
    }

    window.executeQuest = function() {
        if(!window.currentQuest) return log('No quest found');
        
        let quest = window.currentQuest;
        let stores = window.questStores;
        let taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
        let taskName = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"].find(x => taskConfig.tasks[x]);
        let secondsNeeded = taskConfig.tasks[taskName].target;

        log(`Starting ${quest.config.messages.questName}...`);

        if(taskName.includes("WATCH_VIDEO")) {
            stores.api.post({
                url: `/quests/${quest.id}/video-progress`, 
                body: {timestamp: secondsNeeded}
            }).then(() => {
                log('Quest completed!');
                setTimeout(checkQuests, 1000);
            });
        }
        else if(taskName === "PLAY_ON_DESKTOP") {
            const pid = Math.floor(Math.random() * 30000) + 1000;
            
            const fakeGame = {
                cmdLine: `C:\\Program Files\\${quest.config.application.name}\\${quest.config.application.name}.exe`,
                exeName: `${quest.config.application.name}.exe`,
                exePath: `c:/program files/${quest.config.application.name.toLowerCase()}/${quest.config.application.name}.exe`,
                hidden: false,
                isLauncher: false,
                id: quest.config.application.id,
                name: quest.config.application.name,
                pid: pid,
                pidPath: [pid],
                processName: quest.config.application.name,
                start: Date.now() - (secondsNeeded * 1000),
            };
            
            const realGetGames = stores.games.getRunningGames;
            const realGetPID = stores.games.getGameForPID;
            
            stores.games.getRunningGames = () => [fakeGame];
            stores.games.getGameForPID = () => fakeGame;
            
            let heartbeatCheck = (data) => {
                if(data.quest?.id === quest.id && data.userStatus?.completedAt) {
                    log('Quest completed!');
                    stores.games.getRunningGames = realGetGames;
                    stores.games.getGameForPID = realGetPID;
                    stores.dispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", heartbeatCheck);
                    setTimeout(checkQuests, 1000);
                }
            };
            stores.dispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", heartbeatCheck);
            
            log('Game spoofed, waiting for completion');
        }
        else if(taskName === "STREAM_ON_DESKTOP") {
            stores.streaming.getStreamerActiveStreamMetadata = () => ({
                id: quest.config.application.id,
                pid: 12345,
                sourceName: null
            });
            
            setTimeout(() => {
                stores.api.post({url: `/quests/${quest.id}/heartbeat`, body: {}});
                log('Quest completed!');
                setTimeout(checkQuests, 1000);
            }, 2000);
        }
        else if(taskName === "PLAY_ACTIVITY") {
            let channelId = stores.channels.getSortedPrivateChannels()[0]?.id ?? 
                Object.values(stores.guildChannels.getAllGuilds()).find(x => x?.VOCAL?.length > 0)?.VOCAL[0]?.channel?.id;
            
            if(channelId) {
                stores.api.post({
                    url: `/quests/${quest.id}/heartbeat`, 
                    body: {stream_key: `call:${channelId}:1`, terminal: true}
                }).then(() => {
                    log('Quest completed!');
                    setTimeout(checkQuests, 1000);
                });
            }
        }
    };

    checkQuests();
    setInterval(checkQuests, 30000);
    log('Quest bot initialized');
})();
```
