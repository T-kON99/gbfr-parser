<script lang="ts" context="module">
  import { afterUpdate, onMount } from "svelte";
  import Swiper from "swiper";
  import { Navigation } from "swiper/modules";

  import ChevronLeft from "svelte-material-icons/ChevronLeft.svelte";
  import ChevronRight from "svelte-material-icons/ChevronRight.svelte";
  import Close from "svelte-material-icons/Close.svelte";

  const debug = false;
</script>

<script lang="ts">
  import Session from "./Session.svelte";
  import { activeSession, mutex, sessions } from "./lib/Stores";
  import {
    createSession,
    formatTime,
    getAction,
    getActor,
    getTarget,
    loadSavedSessions,
    modifyActionDamage,
    removeSession,
    saveSessions
  } from "./lib/Utils";

  let ws: WebSocket;
  let updateActiveSession = false;
  let connected = false;

  let swiperRoot: HTMLElement;
  let prevBtn: HTMLElement;
  let nextBtn: HTMLElement;
  let swiper: Swiper;

  const onDamage = (data: EventData, time: number) => {
    mutex.wrap(async () => {
      if (data.source[3] === -1 || data.damage <= 0) return;

      let session = $sessions[$sessions.length - 1];
      if (!session || session.done) {
        session = createSession(time);
        updateActiveSession = true;
      } else if (session.mutex) {
        await session.mutex.wrap(() => {
          session.last_damage_at = time;
        });
      } else return;

      if (session.mutex && !session.start_damage_at) {
        await session.mutex.wrap(() => {
          session.start_damage_at = time;
        });
      }

      await session.mutex?.wrap(() => {
        session.total_dmg += data.damage;

        const actor = getActor(data.source);
        actor.dmg += data.damage;
        actor.dmgm += data.damage;
        ++actor.hit;

        const target = getTarget(actor, data.target);
        target.dmg += data.damage;
        target.dmgm += data.damage;

        session.actors?.forEach(e => {
          e.percentage = e.dmg / session.total_dmg;
        });

        let [action, defaultAction] = getAction(actor, target, data.flags & (1 << 15) ? -3 : data.action_id);
        action = modifyActionDamage(action, data);
        defaultAction = modifyActionDamage(defaultAction, data);

        if (!session.events) session.events = [];
        session.events.push({
          time: time,
          dmg: data.damage,
          source: data.source,
          target: data.target
        });
      });

      sessions.set($sessions);
      if (updateActiveSession) {
        updateActiveSession = false;
        $activeSession = session;

        if (swiper) {
          window.requestAnimationFrame(() => swiper.update());
          window.requestAnimationFrame(() => swiper.slideTo($sessions.indexOf($activeSession)));
        }
      }
    });
  };

  afterUpdate(() => {
    if (swiperRoot && !swiper) {
      swiper = new Swiper(swiperRoot, {
        slidesPerView: "auto",
        centeredSlides: false,
        allowTouchMove: false,
        freeMode: true,
        navigation: {
          nextEl: ".slider-next",
          prevEl: ".slider-prev"
        },
        modules: [Navigation]
      });

      window.requestAnimationFrame(() => swiper.slideTo($sessions.indexOf($activeSession)));
    }
  });

  onMount(() => {
    const savedSessions = loadSavedSessions();
    if (savedSessions) {
      $sessions = savedSessions;
      $activeSession = $sessions[$sessions.length - 1];
    }

    const init = () => {
      ws = new WebSocket("ws://localhost:24399");
      ws.addEventListener("message", ev => {
        if (debug) {
          console.debug(ev);
        }

        const msg: Message = JSON.parse(ev.data);
        if (!msg.time_ms) {
          msg.time_ms = +new Date();
        }
        switch (msg.type) {
          case "enter_area":
            createSession(msg.time_ms);
            updateActiveSession = true;
            break;
          case "damage":
            onDamage(msg.data, msg.time_ms);
            break;
        }
      });
      ws.addEventListener("open", () => (connected = true));
      ws.addEventListener("close", () => setTimeout(init, 1000));
      ws.addEventListener("error", console.error);
    };

    init();

    const save = () => {
      saveSessions();
      window.setTimeout(save, 3000);
    };
    save();
  });
</script>

<div id="main">
  {#if !connected}
    <div class="alert alert-3-danger">
      <i class="alert-content">Failed to connect to websocket server, check injector status...</i>
    </div>
  {/if}
  {#if $sessions.some(session => session.total_dmg > 0)}
    <header>
      <button class="slider-prev" bind:this={prevBtn}>
        <ChevronLeft size="2.1rem" />
      </button>
      <div class="slider-wrapper">
        <div class="swiper" bind:this={swiperRoot}>
          <div class="swiper-wrapper">
            {#each $sessions as session, idx}
              {#if session.total_dmg > 0}
                <button
                  class={`swiper-slide` + ($activeSession === session ? " active" : "")}
                  type="button"
                  on:click|self={() => ($activeSession = session)}
                >
                  {formatTime(session.start_damage_at, session.last_damage_at)}
                  <!-- svelte-ignore a11y-no-static-element-interactions -->
                  <!-- svelte-ignore a11y-click-events-have-key-events -->
                  <span
                    on:click={() => {
                      removeSession(session);
                      if (swiper) {
                        window.requestAnimationFrame(() => swiper.update());
                        if ($activeSession === session) {
                          window.requestAnimationFrame(() => swiper.slideTo($sessions.indexOf($activeSession)));
                        }
                      }
                    }}
                  >
                    <Close size="1.6rem" />
                  </span>
                </button>
              {/if}
            {/each}
          </div>
        </div>
      </div>
      <button class="slider-next" bind:this={nextBtn}>
        <ChevronRight size="2.1rem" />
      </button>
    </header>
    <main>
      {#each $sessions as session (session.start_at)}
        {#if $activeSession === session}
          {#key session.start_at}
            <Session bind:session />
          {/key}
        {/if}
      {/each}
    </main>
  {:else}
    <i>Waiting for battle events... </i>
  {/if}
</div>
