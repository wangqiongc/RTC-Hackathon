<script>
  import { onMount, onDestroy } from "svelte";
  import { Replayer, EventType, pack, mirror } from "rrweb";
  import { createMachine, interpret } from "@xstate/fsm";
  import { quintOut } from "svelte/easing";
  import { scale } from "svelte/transition";
  import { RtcTransporter } from "./transport";
  import { BUFFER_MS, MirrorBuffer } from "./buffer";
  import {
    APP_UID,
    CUSTOM_EVENT_TAGS,
    REMOTE_CONTROL_ACTIONS,
  } from "./constant";
  import { formatBytes, onMirror, isIgnoredOnRmoteControl } from "./common.js";
  import Panel from "./components/Panel.svelte";
  import LineChart from "./components/LineChart.svelte";
  import Icon from "./components/Icon.svelte";

  window.mirror = mirror;

  const transporter = new RtcTransporter(APP_UID);
  let login = transporter.login();

  let playerDom;
  let replayer;
  const buffer = new MirrorBuffer({
    onRecord({ chunk }) {
      if (
        !controlCurrent.matches("controlling") ||
        !isIgnoredOnRmoteControl(chunk)
      ) {
        replayer.addEvent(chunk);
      }
    },
  });

  let open = false;

  const appMachine = createMachine(
    {
      initial: "idle",
      states: {
        idle: {
          on: {
            SOURCE_READY: {
              target: "sourceReady",
            },
          },
        },
        sourceReady: {
          on: {
            CONNECT: {
              target: "waiting_first_record",
            },
          },
        },
        waiting_first_record: {
          on: {
            FIRST_RECORD: {
              target: "connected",
            },
          },
        },
        connected: {
          on: {
            STOP: {
              target: "stopped",
              actions: ["stop"],
            },
          },
        },
        stopped: {
          on: {
            RESET: "idle",
          },
        },
      },
    },
    {
      actions: {
        stop() {
          replayer.pause();
          playerDom.innerHTML = "";
          buffer.reset();
          latencies = []
          sizes = []
        },
      },
    }
  );

  let stopControl;
  const controlMachine = createMachine(
    {
      initial: "not_control",
      states: {
        not_control: {
          on: {
            REQUEST: {
              target: "requested",
              actions: ["request"],
            },
          },
        },
        requested: {
          on: {
            ACCEPTED: {
              target: "controlling",
              actions: ["accepted"],
            },
          },
        },
        controlling: {
          on: {
            STOP_CONTROL: {
              target: "not_control",
              actions: ["stopControl"],
            },
          },
        },
      },
    },
    {
      actions: {
        request() {
          transporter.sendRemoteControl({
            action: REMOTE_CONTROL_ACTIONS.REQUEST,
          });
        },
        accepted() {
          replayer.enableInteract();
          stopControl = onMirror(replayer.iframe, (payload) => {
            transporter.sendRemoteControl(payload);
          });
        },
        stopControl() {
          transporter.sendRemoteControl({
            action: REMOTE_CONTROL_ACTIONS.STOP,
          });
          replayer.disableInteract();
          if (stopControl) {
            stopControl();
          }
        },
      },
    }
  );

  function connect() {
    service.send("CONNECT");
    transporter.sendMirrorReady();
    replayer = new Replayer([], {
      root: playerDom,
      loadTimeout: 100,
      liveMode: true,
      insertStyleRules: [".syncit-embed { display: none !important }"],
      showWarning: true,
      showDebug: true,
    });
  }

  let latencies = [];
  let sizes = [];
  function normalizePoints(points) {
    if (points.length > 20) {
      points = points.slice(points.length - 20, points.length);
    } else {
      points = new Array(20 - points.length)
        .fill()
        .map((_, idx) => ({
          x: points[0] ? points[0].x - 1000 * (21 - points.length - idx) : 0,
          y: 0,
        }))
        .concat(points);
    }
    return points;
  }
  $: _latencies = normalizePoints(latencies);
  $: _sizes = normalizePoints(sizes);
  $: {
    if (latencies.length > 20) {
      latencies = latencies.slice(latencies.length - 20, latencies.length);
    }
    if (sizes.length > 20) {
      sizes = sizes.slice(sizes.length - 20, sizes.length);
    }
  }
  $: lastSize = formatBytes(_sizes[_sizes.length - 1].y);
  function getSizeOfString(str) {
    return encodeURI(str).split(/%(?:u[0-9A-F]{2})?[0-9A-F]{2}|./).length - 1;
  }
  function collectSize(timestamp, str) {
    if (sizes.length === 0) {
      sizes.push({ x: Date.now(), y: 0 });
    }
    const lastSize = sizes[sizes.length - 1];
    const size = getSizeOfString(str);
    if (timestamp - lastSize.x < 1000) {
      lastSize.y += size;
    } else {
      sizes.push({ x: Date.now(), y: size });
    }
    sizes = sizes;
  }

  let mouseSize = "syncit-mouse-s2";

  let current = appMachine.initialState;
  const service = interpret(appMachine);

  let controlCurrent = controlMachine.initialState;
  const controlService = interpret(controlMachine);

  onMount(() => {
    service.start();
    service.subscribe((state) => {
      current = state;
    });
    controlService.start();
    controlService.subscribe((state) => {
      controlCurrent = state;
    });

    transporter.on("sourceReady", () => {
      service.send("SOURCE_READY");
    });
    transporter.on("record", (data) => {
      const { id, chunk, t } = data.payload;
      if (!current.matches("connected")) {
        replayer.startLive(chunk.timestamp - BUFFER_MS);
        service.send("FIRST_RECORD");
      }
      if (chunk.type === EventType.Custom) {
        switch (chunk.data.tag) {
          case CUSTOM_EVENT_TAGS.PING:
            latencies = latencies.concat({ x: t, y: Date.now() - t });
            break;
          case CUSTOM_EVENT_TAGS.MOUSE_SIZE:
            mouseSize = `syncit-mouse-s${chunk.data.payload.level}`;
            break;
          case CUSTOM_EVENT_TAGS.ACCEPT_REMOTE_CONTROL:
            controlService.send("ACCEPTED");
            break;
          case CUSTOM_EVENT_TAGS.STOP_REMOTE_CONTROL:
            controlService.send("STOP_CONTROL");
            break;
          default:
            break;
        }
      }
      Promise.resolve().then(() => collectSize(t, pack(chunk)));
      buffer.add({ id, chunk });
      transporter.ackRecord(id);
    });
    transporter.on("stop", () => {
      service.send("STOP");
    });
  });
  onDestroy(() => {
    service.stop();
    controlService.stop();
  });
</script>

<div class="syncit-app {mouseSize}">
  {#await login}
  <div class="syncit-load-text syncit-center">初始化中...</div>
  {:then}
  <!---->
  <div bind:this="{playerDom}"></div>
  {#if current.matches('idle')}
  <div class="syncit-center">
    <div class="syncit-load-text syncit-hint">
      <h3>等待连接中</h3>
      <div>源端点击“启用 syncit 分享”按钮，即可建立连接。</div>
    </div>
  </div>
  {:else if current.matches('sourceReady') ||
  current.matches('waiting_first_record')}
  <div class="syncit-center">
    <button
      class="syncit-btn"
      on:click="{connect}"
      disabled="{!current.matches('sourceReady')}"
    >
      建立连接
    </button>
  </div>
  {:else if current.matches('connected')}
  <div class="syncit-app-control">
    {#if open}
    <div
      transition:scale="{{duration: 500, opacity: 0.5, easing: quintOut}}"
      style="transform-origin: right bottom;"
    >
      <Panel>
        <div class="syncit-metric">
          <div class="syncit-chart-title">
            时延
            <span style="color: #41efc5;">
              {_latencies.length ? _latencies[_latencies.length - 1].y : '-'} ms
            </span>
          </div>
          <div class="syncit-metric-line">
            <LineChart points="{_latencies}"></LineChart>
          </div>
        </div>
        <div class="syncit-metric">
          <div class="syncit-chart-title">
            流量
            <span style="color: #8c83ed;">
              {lastSize.value} {lastSize.unit}
            </span>
          </div>
          <div class="syncit-metric-line">
            <LineChart points="{_sizes}" color="#8C83ED"></LineChart>
          </div>
        </div>
        <div>
          <p>远程控制</p>
          {#if controlCurrent.matches('not_control')}
          <button
            class="syncit-btn ordinary"
            on:click="{() => controlService.send('REQUEST')}"
          >
            申请控制
          </button>
          {:else if controlCurrent.matches('requested')}
          <button class="syncit-btn ordinary" disabled>
            已申请
          </button>
          {:else if controlCurrent.matches('controlling')}
          <button
            class="syncit-btn ordinary"
            on:click="{() => controlService.send('STOP_CONTROL')}"
          >
            停止控制
          </button>
          {/if}
        </div>
      </Panel>
    </div>
    {/if}
    <!---->
    <button class="syncit-toggle syncit-btn" on:click="{() => open = !open}">
      <Icon name={open ? 'close' : 'team'}></Icon>
    </button>
  </div>
  {:else if current.matches('stopped')}
  <div class="syncit-center">
    <div class="syncit-load-text syncit-hint">
      <div>连接已被断开</div>
      <button
        class="syncit-btn"
        on:click="{() => service.send('RESET')}"
        style="display: block; margin: 0.5em auto;"
      >
        重新准备
      </button>
    </div>
  </div>
  {/if}
  <!---->
  {:catch error}
  <div class="syncit-error syncit-center">{error.message}</div>
  {/await}
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
  }
  :global(iframe) {
    border: none;
  }
  :global(p) {
    margin-top: 0;
    margin-bottom: 8px;
  }
  :global(.replayer-wrapper) {
    position: relative;
  }
  :global(.replayer-mouse) {
    position: absolute;
    width: 20px;
    height: 20px;
    transition: 0.05s linear;
    background-size: contain;
    background-position: center center;
    background-repeat: no-repeat;
    background-image: url("data:image/svg+xml;base64,PHN2ZyBoZWlnaHQ9JzMwMHB4JyB3aWR0aD0nMzAwcHgnICBmaWxsPSIjMDAwMDAwIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIGRhdGEtbmFtZT0iTGF5ZXIgMSIgdmlld0JveD0iMCAwIDUwIDUwIiB4PSIwcHgiIHk9IjBweCI+PHRpdGxlPkRlc2lnbl90bnA8L3RpdGxlPjxwYXRoIGQ9Ik00OC43MSw0Mi45MUwzNC4wOCwyOC4yOSw0NC4zMywxOEExLDEsMCwwLDAsNDQsMTYuMzlMMi4zNSwxLjA2QTEsMSwwLDAsMCwxLjA2LDIuMzVMMTYuMzksNDRhMSwxLDAsMCwwLDEuNjUuMzZMMjguMjksMzQuMDgsNDIuOTEsNDguNzFhMSwxLDAsMCwwLDEuNDEsMGw0LjM4LTQuMzhBMSwxLDAsMCwwLDQ4LjcxLDQyLjkxWm0tNS4wOSwzLjY3TDI5LDMyYTEsMSwwLDAsMC0xLjQxLDBsLTkuODUsOS44NUwzLjY5LDMuNjlsMzguMTIsMTRMMzIsMjcuNThBMSwxLDAsMCwwLDMyLDI5TDQ2LjU5LDQzLjYyWiI+PC9wYXRoPjwvc3ZnPg==");
  }
  .syncit-mouse-s1 :global(.replayer-mouse) {
    width: 10px;
    height: 10px;
  }
  .syncit-mouse-s2 :global(.replayer-mouse) {
    width: 20px;
    height: 20px;
  }
  .syncit-mouse-s3 :global(.replayer-mouse) {
    width: 30px;
    height: 30px;
  }
  :global(.replayer-mouse::after) {
    content: "";
    display: inline-block;
    width: 20px;
    height: 20px;
    border-radius: 10px;
    background: #e75a3a;
    transform: translate(-10px, -10px);
    opacity: 0.3;
  }
  :global(.replayer-mouse.active::after) {
    animation: click 0.2s ease-in-out 1;
  }

  @keyframes click {
    0% {
      opacity: 0.3;
      width: 20px;
      height: 20px;
      border-radius: 10px;
      transform: translate(-10px, -10px);
    }
    50% {
      opacity: 0.5;
      width: 10px;
      height: 10px;
      border-radius: 5px;
      transform: translate(-5px, -5px);
    }
  }

  .syncit-app {
    width: 100%;
    height: 100%;
  }

  button {
    outline: none;
  }
  .syncit-btn:hover {
    background: #3399ff;
  }

  .syncit-btn,
  .syncit-btn:active {
    cursor: pointer;
    background: #0078f0;
    border: 1px solid rgba(62, 70, 82, 0.18);
    box-shadow: 0px 1px 2px rgba(184, 192, 204, 0.6);
    color: #fff;
    padding: 8px 16px;
    border-radius: 4px;
    font-size: 14px;
    line-height: 22px;
    margin-bottom: 0.5em;
  }

  .syncit-btn.ordinary {
    background: #fff;
    color: #3e4652;
    border: 1px solid rgba(129, 138, 153, 0.6);
  }
  .syncit-btn.ordinary:hover {
    background: #f5f7fa;
  }
  .syncit-btn.ordinary:active {
    background: #dfe4eb;
  }

  .syncit-center {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
  }

  .syncit-load-text {
    font-size: 14px;
    line-height: 22px;
    color: #3e4652;
  }

  .syncit-load-text h3 {
    margin: 8px 0;
  }

  .syncit-error {
    color: #e75a3a;
  }

  .syncit-hint {
    background: rgba(245, 247, 250);
    border-radius: 4px;
    padding: 8px;
    min-width: 150px;
  }

  .syncit-toggle,
  .syncit-toggle:active {
    width: 40px;
    height: 40px;
    line-height: 40px;
    border-radius: 20px;
    padding: 0;
    align-self: flex-end;
  }

  .syncit-app-control {
    position: absolute;
    right: 1em;
    bottom: 1em;
    display: flex;
    flex-direction: column;
  }

  .syncit-metric {
    display: flex;
    align-items: center;
    margin-bottom: 8px;
  }

  .syncit-chart-title {
    font-size: 13px;
    line-height: 20px;
    color: #3e4652;
    width: 100px;
    margin-right: 8px;
  }

  .syncit-metric-line {
    flex: 1;
  }
</style>
