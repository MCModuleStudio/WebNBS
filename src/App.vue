<template>
  <div id="app">

    <vue-title :title="tabTitle"></vue-title>

    <!-- Overlays -->
    <overlay v-model="state.showWelcome" ref="welcomeOverlay" dismissable>
      <welcome-overlay></welcome-overlay>
    </overlay>

    <overlay :visible="state.loading">
      <loading-overlay></loading-overlay>
    </overlay>

    <overlay v-model="state.showSongDetails" dismissable>
      <song-details-overlay :song="state.song"></song-details-overlay>
    </overlay>

    <overlay v-model="state.showSettings" dismissable>
      <settings-overlay :options="state.options"></settings-overlay>
    </overlay>

    <!-- Core Interface -->
    <div id="main">
      <toolbar id="toolbar"></toolbar>
      <div id="middle">
        <layer-list :song="state.song" id="layer-list"></layer-list>
        <editor :song="state.song" :editor="state.editor" ref="canvas" id="editor"></editor>
      </div>
      <keyboard id="keyboard" :editor="state.editor"></keyboard>
    </div>

  </div>
</template>

<script>
import * as NBS from "./NBS.js";
import { WebAudioNotePlayer } from "./audio.js";
import Editor from "./components/editor/Editor.vue";
import LayerList from "./components/layers/LayerList.vue";
import Overlay from "./components/overlays/Overlay.vue";
import LoadingOverlay from "./components/overlays/LoadingOverlay.vue";
import WelcomeOverlay from "./components/overlays/WelcomeOverlay.vue";
import SettingsOverlay from "./components/overlays/SettingsOverlay.vue";
import SongDetailsOverlay from "./components/overlays/SongDetailsOverlay.vue";
import VueTitle from "./components/VueTitle.vue";
import Toolbar from "./components/toolbar/Toolbar.vue";
import Keyboard from "./components/keyboard/Keyboard.vue";
import { state } from "@/state.js";

function getQueryVariable(variable)
{
       var query = window.location.search.substring(1);
       var vars = query.split("&");
       for (var i=0;i<vars.length;i++) {
               var pair = vars[i].split("=");
               if(pair[0] == variable){return pair[1];}
       }
       return(false);
}

export default {
  components: {
    Editor,
    Overlay,
    LoadingOverlay,
    WelcomeOverlay,
    SettingsOverlay,
    SongDetailsOverlay,
    Toolbar,
    LayerList,
    Keyboard,
    VueTitle,
  },

  data() {
    return {
      state,
      previousTime: -1,
      lastPlayedTick: -1,
      interval: 0,
    };
  },

  mounted() {
    // Load builtin instruments and other assets
    const instruments = NBS.Instrument.builtin;
    Promise.all(instruments.map((i) => i.load()))
      .then(() => {
        if (!getQueryVariable("fromURL")) {
          this.state.loading = false;
          this.state.showWelcome = true;
        } else {
          fetch(getQueryVariable("fromURL")).then(r => r.arrayBuffer()).then(buf => {
            const song = NBS.Song.fromArrayBuffer(buf)
            this.state.setSong(song)
            this.state.loading = false
          })
        }
        this.interval = setInterval(() => this.tick());
      });

    window.onbeforeunload = (e) => {
      if (localStorage != undefined) {
        localStorage['options'] = JSON.stringify(this.state.options)
      }
      if (this.state.editor.modified) {
        // Most browsers don't actually show this message.
        return "Your changes might not be saved";
      }
    };
  },

  beforeDestroy() {
    // Stop the frame loop if the app is unmounted for whatever reason.
    clearInterval(this.interval);
  },

  computed: {
    // The title of the browser tab.
    tabTitle() {
      // If no song name & not edited, "WebNBS"
      // If no song name & edited, "*WebNBS"
      // If song name & not edited, "Song Name - WebNBS"
      // If song name & edited, "*Song Name - WebNBS"

      const base = state.song.name ? `${state.song.name} - WebNBS` : "WebNBS";
      if (state.editor.modified) {
        return `*${base}`;
      }
      return base;
    }
  },

  methods: {
    /**
     * Advanced the song forward.
     */
    advanceSong(time, timePassed) {
      const song = this.state.song;

      if (song.paused) {
        return;
      }

      // Handle the song ending.
      if (song.currentTick >= song.size) {
        if (this.state.options.loop) {
          song.currentTick = 0;
        } else {
          song.paused = true;
        }
        return;
      }

      const ticksPassed = timePassed / song.timePerTick;
      song.currentTick += ticksPassed;

      // song.tick is a getter that uses currenTick, we do not have to manually set it.
      if (song.tick === this.lastPlayedTick) {
        return;
      }
      this.lastPlayedTick = song.tick;

      for (const layer of song.layers) {
        const note = layer.notes[song.tick];
        if (note&&!layer.locked) {
          state.playNote(note, layer);
          note.lastPlayed = time;
        }
      }
    },

    /**
     * Global frame loop.
     */
    tick() {
      const time = performance.now();

      // Determine the amount of time that has passed, up to 500 ms
      // If the time is above 500 ms then most likely the timer stopped for a bit (tabbed out, or something)
      // and skipping ahead 30 seconds if the user has been gone 30 seconds is bad design.
      const timePassed = Math.min(time - this.previousTime, 500);
      this.previousTime = time;

      // Advance the song forward.
      this.advanceSong(time, timePassed);

      // Draw the canvas after updating the song.
      if (this.$refs.canvas) {
        this.$refs.canvas.update(time);
      }
    },
  }
}
</script>

<style>
/* Make body look half decent */
body {
  font-family: sans-serif;
  margin: 0;
}

/* Make links look more like links by default */
a {
  cursor: pointer;
}
a:hover {
  text-decoration: underline;
}

/* Hide spinners on some numer inputs */
.no-spinners {
  -moz-appearance: textfield;
}
.no-spinners::-webkit-outer-spin-button,
.no-spinners::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.flex {
  display: flex;
}
.flex-row {
  flex-direction: row;
}
.flex-column {
  flex-direction: column;
}
.flex-center {
  align-items: center;
}

#main {
  display: flex;
  flex-direction: column;
  width: 100vw;
  height: 100vh;
}

#middle {
  /* I don't know why, but I can't get flex to work here. */
  /* Once the canvas needs a vertical scrollbar, it doesn't grow to match. */
  /* I really don't want to use grid, but it's the only thing I can get to work. */
  /* display: flex;
  flex-direction: row; */

  flex: 1;
  display: grid;
  grid-template-rows: auto;
  grid-template-columns: 200px auto;

  overflow-y: auto;
  background-image: url("assets/layersbackground.jpg");
  background-attachment: local;
  border-top: 1px solid #777;
  border-bottom: 1px solid #777;
}
#layer-list {
  border-right: 1px solid #777;
  width: 200px;
}
#editor {
  flex: 1;
}
#keyboard {
  height: 130px;
}
</style>
