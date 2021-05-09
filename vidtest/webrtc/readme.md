#created by stanley joining various webrtc implementations 


Live Test-Adresss: http://csc.rf.gd/vdtest/webrtc
----

1. Mesh networking model is implemented to open multiple interconnected peer connections.
2. Maximum peer connections limit is 256 (on chrome). It means that 256 users can be interconnected!

# using video-conferencing in your webpage?

```html
<script src="https://cdn.webrtc-experiment.com/socket.io.js"> </script>
<script src="https://webrtc.github.io/adapter/adapter-latest.js"></script>
<script src="https://cdn.webrtc-experiment.com/IceServersHandler.js"></script>
<script src="https://cdn.webrtc-experiment.com/CodecsHandler.js"></script>
<script src="https://cdn.webrtc-experiment.com/video-conferencing/RTCPeerConnection-v1.5.js"> </script>
<script src="https://cdn.webrtc-experiment.com/video-conferencing/conference.js"> </script>

<button id="setup-new-room">Setup New Conference</button>
<table style="width: 100%;" id="rooms-list"></table>
<div id="videos-container"></div>

<script>
var config = {
    openSocket: function (config) {
        var SIGNALING_SERVER = 'https://socketio-over-nodejs2.herokuapp.com:443/',
            defaultChannel = location.href.replace(/\/|:|#|%|\.|\[|\]/g, '');

        var channel = config.channel || defaultChannel;
        var sender = Math.round(Math.random() * 999999999) + 999999999;

        io.connect(SIGNALING_SERVER).emit('new-channel', {
            channel: channel,
            sender: sender
        });

        var socket = io.connect(SIGNALING_SERVER + channel);
        socket.channel = channel;
        socket.on('connect', function () {
            if (config.callback) config.callback(socket);
        });

        socket.send = function (message) {
            socket.emit('message', {
                sender: sender,
                data: message
            });
        };

        socket.on('message', config.onmessage);
    },
    onRemoteStream: function (media) {
        var video = media.video;
        video.setAttribute('id', media.stream.id);
        videosContainer.appendChild(video);
    },
    onRemoteStreamEnded: function (stream) {
        var video = document.getElementById(stream.id);
        if (video) video.parentNode.removeChild(video);
    },
    onRoomFound: function (room) {
        var alreadyExist = document.querySelector('button[data-broadcaster="' + room.broadcaster + '"]');
        if (alreadyExist) return;

        var tr = document.createElement('tr');
        tr.innerHTML = '<td><strong>' + room.roomName + '</strong> shared a conferencing room with you!</td>' +
            '<td><button class="join">Join</button></td>';
        roomsList.appendChild(tr);

        var joinRoomButton = tr.querySelector('.join');
        joinRoomButton.setAttribute('data-broadcaster', room.broadcaster);
        joinRoomButton.setAttribute('data-roomToken', room.broadcaster);
        joinRoomButton.onclick = function () {
            this.disabled = true;

            var broadcaster = this.getAttribute('data-broadcaster');
            var roomToken = this.getAttribute('data-roomToken');
            captureUserMedia(function () {
                conferenceUI.joinRoom({
                    roomToken: roomToken,
                    joinUser: broadcaster
                });
            });
        };
    }
};

var conferenceUI = conference(config);
var videosContainer = document.getElementById('videos-container') || document.body;
var roomsList = document.getElementById('rooms-list');

document.getElementById('setup-new-room').onclick = function () {
    this.disabled = true;
    captureUserMedia(function () {
        conferenceUI.createRoom({
            roomName: 'Anonymous'
        });
    });
};

function captureUserMedia(success_callback, failure_callback) {
    var video = document.createElement('video');
    video.muted = true;
    video.volume = 0;

    video.setAttributeNode(document.createAttribute('autoplay'));
    video.setAttributeNode(document.createAttribute('playsinline'));
    video.setAttributeNode(document.createAttribute('controls'));

    getUserMedia({
        video: video,
        onsuccess: function (stream) {
            config.attachStream = stream;
            videosContainer.appendChild(video);
            success_callback();
        }
    });
}
</script>
```



# Want to use [Firebase](https://www.firebase.com/) for signaling?

```javascript
var config = {
    openSocket: function (config) {
        var channel = config.channel || location.href.replace(/\/|:|#|%|\.|\[|\]/g, '');
        var socket = new Firebase('https://chat.firebaseIO.com/' + channel);
        socket.channel = channel;
        socket.on('child_added', function (data) {
            config.onmessage(data.val());
        });
        socket.send = function (data) {
            this.push(data);
        }
        config.onopen && setTimeout(config.onopen, 1);
        socket.onDisconnect().remove();
        return socket;
    }
}
```

# Want to use [PubNub](http://www.pubnub.com/) for signaling?

```javascript
var config = {
    openSocket: function (config) {
        var channel = config.channel || location.href.replace(/\/|:|#|%|\.|\[|\]/g, '');
        var socket = io.connect('https://pubsub.pubnub.com/' + channel, {
            publish_key: 'demo',
            subscribe_key: 'demo',
            channel: config.channel || channel,
            ssl: true
        });
        if (config.onopen) socket.on('connect', config.onopen);
        socket.on('message', config.onmessage);
        return socket;
    }
}
```

# API

### `conference` function

```javascript
var conf = conference(options);
```

### `createRoom` method

```javascript
var conf = conference(options);
conf.createRoom({
    roomName: 'roomid'
});
```

### `joinRoom` method

```javascript
var conf = conference({
    onRoomFound: function(room) {
        conf.joinRoom({
            roomToken: room.roomToken,
            joinUser: room.broadcaster
        });
    }
});
```

### `conference` function's options

```javascript
var conf = conference({
    attachStream: MediaStream,
    openSocket: function(config) {},
    onRemoteStream: function(event) {},
    onRoomFound: function(room) {},
    onRoomClosed: function(room) {},
    onReady: function() {}
});
```

### `attachStream` option

```javascript
navigator.mediaDevices.getUserMedia({ video: true }).then(function(camera) {
    var conf = conference({
        attachStream: camera
    });
});
```

### `onRemoteStream` event

```javascript
var conf = conference({
    onRemoteStream: function(event) {
        document.body.appendChild(event.media);
    }
});
```

### `onRoomFound` event

```javascript
var join_only_one_room = true;
var conf = conference({
    onRoomFound: function(room) {
        if(!join_only_one_room) return;
        join_only_one_room = false;
        
        conf.joinRoom({
            roomToken: room.roomToken,
            joinUser: room.broadcaster
        });
    }
});
```

### `openSocket` option

```javascript
var conf = conference({
    openSocket: function(config) {
        var SIGNALING_SERVER = 'https://socketio-over-nodejs2.herokuapp.com:443/',
            defaultChannel = location.href.replace(/\/|:|#|%|\.|\[|\]/g, '');

        var channel = config.channel || defaultChannel;
        var sender = Math.round(Math.random() * 999999999) + 999999999;

        io.connect(SIGNALING_SERVER).emit('new-channel', {
            channel: channel,
            sender: sender
        });

        var socket = io.connect(SIGNALING_SERVER + channel);
        socket.channel = channel;
        socket.on('connect', function() {
            if (config.callback) config.callback(socket);
        });

        socket.send = function(message) {
            socket.emit('message', {
                sender: sender,
                data: message
            });
        };

        socket.on('message', config.onmessage);
    }
});
```

# Browser Support

| ------------- |-------------|
| Firefox
| Google Chrome 
| Android 
| Edge | Version 16 or higher |
| Safari | Version 11 on both MacOSX and iOS |


