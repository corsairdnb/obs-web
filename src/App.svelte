<script>
  /* eslint-env browser */
  const OBS_WEBSOCKET_LATEST_VERSION = '5.0.1' // https://api.github.com/repos/Palakis/obs-websocket/releases/latest

  // Imports
  import { onMount } from 'svelte'
  import {
    mdiSquareRoundedBadge,
    mdiSquareRoundedBadgeOutline,
    mdiImageEdit,
    mdiImageEditOutline,
    mdiFullscreen,
    mdiFullscreenExit,
    mdiBorderVertical,
    mdiArrowSplitHorizontal,
    mdiAccessPoint,
    mdiAccessPointOff,
    mdiRecord,
    mdiStop,
    mdiPause,
    mdiPlayPause,
    mdiConnection,
    mdiCameraOff,
    mdiCamera,
    mdiMotionPlayOutline,
    mdiMotionPlay,
    mdiContentSaveMoveOutline,
    mdiContentSaveCheckOutline
  } from '@mdi/js'
  import Icon from 'mdi-svelte'
  import { compareVersions } from 'compare-versions'

  import './style.scss'
  import { obs, sendCommand } from './obs.js'
  import ProgramPreview from './ProgramPreview.svelte'
  import SceneSwitcher from './SceneSwitcher.svelte'
  import SourceSwitcher from './SourceSwitcher.svelte'
  import ProfileSelect from './ProfileSelect.svelte'
  import SceneCollectionSelect from './SceneCollectionSelect.svelte'
  import {WebMidi} from 'webmidi';

  onMount(async () => {
    // if ('serviceWorker' in navigator) {
    //   navigator.serviceWorker.register('/service-worker.js')
    // }

    // Request screen wakelock
    if ('wakeLock' in navigator) {
      try {
        await navigator.wakeLock.request('screen')
        // Re-request when coming back
        document.addEventListener('visibilitychange', async () => {
          if (document.visibilityState === 'visible') {
            await navigator.wakeLock.request('screen')
          }
        })
      } catch (e) {}
    }

    // Toggle the navigation hamburger menu on mobile
    const navbar = document.querySelector('.navbar-burger')
    navbar.addEventListener('click', () => {
      navbar.classList.toggle('is-active')
      document
        .getElementById(navbar.dataset.target)
        .classList.toggle('is-active')
    })

    // Listen for fullscreen changes
    document.addEventListener('fullscreenchange', () => {
      isFullScreen = document.fullscreenElement
    })

    document.addEventListener('webkitfullscreenchange', () => {
      isFullScreen = document.webkitFullscreenElement
    })

    document.addEventListener('msfullscreenchange', () => {
      isFullScreen = document.msFullscreenElement
    })

    if (document.location.hash !== '') {
      // Read address from hash
      address = document.location.hash.slice(1)

      // This allows you to add a password in the URL like this:
      // http://obs-web.niek.tv/#ws://localhost:4455#password
      if (address.includes('#')) {
        [address, password] = address.split('#')
      }
      await connect()
    }

    await connect() // TODO: called only when no password

    // Export the sendCommand() function to the window object
    window.sendCommand = sendCommand
  })

  // State
  let connected
  let heartbeat = {}
  let heartbeatInterval
  let isFullScreen
  let isStudioMode
  let isSceneOnTop = window.localStorage.getItem('isSceneOnTop') || false
  let isVirtualCamActive
  let isIconMode = window.localStorage.getItem('isIconMode') || false
  let isReplaying
  let editable = false
  let address
  let password
  let scenes = []
  let replayError = ''
  let errorMessage = ''
  let imageFormat = 'jpg'
  let isSaveReplay = false
  let isSaveReplayDisabled = false
  const sceneName = 'table';

  let timerDisabled = true

  $: isSceneOnTop
    ? window.localStorage.setItem('isSceneOnTop', 'true')
    : window.localStorage.removeItem('isSceneOnTop')

  $: isIconMode
    ? window.localStorage.setItem('isIconMode', 'true')
    : window.localStorage.removeItem('isIconMode')

  function formatTime (secs) {
    secs = Math.round(secs / 1000)
    const hours = Math.floor(secs / 3600)
    secs -= hours * 3600
    const mins = Math.floor(secs / 60)
    secs -= mins * 60
    return hours > 0
      ? `${hours}:${mins < 10 ? '0' : ''}${mins}:${secs < 10 ? '0' : ''}${secs}`
      : `${mins < 10 ? '0' : ''}${mins}:${secs < 10 ? '0' : ''}${secs}`
  }

  function toggleFullScreen () {
    if (isFullScreen) {
      if (document.exitFullscreen) {
        document.exitFullscreen()
      } else if (document.webkitExitFullscreen) {
        document.webkitExitFullscreen()
      } else if (document.msExitFullscreen) {
        document.msExitFullscreen()
      }
    } else {
      if (document.documentElement.requestFullscreen) {
        document.documentElement.requestFullscreen()
      } else if (document.documentElement.webkitRequestFullscreen) {
        document.documentElement.webkitRequestFullscreen()
      } else if (document.documentElement.msRequestFullscreen) {
        document.documentElement.msRequestFullscreen()
      }
    }
  }

  const cam1 = 'cam1'
  const cam2 = 'cam2'

  function isCameraSourceName(name) {
    return [cam1, cam2].includes(name)
  }

  async function toggleCamera() {
    const data = await sendCommand('GetSceneItemList', {sceneName, sceneItemId: 2})
    // console.log(data.sceneItems)
    const enabledId = data.sceneItems.find(item => item.sceneItemEnabled === true && isCameraSourceName(item.sourceName)).sceneItemId
    const disabledId = data.sceneItems.find(item => item.sceneItemEnabled === false && isCameraSourceName(item.sourceName)).sceneItemId
    await sendCommand('SetSceneItemEnabled', {sceneName, sceneItemId: disabledId, sceneItemEnabled: true})
    await sendCommand('SetSceneItemEnabled', {sceneName, sceneItemId: enabledId, sceneItemEnabled: false})
  }

  async function setCamera(index) {
    const firstCam = index === 0;
    const secondCam = index === 1;
    const data = await sendCommand('GetSceneItemList', {sceneName, sceneItemId: 2})
    const camera1 = data.sceneItems.find(item => item.sceneItemIndex === 0 && isCameraSourceName(item.sourceName)).sceneItemId
    const camera2 = data.sceneItems.find(item => item.sceneItemIndex === 1 && isCameraSourceName(item.sourceName)).sceneItemId
    await sendCommand('SetSceneItemEnabled', {sceneName, sceneItemId: camera1, sceneItemEnabled: firstCam})
    await sendCommand('SetSceneItemEnabled', {sceneName, sceneItemId: camera2, sceneItemEnabled: secondCam})
  }

  async function toggleStudioMode () {
    await sendCommand('SetStudioModeEnabled', {
      studioModeEnabled: !isStudioMode
    })
  }

  function setReplayError (message) {
    replayError = message
    setTimeout(() => {
      replayError = ''
    }, 5000)
  }

  async function toggleReplay () {
    const data = await sendCommand('ToggleReplayBuffer')
    console.debug('ToggleReplayBuffer', data.outputActive)
    if (data.outputActive === undefined) {
      setReplayError('Replay buffer is not enabled.')
    } else isReplaying = data.outputActive
  }

  async function saveReplay () {
    const data = await sendCommand('GetReplayBufferStatus')
    console.debug('GetReplayBufferStatus', data.outputActive)
    if (!data.outputActive) {
      setReplayError('Replay buffer is not enabled.')
      return
    }
    await sendCommand('SaveReplayBuffer')
    isSaveReplayDisabled = true
    isSaveReplay = true
    setTimeout(() => {
      isSaveReplay = false
      isSaveReplayDisabled = false
    }, 2500)
  }

  async function switchSceneView () {
    isSceneOnTop = !isSceneOnTop
  }

  async function startStream () {
    await sendCommand('StartStream')
  }

  async function stopStream () {
    await sendCommand('StopStream')
  }

  async function startRecording () {
    await sendCommand('StartRecord')
  }

  async function stopRecording () {
    await sendCommand('StopRecord')
  }

  async function startVirtualCam () {
    await sendCommand('StartVirtualCam')
  }

  async function stopVirtualCam () {
    await sendCommand('StopVirtualCam')
  }

  async function pauseRecording () {
    await sendCommand('PauseRecord')
  }

  async function resumeRecording () {
    await sendCommand('ResumeRecord')
  }

  async function connect () {
    address = address || 'ws://localhost:4455'
    if (address.indexOf('://') === -1) {
      const secure = location.protocol === 'https:' || address.endsWith(':443')
      address = secure ? 'wss://' : 'ws://' + address
    }
    console.log('Connecting to:', address, '- using password:', password)
    await disconnect()
    try {
      const { obsWebSocketVersion, negotiatedRpcVersion } = await obs.connect(
        address,
        password
      )
      console.log(
        `Connected to obs-websocket version ${obsWebSocketVersion} (using RPC ${negotiatedRpcVersion})`
      )
    } catch (e) {
      console.log(e)
      errorMessage = e.message
    }
  }

  async function disconnect () {
    await obs.disconnect()
    clearInterval(heartbeatInterval)
    connected = false
    errorMessage = 'Disconnected'
  }

  // OBS events
  obs.on('ConnectionClosed', () => {
    connected = false
    window.history.pushState(
      '',
      document.title,
      window.location.pathname + window.location.search
    ) // Remove the hash
    console.log('Connection closed')
  })

  obs.on('Identified', async () => {
    console.log('Connected')
    connected = true
    document.location.hash = address // For easy bookmarking
    const data = await sendCommand('GetVersion')
    const version = data.obsWebSocketVersion || ''
    console.log('OBS-websocket version:', version)
    if (compareVersions(version, OBS_WEBSOCKET_LATEST_VERSION) < 0) {
      alert(
        'You are running an outdated OBS-websocket (version ' +
          version +
          '), please upgrade to the latest version for full compatibility.'
      )
    }
    if (
      data.supportedImageFormats.includes('webp') &&
      document
        .createElement('canvas')
        .toDataURL('image/webp')
        .indexOf('data:image/webp') === 0
    ) {
      imageFormat = 'webp'
    }
    heartbeatInterval = setInterval(async () => {
      const stats = await sendCommand('GetStats')
      const streaming = await sendCommand('GetStreamStatus')
      const recording = await sendCommand('GetRecordStatus')
      heartbeat = { stats, streaming, recording }
      // console.log(heartbeat);
    }, 300) // Heartbeat
    isStudioMode =
      (await sendCommand('GetStudioModeEnabled')).studioModeEnabled || false
    isVirtualCamActive =
      (await sendCommand('GetVirtualCamStatus')).outputActive || false
  })

  obs.on('ConnectionError', async () => {
    errorMessage = 'Please enter your password:'
    document.getElementById('password').focus()
    if (!password) {
      connected = false
    } else {
      await connect()
    }
  })

  obs.on('VirtualcamStateChanged', async (data) => {
    console.log('VirtualcamStateChanged', data.outputActive)
    isVirtualCamActive = data && data.outputActive
  })

  obs.on('StudioModeStateChanged', async (data) => {
    console.log('StudioModeStateChanged', data.studioModeEnabled)
    isStudioMode = data && data.studioModeEnabled
  })

  obs.on('ReplayBufferStateChanged', async (data) => {
    console.log('ReplayBufferStateChanged', data)
    isReplaying = data && data.outputActive
  })


  WebMidi
    .enable()
    .then(() => listenMidi())
    .catch(err => console.log(err));

  function listenMidi() {
    const controllerName = "Akai MPD32";
    const input = WebMidi.getInputByName(controllerName);
    if (input) {
      input.addListener("noteon", (e) => {
        console.log(e.note.identifier);
        switch (e.note.identifier) {
          case 'C2': // pad 1
            setCamera(0);
            break;
          case 'C#2': // pad 2
            setCamera(1);
            break;
          case 'D2': // pad 3
            setCamera(2);
            break;
          case 'D#2': // pad 4
            toggleCamera();
            break;
        }
      })
    } else {
      console.log("NO MIDI CONTROLLER");
      alert("Не обнаружен MIDI-контроллер. Попробуйте переподключить или закрыть/открыть Reaper.")
    }
  }

  let midi = null; // global MIDIAccess object

  function onMIDISuccess(midiAccess) {
    console.log(midiAccess);
    console.log("MIDI ready!");
    midi = midiAccess; // store in the global (in real usage, would probably keep in an object instance)
    // startLoggingMIDIInput(midiAccess);

    for (const entry of midiAccess.inputs) {
      const input = entry[1];
      console.log(input)
      console.log(
        `Input port [type:'${input.type}']` +
        ` id:'${input.id}'` +
        ` manufacturer:'${input.manufacturer}'` +
        ` name:'${input.name}'` +
        ` version:'${input.version}'`,
      );
    }
  }

  function onMIDIFailure(msg) {
    console.error(`Failed to get MIDI access - ${msg}`);
  }

  navigator.requestMIDIAccess().then(onMIDISuccess, onMIDIFailure);

  // function onMIDIMessage(event) {
  //   // console.log(event)
  //   let str = `MIDI message received at timestamp ${event.timeStamp}[${event.data.length} bytes]: `;
  //   for (const character of event.data) {
  //     str += `0x${character.toString(16)} `;
  //   }
  //   // console.log(str);
  // }
  //
  // function startLoggingMIDIInput(midiAccess) {
  //   midiAccess.inputs.forEach((entry) => {
  //     entry.onmidimessage = onMIDIMessage;
  //   });
  // }

  function handleKeyPress(event) {
    const key = Number(event.key);

    if (document.querySelector('#timer-value:focus-within')) {
      return
    }

    if (key >= 0 && key <= 9) {
      switch (key) {
        case 1:
          setCamera(0);
          document.querySelector('#pad1').focus()
          break;
        case 2:
          setCamera(1);
          document.querySelector('#pad2').focus()
          break;
        // case 3:
        //   setCamera(2);
        //   break;
        case 4:
          document.querySelector('#pad4').focus()
          toggleCamera()
      }
    }
  }

  document.addEventListener('keydown', handleKeyPress);

  const toggleTimerDisabled = (event) => {
    timerDisabled = !event.target.checked
    if (timerDisabled) {
      toggleTimer(false)
    } else {
      toggleTimer(true)
    }
  }

  let timerInterval = null

  const toggleTimer = (active = true) => {
    const input = document.querySelector('#timer-value')
    let value = Math.round(input.value || 0)
    if (value <= 1) {
      value = 1
    } else if (value > 9999) {
      value = 9999
    }
    input.value = value
    const enabled = active && value && value > 0
    if (enabled) {
      clearInterval(timerInterval)
      timerInterval = setInterval(() => {
        toggleCamera()
        console.log(1)
      }, value * 1000)
    } else {
      clearInterval(timerInterval)
    }
  }

</script>

<svelte:head>
  <title>OBS-web remote control</title>
</svelte:head>

<nav class="navbar is-primary" aria-label="main navigation" style="display: none;">
  <div class="navbar-brand">
    <a class="navbar-item is-size-4 has-text-weight-bold" href="/">
      <img src="favicon.png" alt="OBS-web" class="rotate" /></a
    >

    <!-- svelte-ignore a11y-missing-attribute -->
    <button
      class="navbar-burger burger"
      aria-label="menu"
      aria-expanded="false"
      data-target="navmenu"
    >
      <span aria-hidden="true" />
      <span aria-hidden="true" />
      <span aria-hidden="true" />
    </button>
  </div>

  <div id="navmenu" class="navbar-menu">
    <div class="navbar-end">
      <div class="navbar-item">
        <div class="buttons">
          <!-- svelte-ignore a11y-missing-attribute -->
          {#if connected}
            <button class="button is-info is-light" disabled>
              {#if heartbeat && heartbeat.stats}
                {Math.round(heartbeat.stats.activeFps)} fps, {Math.round(
                  heartbeat.stats.cpuUsage
                )}% CPU, {heartbeat.stats.renderSkippedFrames} skipped frames
              {:else}Connected{/if}
            </button>
            {#if heartbeat && heartbeat.streaming && heartbeat.streaming.outputActive}
              <button
                class="button is-danger"
                on:click={stopStream}
                title="Stop Stream"
              >
                <span class="icon"><Icon path={mdiAccessPointOff} /></span>
                <span>{formatTime(heartbeat.streaming.outputDuration)}</span>
              </button>
            {:else}
              <button
                class="button is-danger is-light"
                on:click={startStream}
                title="Start Stream"
              >
                <span class="icon"><Icon path={mdiAccessPoint} /></span>
              </button>
            {/if}
            {#if heartbeat && heartbeat.recording && heartbeat.recording.outputActive}
              {#if heartbeat.recording.outputPaused}
                <button
                  class="button is-danger"
                  on:click={resumeRecording}
                  title="Resume Recording"
                >
                  <span class="icon"><Icon path={mdiPlayPause} /></span>
                </button>
              {:else}
                <button
                  class="button is-success"
                  on:click={pauseRecording}
                  title="Pause Recording"
                >
                  <span class="icon"><Icon path={mdiPause} /></span>
                </button>
              {/if}
              <button
                class="button is-danger"
                on:click={stopRecording}
                title="Stop Recording"
              >
                <span class="icon"><Icon path={mdiStop} /></span>
                <span>{formatTime(heartbeat.recording.outputDuration)}</span>
              </button>
            {:else}
              <button
                class="button is-danger is-light"
                on:click={startRecording}
                title="Start Recording"
              >
                <span class="icon"><Icon path={mdiRecord} /></span>
              </button>
            {/if}
            {#if isVirtualCamActive}
              <button
                class="button is-danger"
                on:click={stopVirtualCam}
                title="Stop Virtual Webcam"
              >
                <span class="icon"><Icon path={mdiCameraOff} /></span>
              </button>
            {:else}
              <button
                class="button is-danger is-light"
                on:click={startVirtualCam}
                title="Start Virtual Webcam"
              >
                <span class="icon"><Icon path={mdiCamera} /></span>
              </button>
            {/if}
            <button
              class:is-light={!isStudioMode}
              class="button is-link"
              on:click={toggleStudioMode}
              title="Toggle Studio Mode"
            >
              <span class="icon"><Icon path={mdiBorderVertical} /></span>
            </button>
            <button
              class:is-light={!isSceneOnTop}
              class="button is-link"
              on:click={switchSceneView}
              title="Show Scene on Top"
            >
              <span class="icon"><Icon path={mdiArrowSplitHorizontal} /></span>
            </button>
            <button
              class:is-light={!editable}
              class="button is-link"
              title="Edit Scenes"
              on:click={() => (editable = !editable)}
            >
              <span class="icon">
                <Icon path={editable ? mdiImageEditOutline : mdiImageEdit} />
              </span>
            </button>
            <button
              class:is-light={!isIconMode}
              class="button is-link"
              title="Show Scenes as Icons"
              on:click={() => (isIconMode = !isIconMode)}
            >
              <span class="icon">
                <Icon
                  path={isIconMode
                    ? mdiSquareRoundedBadgeOutline
                    : mdiSquareRoundedBadge}
                />
              </span>
            </button>
            <button
              class:is-light={!isReplaying}
              class:is-danger={replayError}
              class="button is-link"
              title="Toggle Replay Buffer"
              on:click={toggleReplay}
            >
              <span class="icon">
                <Icon
                  path={isReplaying ? mdiMotionPlayOutline : mdiMotionPlay}
                />
              </span>
              {#if replayError}<span>{replayError}</span>{/if}
            </button>
            <button
              class:is-light={!isSaveReplay}
              class="button is-link"
              title="Save Replay Buffer"
              on:click={() => {
                if (!isSaveReplayDisabled) {
                  saveReplay()
                }
                isSaveReplayDisabled = !isSaveReplayDisabled
              }}
            >
              <span class="icon">
                <Icon
                  path={isSaveReplay ? mdiContentSaveCheckOutline : mdiContentSaveMoveOutline}
                />
              </span>
              {#if replayError}<span>{replayError}</span>{/if}
            </button>
            <ProfileSelect />
            <SceneCollectionSelect />
            <button
              class="button is-danger is-light"
              on:click={disconnect}
              title="Disconnect"
            >
              <span class="icon"><Icon path={mdiConnection} /></span>
            </button>
          {:else}
            <button class="button is-danger" disabled
              >{errorMessage || 'Disconnected'}</button
            >
          {/if}
          <!-- svelte-ignore a11y-missing-attribute -->
          <button
            class:is-light={!isFullScreen}
            class="button is-link"
            on:click={toggleFullScreen}
            title="Toggle Fullscreen"
          >
            <span class="icon">
              <Icon path={isFullScreen ? mdiFullscreenExit : mdiFullscreen} />
            </span>
          </button>
        </div>
      </div>
    </div>
  </div>
</nav>

<section class="section">
  <div>

    {#if connected}
      <div class="controls">
        <label for="timer-checkbox">
          <input type="checkbox" id="timer-checkbox" on:click={toggleTimerDisabled}>
        </label>
        <label for="timer-value" id="timer-label">
          Автопереключение, сек<br />
          <input type="number" id="timer-value" disabled={timerDisabled} max="9999" min="1" step="1" value="10" on:change={toggleTimer}>
        </label>
      </div>
      <div class="pads" id="pads">
        <button on:click={() => setCamera(0)} class="pad pad--cam1" id="pad1">1</button>
        <button on:click={() => setCamera(1)} class="pad pad--cam2" id="pad2">2</button>
        <button class="pad pad--cam3">3</button>
        <button on:click={toggleCamera} class="pad pad--toggle" id="pad4">1 ⇆ 2</button>
      </div>
    {/if}

    {#if connected}
      <!--{#if isSceneOnTop}-->
      <!--  <ProgramPreview {imageFormat} />-->
      <!--{/if}-->
<!--      <SceneSwitcher-->
<!--        bind:scenes-->
<!--        buttonStyle={isIconMode ? 'icon' : 'text'}-->
<!--        {editable}-->
<!--      />-->
<!--      {#if !isSceneOnTop}-->
<!--        <ProgramPreview {imageFormat} />-->
<!--      {/if}-->
      {#each scenes as scene}
        {#if scene.sceneName.indexOf('(switch)') > 0}
          <SourceSwitcher
            name={scene.sceneName}
            {imageFormat}
            buttonStyle="screenshot"
          />
        {/if}
      {/each}
    {:else}
      <h1 class="subtitle">
        Welcome to
        <strong>OBS-web</strong>
        - the easiest way to control
        <a href="https://obsproject.com/" target="_blank" rel="noreferrer"
          >OBS</a
        >
        remotely!
      </h1>

      {#if document.location.protocol === 'https:'}
        <div class="notification is-danger">
          You are checking this page on a secure HTTPS connection. That's great,
          but it means you can
          <strong>only</strong>
          connect to WSS (secure websocket) addresses, for example OBS exposed with
          <a href="https://ngrok.com/">ngrok</a>
          or
          <a href="https://pagekite.net/">pagekite</a>
          . If you want to connect to a local OBS instance,
          <strong>
            <a
              href="http://{document.location.hostname}{document.location.port
                ? ':' + document.location.port
                : ''}{document.location.pathname}"
            >
              please click here to load the non-secure version of this page
            </a>
          </strong>
          .
        </div>
      {/if}

      <p>To get started, enter your OBS host:port below and click "connect".</p>

      <form on:submit|preventDefault={connect}>
        <div class="field is-grouped">
          <p class="control is-expanded">
            <input
              id="host"
              bind:value={address}
              class="input"
              type="text"
              autocomplete=""
              placeholder="ws://localhost:4455"
            />
            <input
              id="password"
              bind:value={password}
              class="input"
              type="password"
              autocomplete="current-password"
              placeholder="password (leave empty if you have disabled authentication)"
            />
          </p>
          <p class="control">
            <button class="button is-success">Connect</button>
          </p>
        </div>
      </form>
      <p class="help">
        Make sure that you use <a
          href="https://github.com/obsproject/obs-studio/releases">OBS v28+</a
        >
        or install the
        <a
          href="https://github.com/obsproject/obs-websocket/releases/tag/{OBS_WEBSOCKET_LATEST_VERSION}"
          target="_blank"
          rel="noreferrer"
          >obs-websocket {OBS_WEBSOCKET_LATEST_VERSION} plugin</a
        >
        for v27. If you use an older version of OBS, see the
        <a href="/v4/">archived OBS-web v4</a> page.
      </p>
    {/if}
  </div>
</section>

<!--<footer class="footer">-->
<!--  <div class="content has-text-centered">-->
<!--    <p>-->
<!--      <strong>OBS-web</strong>-->
<!--      by-->
<!--      <a href="https://niekvandermaas.nl/">Niek van der Maas</a>-->
<!--      &mdash; see-->
<!--      <a href="https://github.com/Niek/obs-web">GitHub</a>-->
<!--      for source code.-->
<!--    </p>-->
<!--  </div>-->
<!--</footer>-->
