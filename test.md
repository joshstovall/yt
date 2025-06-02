<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body {
      margin: 0;
      background: #000;
    }
    .container {
      position: relative;
      width: 100%;
      height: 0;
      padding-bottom: 56.25%;
    }
    .video {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="video" id="player"></div>
  </div>
  
  <script>
    // Create unique player ID
    const randomPlayerId = 'player' + Date.now();
    document.getElementById('player').id = randomPlayerId;
    
    // Parse URL query parameters to get configuration
    const parsedUrl = new URL(window.location.href);
    const UrlQueryData = parsedUrl.searchParams.get('data');
    
    // Extract configuration - using same variable names as react-native-youtube-iframe
    const {
      end,
      list,
      color,
      start,
      rel_s,
      loop_s,
      listType,
      playerLang,
      playlist,
      videoId_s,
      controls_s,
      cc_lang_pref_s,
      contentScale_s,
      allowWebViewZoom,
      modestbranding_s,
      iv_load_policy,
      preventFullScreen_s,
      showClosedCaptions_s,
      autoplay,
      progressUpdateInterval
    } = UrlQueryData ? JSON.parse(decodeURIComponent(UrlQueryData)) : {};

    function sendMessageToRN(msg) {
      if (window.ReactNativeWebView) {
        window.ReactNativeWebView.postMessage(JSON.stringify(msg));
      }
    }

    // Handle viewport scaling
    let metaString = '';
    if (contentScale_s) {
      metaString += `initial-scale=${contentScale_s}, `;
    }
    if (!allowWebViewZoom) {
      metaString += `maximum-scale=${contentScale_s || 1.0}`;
    }
    
    const viewport = document.querySelector('meta[name=viewport]');
    if (viewport && metaString) {
      viewport.setAttribute('content', 'width=device-width, ' + metaString);
    }

    // Load YouTube IFrame API
    var tag = document.createElement('script');
    tag.src = 'https://www.youtube.com/iframe_api';
    var firstScriptTag = document.getElementsByTagName('script')[0];
    firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);

    var player;
    var progressInterval;

    function onYouTubeIframeAPIReady() {
      player = new YT.Player(randomPlayerId, {
        width: '1000',
        height: '1000',
        videoId: videoId_s || 'dQw4w9WgXcQ', // Default to Rick Roll if no video specified
        playerVars: {
          autoplay: autoplay || 0,
          end: end,
          rel: rel_s || 0,
          list: list,
          color: color || 'red',
          loop: loop_s || 0,
          start: start || 0,
          playsinline: 1,
          hl: playerLang || 'en',
          playlist: playlist,
          listType: listType,
          controls: controls_s !== undefined ? controls_s : 1,
          fs: preventFullScreen_s !== undefined ? preventFullScreen_s : 1,
          cc_lang_pref: cc_lang_pref_s,
          iv_load_policy: iv_load_policy !== undefined ? iv_load_policy : 3,
          modestbranding: modestbranding_s !== undefined ? modestbranding_s : 1,
          cc_load_policy: showClosedCaptions_s || 0,
          enablejsapi: 1,
          origin: window.location.origin,
          widget_referrer: window.location.origin,
          host: 'https://www.youtube-nocookie.com' // Use privacy-enhanced mode
        },
        events: {
          onReady: onPlayerReady,
          onError: onPlayerError,
          onStateChange: onPlayerStateChange,
          onPlaybackRateChange: onPlaybackRateChange,
          onPlaybackQualityChange: onPlaybackQualityChange,
        },
      });
    }

    function onPlayerError(event) {
      let errorMessage = '';
      switch (event.data) {
        case 2:
          errorMessage = 'Invalid video ID';
          break;
        case 5:
          errorMessage = 'HTML5 player error';
          break;
        case 100:
          errorMessage = 'Video not found or private';
          break;
        case 101:
          errorMessage = 'Video embedding restricted by owner';
          break;
        case 150:
          errorMessage = 'Video embedding not allowed';
          break;
        default:
          errorMessage = `Unknown error: ${event.data}`;
      }
      
      sendMessageToRN({
        eventType: 'playerError', 
        data: event.data,
        errorMessage: errorMessage
      });
    }

    function onPlaybackRateChange(event) {
      sendMessageToRN({eventType: 'playbackRateChange', data: event.data});
    }

    function onPlaybackQualityChange(event) {
      sendMessageToRN({eventType: 'playerQualityChange', data: event.data});
    }

    function onPlayerReady(event) {
      sendMessageToRN({eventType: 'playerReady'});
      
      // Start progress tracking if needed
      if (progressUpdateInterval) {
        startProgressTracking();
      }
    }

    function onPlayerStateChange(event) {
      sendMessageToRN({eventType: 'playerStateChange', data: event.data});
      
      if (event.data === 1 && progressUpdateInterval) { // playing
        startProgressTracking();
      } else {
        stopProgressTracking();
      }
    }

    function startProgressTracking() {
      stopProgressTracking();
      progressInterval = setInterval(() => {
        if (player && player.getCurrentTime && player.getDuration) {
          try {
            const currentTime = player.getCurrentTime();
            const duration = player.getDuration();
            const percent = duration > 0 ? (currentTime / duration) * 100 : 0;
            
            sendMessageToRN({
              eventType: 'progress',
              data: {
                currentTime: currentTime,
                duration: duration,
                percent: percent
              }
            });
          } catch (error) {
            // Ignore progress tracking errors
          }
        }
      }, progressUpdateInterval || 1000);
    }

    function stopProgressTracking() {
      if (progressInterval) {
        clearInterval(progressInterval);
        progressInterval = null;
      }
    }

    // Handle fullscreen changes
    var isFullScreen = false;
    function onFullScreenChange() {
      isFullScreen =
        document.fullscreenElement ||
        document.msFullscreenElement ||
        document.mozFullScreenElement ||
        document.webkitFullscreenElement;

      sendMessageToRN({
        eventType: 'fullScreenChange',
        data: Boolean(isFullScreen),
      });
    }

    document.addEventListener('fullscreenchange', onFullScreenChange);
    document.addEventListener('msfullscreenchange', onFullScreenChange);
    document.addEventListener('mozfullscreenchange', onFullScreenChange);
    document.addEventListener('webkitfullscreenchange', onFullScreenChange);

    // Control functions
    function sendCommand(command, meta = {}) {
      if (!player) return;

      try {
        switch (command) {
          case 'playVideo':
            player.playVideo();
            break;
          case 'pauseVideo':
            player.pauseVideo();
            break;
          case 'muteVideo':
            player.mute();
            break;
          case 'unMuteVideo':
            player.unMute();
            break;
          case 'setVolume':
            player.setVolume(meta.volume);
            break;
          case 'setPlaybackRate':
            player.setPlaybackRate(meta.playbackRate);
            break;
          case 'seekTo':
            player.seekTo(meta.seconds, meta.allowSeekAhead !== false);
            break;
          case 'loadVideo':
            player.loadVideoById(meta.videoId);
            break;
        }
      } catch (error) {
        // Ignore command errors
      }
    }

    // Handle messages from React Native
    window.addEventListener('message', function (event) {
      try {
        const parsedData = JSON.parse(event.data);
        const {eventName, meta} = parsedData;
        sendCommand(eventName, meta);
      } catch (error) {
        // Ignore message parsing errors
      }
    });

    // Expose functions for React Native
    window.sendCommand = sendCommand;
    window.getCurrentTime = () => player ? player.getCurrentTime() : 0;
    window.getDuration = () => player ? player.getDuration() : 0;
    window.getPlaybackPercent = () => {
      if (!player) return 0;
      const current = player.getCurrentTime();
      const duration = player.getDuration();
      return duration > 0 ? (current / duration) * 100 : 0;
    };
  </script>
</body>
</html>
