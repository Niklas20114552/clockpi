<script>
  import "$lib/app.css";
  import gong from "$lib/sounds/gong.wav";
  import notification from "$lib/sounds/notification.wav";
  import alarm_sound from "$lib/sounds/alarm.wav";
  import { format } from "jsr:@std/datetime";
  import { onMount } from "svelte";
  import { fade } from "svelte/transition";
  import Time from "$lib/Time.svelte";
  import Left from "$lib/Left.svelte";
  import Moon from "$lib/Moon.svelte";
  import Right from "$lib/Right.svelte";

  function getTime() {
    const d = new Date();

    if (d.getMinutes() == 0 && d.getSeconds() == 0 && hour_gong && on) {
      const a = new Audio(gong);
      const max_plays = d.getHours() > 12 ? d.getHours() - 12 : d.getHours();
      for (let i = 0; i < max_plays; i++) {
        setTimeout(() => {
          a.play();
        }, i * 1500);
      }
    }

    if (d.getSeconds() == 0) {
      if (format(d, "HH:mm") == nightmode_on && auto_nightmode) {
        on = false;
      }
      if (format(d, "HH:mm") == nightmode_off && auto_nightmode) {
        on = true;
      }

      for (const alarm of alarms) {
        if (format(d, "HH:mm") == alarm) {
          alarm_ringing = true;
          const a = new Audio(alarm_sound);
          if (!on) {
            nightmode_show_clock = true;
          }

          a.addEventListener("ended", () => {
            if (alarm_ringing) {
              a.play();
            } else {
              nightmode_show_clock = false;
            }
          });

          a.play();
        }
      }

      for (const noti of notifications) {
        if (format(d, "HH:mm") == noti) {
          noti_shown = noti;
          const a = new Audio(notification);
          a.play();
        }
      }
    }

    return [format(d, "HH:mm"), format(d, "ss")];
  }

  function toggleSecond() {
    second_settings++;
    if (second_settings == 3) {
      second_settings = 0;
    }
    localStorage.setItem("seconds", second_settings);
  }

  function setSecond(setting) {
    second_settings = setting;
    localStorage.setItem("seconds", second_settings);

    ftsetup_page++;
  }

  function toggleAutoNightmode() {
    auto_nightmode = !auto_nightmode;
    localStorage.setItem("autonightmode", auto_nightmode);
  }

  function toggleHour() {
    hour_gong = !hour_gong;
    localStorage.setItem("hour", hour_gong);
  }

  function getSecondState(setting) {
    if (setting == 0) {
      return "Off";
    }
    if (setting == 1) {
      return "On";
    }
    if (setting == 2) {
      return "On with pulsing";
    }
  }

  function getLocalStorage(id, defaults) {
    if (!localStorage.getItem(id)) {
      localStorage.setItem(id, defaults);
    }
    return localStorage.getItem(id);
  }

  function setTheme(new_theme) {
    theme = new_theme;
    localStorage.setItem("theme", theme);
  }

  function activate() {
    if (alarm_ringing) {
      return;
    }
    nightmode_show_clock = true;

    if (nightmode_clock_timeout) {
      clearTimeout(nightmode_clock_timeout);
    }

    nightmode_clock_timeout = setTimeout(() => {
      nightmode_show_clock = false;
    }, 5000);
  }

  function setAutonightmode(timing) {
    if (popupstate == 1) {
      nightmode_on = timing.join(":");
    } else {
      nightmode_off = timing.join(":");
    }

    localStorage.setItem("nighton", nightmode_on);
    localStorage.setItem("nightoff", nightmode_off);
  }

  function convertToNumber(number) {
    let nmb = number.toString();
    if (nmb.length == 1) {
      nmb = "0" + nmb;
    }
    return nmb;
  }

  function intTimeListToStrList(lst) {
    return convertToNumber(lst[0]) + ":" + convertToNumber(lst[1]);
  }

  function createAlarm(isNoti) {
    const alarm = [Number(alarm_times[0]), Number(alarm_times[1])];
    alarm_response = "";

    if (alarm[0] > 23 || alarm[1] > 59 || alarm[0] < 0 || alarm[1] < 0) {
      return (alarm_response = "This is not a valid time");
    }

    if (isNoti) {
      notifications = [...notifications, intTimeListToStrList(alarm)];
    } else {
      alarms = [...alarms, intTimeListToStrList(alarm)];
    }
    localStorage.setItem("alarms", JSON.stringify(alarms));
    localStorage.setItem("notifications", JSON.stringify(notifications));

    popupstate = 3;
  }

  function toggleAlarmAt(index, disabled, isNoti) {
    let alrm;
    if (isNoti) {
      alrm = notifications[index];
    } else {
      alrm = alarms[index];
    }

    if (disabled) {
      alrm = alrm.substring(1);
    } else {
      alrm = "-" + alrm;
    }

    if (isNoti) {
      notifications[index] = alrm;
    } else {
      alarms[index] = alrm;
    }

    localStorage.setItem("alarms", JSON.stringify(alarms));
    localStorage.setItem("notifications", JSON.stringify(notifications));
  }

  function deleteAlarmAt(index, isNoti) {
    console.log(index);

    if (isNoti) {
      notifications.splice(index, 1);
    } else {
      alarms.splice(index, 1);
    }
    localStorage.setItem("alarms", JSON.stringify(alarms));
    localStorage.setItem("notifications", JSON.stringify(notifications));
  }

  function swipe(end) {
    let distance = swipeXstart - end;
    if (distance > 5 && page_number < 7) {
      page_number++;
    }
    if (distance < -5) {
      if (page_number == 1) {
        popupstate = 3;
      } else {
        page_number--;
      }
    }
  }

  function skip_tour() {
    already_ran = true;
    localStorage.setItem("alreadyran", true);
  }

  function reset() {
    localStorage.clear();
    location.reload();
  }

  let already_ran = $state(getLocalStorage("alreadyran", false) == "true");
  let ftsetup_page = $state(0);

  let theme = $state(getLocalStorage("theme", "lightblue"));
  let second_settings = $state(getLocalStorage("seconds", 1));
  let hour_gong = $state(getLocalStorage("hour", false) == "true");
  let auto_nightmode = $state(
    getLocalStorage("autonightmode", false) == "true",
  );
  let alarms = $state(JSON.parse(getLocalStorage("alarms", "[]")));
  let notifications = $state(
    JSON.parse(getLocalStorage("notifications", "[]")),
  );

  let nightmode_on = getLocalStorage("nighton", "21:00");
  let nightmode_off = getLocalStorage("nightoff", "08:00");

  let [time, seconds] = $state(getTime());
  let page_number = $state(1);
  let on = $state(true);
  let nightmode_show_clock = $state(false);
  let nightmode_clock_timeout;

  let swipeXstart;

  let popupstate = $state(0);

  let splitted_time = $state(); // for setting auto nightmode
  let alarm_times = $state(); // for creating alarms
  let alarm_response = $state("");
  let alarm_ringing = $state(false);

  let noti_shown = $state();

  $effect(() => {
    splitted_time = (popupstate == 1 ? nightmode_on : nightmode_off).split(":");
  });
  $effect(() => {
    setAutonightmode(splitted_time);
  });

  onMount(() => {
    setInterval(() => {
      [time, seconds] = getTime();
    }, 1000);
  });
</script>

<!-- svelte-ignore a11y_no_static_element_interactions -->
<!-- svelte-ignore a11y_click_events_have_key_events -->
<bg
  class={theme}
  class:blackmode={!(nightmode_show_clock || on)}
  class:dimmed={!on}
>
  {#if already_ran}
    {#if on || nightmode_show_clock}
      <div class="centered" in:fade out:fade>
        <h1 class="clock">{time}</h1>
        {#if second_settings != 0}
          {#key seconds}
            <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
            <!-- For debug purposes -->
            <h2
              class="clock"
              class:pulsing={second_settings == 2}
              onclick={() => location.reload()}
            >
              {seconds}
            </h2>
          {/key}
        {/if}

        {#if nightmode_show_clock && !alarm_ringing}
          <button
            type="button"
            onclick={() => {
              on = true;
              nightmode_show_clock = false;
            }}>Turn off Nightmode</button
          >
        {/if}

        {#if alarm_ringing}
          <button
            type="button"
            onclick={() => {
              alarm_ringing = false;
            }}>Dismiss alarm</button
          >
        {/if}
      </div>
    {/if}

    {#if on}
      <div
        class="screenbox"
        onmousedown={(e) => {
          swipeXstart = e.clientX;
        }}
        onmouseup={(e) => {
          swipe(e.clientX);
        }}
      ></div>

      {#if (popupstate > 0 || noti_shown) && !alarm_ringing}
        <div class="popout" transition:fade>
          {#if noti_shown}
            <h1>Notification</h1>
            <h2>It's {noti_shown}!</h2>
          {:else if popupstate == 1 || popupstate == 2}
            <h1>Automatic Nightmode</h1>
            <span>Turning {popupstate == 1 ? "on" : "off"} at</span>
            <Time {splitted_time} />
            <br />
            <span
              >Setting an invalid time will result in nightmode not turning {popupstate ==
              1
                ? "on"
                : "off"}.</span
            >
            <br />
            <button
              type="button"
              onclick={() => {
                popupstate == 1 ? (popupstate = 2) : (popupstate = 1);
              }}>Change {popupstate == 1 ? "off" : "on"}</button
            >
          {:else if popupstate == 3}
            <h1>Alarms</h1>
            {#if alarms.length}
              {#each alarms as alarm, index}
                {@const disabled = alarm.startsWith("-")}
                {@const alrm = disabled ? alarm.replace("-", "") : alarm}
                <span style="font-size: 14pt;"
                  >Alarm at {alrm}
                  {disabled ? "is disabled" : "is active"}</span
                >
                <button
                  type="button"
                  onclick={() => {
                    toggleAlarmAt(index, disabled, false);
                  }}>{disabled ? "Activate" : "Deactivate"}</button
                >
                <button
                  type="button"
                  onclick={() => {
                    deleteAlarmAt(index, false);
                  }}>Delete</button
                >
                <br />
              {/each}
            {:else}
              <span>No alarms created</span>
              <br />
            {/if}
            <hr />
            {#if notifications.length}
              {#each notifications as noti, index}
                {@const disabled = noti.startsWith("-")}
                {@const alrm = disabled ? noti.replace("-", "") : noti}
                <span style="font-size: 14pt;"
                  >Notification at {alrm}
                  {disabled ? "is disabled" : "is active"}</span
                >
                <button
                  type="button"
                  onclick={() => {
                    toggleAlarmAt(index, disabled, true);
                  }}>{disabled ? "Activate" : "Deactivate"}</button
                >
                <button
                  type="button"
                  onclick={() => {
                    deleteAlarmAt(index, true);
                  }}>Delete</button
                >
                <br />
              {/each}
            {:else}
              <span>No notifications created</span>
              <br />
            {/if}

            <button
              type="button"
              onclick={() => {
                alarm_times = ["06", "00"];
                popupstate = 4;
              }}>Create new...</button
            >
          {:else if popupstate == 4}
            <h1>Create alarm or notification</h1>
            <span>Ring at</span>
            <Time splitted_time={alarm_times} />
            <br />
            {alarm_response}
            <br />
            <button type="button" onclick={() => createAlarm(false)}
              >Create alarm</button
            >
            <button type="button" onclick={() => createAlarm(true)}
              >Create notification</button
            >
            <button
              type="button"
              onclick={() => {
                popupstate = 3;
              }}>Back</button
            >
          {/if}
          <button
            type="button"
            onclick={() => {
              popupstate = 0;
              noti_shown = undefined;
            }}>Close</button
          >
        </div>
      {/if}
    {:else}
      <div
        class="screenbox"
        onclick={() => activate()}
        onmousemove={() => activate()}
      ></div>
    {/if}
  {:else}
    <div class="popout trimmed" class:short={ftsetup_page == 5} transition:fade>
      {#if ftsetup_page == 0}
        <h1>Welcome to ClockPi!</h1>
        <p>
          Welcome to your new Smart Clock which also focuses on being
          customizable.
        </p>
        <p>
          During the first time setup the buttom bar serves as the navigation
          bar.<br />In normal use this serves as a place for all kinds of
          settings.
        </p>
      {:else if ftsetup_page == 1}
        <h1>This is a alarm clock. yay!</h1>
        <p>
          You can set alarms and notifications using the <i
            >Manage alarms and notifications</i
          > button in the bottombar.
        </p>
        <p>
          Currently alarms and notifications will activate every day, but you
          can disable them.
        </p>
      {:else if ftsetup_page == 2}
        <h1>But what is that?</h1>

        <h2>Notifications</h2>
        <ul>
          <li>Won't work in Nightmode</li>
          <li>Plays a short sound</li>
          <li>Show a popup</li>
        </ul>

        <h2>Alarms</h2>
        <ul>
          <li>Works in Nightmode</li>
          <li>Plays a continuous sound until dismissed</li>
        </ul>
      {:else if ftsetup_page == 3}
        <h1>What, Nightmode?</h1>
        <p>
          On the last page was the <i>Nightmode</i> mentioned. So let me explain
          what this is.
        </p>
        <p>
          By pressing <Moon /> in the bottom bar you can activate the Nightmode on
          your ClockPi.
        </p>
        <p>
          When the Nightmode is activated your Screen is turned back (off when
          using the official rom).<br />By tapping or swiping on the screen you
          can show the clock for 5 seconds alongside the button to deactive the
          Nightmode.
        </p>
      {:else if ftsetup_page == 4}
        <h1>Cuckoo included!</h1>
        <p>
          The <i>hour gong</i> feature plays a gong sound every full hour.<br
          />That's it.
        </p>
        <p>
          Oh and you're now done with the tour and will continue to the first
          time setup.
        </p>
      {:else if ftsetup_page == 5}
        <h1>Customization</h1>
        <p>ClockPi includes six built-in themes:</p>
        <button type="button" onclick={() => setTheme("blue")}>Cyan</button>
        <button type="button" onclick={() => setTheme("purple")}>Purple</button>
        <button type="button" onclick={() => setTheme("lightblue")}>Blue</button
        >
        <button type="button" onclick={() => setTheme("gray")}>Gray</button>
        <button type="button" onclick={() => setTheme("white")}>White</button>
        <button type="button" onclick={() => setTheme("black")}>Black</button>
        <br />
        <h2>Example clock:</h2>
        <h1 class="clock">12:34</h1>
        <h2 class="clock">45</h2>
      {:else if ftsetup_page == 6}
        <h1>Seconds or no seconds</h1>
        <p>
          You can turn on or off the second subtitle under the clock or have it
          pulse every second.
        </p>
      {:else if ftsetup_page == 7}
        <h1>Almost done! Just a few gestures</h1>
        <p>
          The last thing:<br />You can swipe in the normal mode to the left or
          right to navigate the bottom bar.
        </p>
        <p>
          You can also swipe to the left on the first page to quickly edit
          alarms.
        </p>
      {/if}
    </div>
  {/if}

  {#if on}
    <div class="bottombar" class:justifycenter={!already_ran} in:fade>
      {#if already_ran}
        <div class="bottomitem">
          {#if page_number > 1}
            <Left
              onclick={() => {
                page_number--;
              }}
            />
          {:else}
            <div class="spacebox"></div>
          {/if}
          <Moon
            onclick={() => {
              on = false;
            }}
          />
        </div>
        {#key page_number}
          <div class="bottomitem" in:fade>
            {#if page_number == 1}
              <p>Music sync: Work in progress!</p>
            {:else if page_number == 2}
              <button
                type="button"
                onclick={() => {
                  popupstate = 3;
                }}>Manage alarms and notifications</button
              >
            {:else if page_number == 3}
              <p>Hour gong:&nbsp;</p>
              <button type="button" onclick={() => toggleHour()}
                >{hour_gong ? "On" : "Off"}</button
              >
            {:else if page_number == 4}
              <p>Automatic Nightmode:&nbsp;</p>
              <button type="button" onclick={() => toggleAutoNightmode()}
                >{auto_nightmode ? "On" : "Off"}</button
              >
              <button
                type="button"
                onclick={() => {
                  popupstate = 1;
                }}
                disabled={!auto_nightmode}>Change timings</button
              >
            {:else if page_number == 5}
              <p>Seconds:&nbsp;</p>
              <button type="button" onclick={() => toggleSecond()}
                >{getSecondState(second_settings)}</button
              >
            {:else if page_number == 6}
              <p style="font-size: 16pt;">Set theme:&nbsp;</p>
              <button type="button" onclick={() => setTheme("blue")}
                >Cyan</button
              >
              <button type="button" onclick={() => setTheme("purple")}
                >Purple</button
              >
              <button type="button" onclick={() => setTheme("lightblue")}
                >Blue</button
              >
              <button type="button" onclick={() => setTheme("gray")}
                >Gray</button
              >
              <button type="button" onclick={() => setTheme("white")}
                >White</button
              >
              <button type="button" onclick={() => setTheme("black")}
                >Black</button
              >
            {:else if page_number == 7}
              <button type="button" onclick={() => reset()}
                >Reset all settings and reload</button
              >
            {/if}
          </div>
        {/key}

        <div class="bottomitem">
          {#if page_number < 7}
            <Right
              onclick={() => {
                page_number++;
              }}
            />
          {:else}
            <div class="spacebox"></div>
          {/if}
        </div>
      {:else}
        <div class="bottomitem">
          {#if ftsetup_page == 0}
            <button
              type="button"
              onclick={() => {
                ftsetup_page = 1;
              }}>Start tour</button
            >
            <button
              type="button"
              onclick={() => {
                ftsetup_page = 5;
              }}>Go to first time setup</button
            >
            <button type="button" onclick={() => skip_tour()}
              >Skip entirely</button
            >
          {:else if ftsetup_page <= 5}
            <button
              type="button"
              onclick={() => {
                ftsetup_page++;
              }}>Next</button
            >
          {:else if ftsetup_page == 6}
            <button type="button" onclick={() => setSecond(0)}>Off</button>
            <button type="button" onclick={() => setSecond(1)}>On</button>
            <button type="button" onclick={() => setSecond(2)}
              >On with pulsing</button
            >
          {:else}
            <button type="button" onclick={() => skip_tour()}>Close</button>
          {/if}
        </div>
      {/if}
    </div>
  {/if}
</bg>
