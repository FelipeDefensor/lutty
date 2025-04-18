<template>
  <svg class="fretboard" :width="width + 150" :height="height + 100">
    <!--
    <text font-size="11" x="10" y="20" fill="black">TODO: Tuning: E A D G</text>
    -->

    <g transform="translate(80, 50)">
      <!-- Copyright
      <text
        font-size="11"
        :x="width"
        :y="height + 20"
        fill="gray"

        dominant-baseline="hanging"
        text-anchor="end"
      >
        created with lutty 
      </text>
      -->

      <!-- fret inlays -->
      <polygon
        v-for="inlay in inlay_polys"
        :key="'inlay_' + inlay.fret"
        :points="inlay.points"
        style="fill: #eee"
      />

      <!-- string lines -->
      <line
        v-for="string in strings"
        :key="'string_' + string.nr"
        x1="0"
        :y1="string.y"
        :x2="width"
        :y2="string.y"
        stroke="#000"
        stroke-width="1"
      />

      <!-- nut -->
      <line
        x1="0"
        y1="0"
        x2="0"
        :y2="height"
        stroke="#000"
        stroke-width="5"
        stroke-linecap="round"
      />

      <!-- frets -->
      <line
        v-for="fret in fret_lines.lines"
        :key="'fret_' + fret.nr"
        :x1="fret.x"
        :y1="fret_lines.y1"
        :x2="fret.x"
        :y2="fret_lines.y2"
        stroke="#000"
        :stroke-width="fret.width"
      />

      <!-- notes -->
      <g v-for="string in strings" :key="'ng_' + string.nr">
        <!-- hidden notes -->
        <g v-for="note in string.hidden" :key="note.key">
          <transition name="fade">
            <g v-show="note.pitchClass == hover_note">
              <!-- circle -->
              <circle
                :cx="note.x"
                :cy="string.y"
                r="10"
                stroke-width="1"
                fill="white"
                stroke="white"
              />
              <!-- name -->
              <text
                font-size="11"
                :x="note.x"
                :y="string.y"
                dominant-baseline="central"
                fill="black"
                text-anchor="middle"
              >
                {{ note.name }}
              </text>
            </g>
          </transition>
          <circle
            @mouseleave="hover_note = -1"
            @mouseover="hover_note = note.pitchClass"
            r="10"
            :cx="note.x"
            :cy="string.y"
            fill="transparent"
          />
        </g>

        <!-- visible notes -->
        <transition-group name="list" tag="g" appear>
          <g v-for="note in string.visible" :key="note.key">
            <!-- circle -->
            <circle
              :cx="note.x"
              :cy="string.y"
              r="10"
              :stroke-dasharray="hover_note == note.pitchClass ? '4,4' : '0'"
              :fill="
                shouldEmphasize(string.nr, note.fret, note.pitchClass)
                  ? 'black'
                  : 'white'
              "
              stroke="black"
            />
            <!-- name -->
            <text
              font-size="11"
              :x="note.x"
              :y="string.y"
              dominant-baseline="central"
              :fill="
                shouldEmphasize(string.nr, note.fret, note.pitchClass)
                  ? 'white'
                  : 'black'
              "
              :font-weight="
                shouldEmphasize(string.nr, note.fret, note.pitchClass)
                  ? 'bold'
                  : 'normal'
              "
              text-anchor="middle"
            >
              {{ note.name }}
            </text>
            <circle
              @mouseleave="hover_note = -1"
              @mouseover="hover_note = note.pitchClass"
              r="10"
              :cx="note.x"
              :cy="string.y"
              fill="transparent"
            />
          </g>
        </transition-group>
      </g>
    </g>
  </svg>
</template>

<script>
import { Midi } from "@tonaljs/tonal";

// The following has removes all "Loading instrument..."
// divs only if done on this particular module.
// Don't know why.
window.m21conf = { loadSoundfont: false };

export default {
  name: "Fretboard",

  props: {
    tuning: {
      type: Array,
      default: () => [],
    },
    notes: {
      type: Array,
      default: () => [],
    },
    inlays: {
      default: () => [3, 5, 7, 9, 12, 15, 17, 19, 21],
    },
    showInlays: {
      type: String,
      default: "yes",
    },
    emphasize: {
      type: Object,
      default: null,
    },
    frets: {
      type: Number,
      default: 18,
    },
    sharps: {
      type: String,
      default: "sharps",
    },
  },

  data() {
    return {
      string_spacing: 25,
      hover_note: -1,
    };
  },

  computed: {
    width: function () {
      return this.fretpos(this.frets - 1);
    },
    height: function () {
      return (this.tuning.length - 1) * this.string_spacing;
    },
    strings: function () {
      let result = [];
      this.tuning.forEach((tuning, string) => {
        // find notes
        let normalized_notes = this.normalize(this.notes);
        let visible = [];
        let hidden = [];
        for (let fret = 0; fret < this.frets; fret++) {
          let pitchClass = (tuning + fret) % 12;
          let note = {
            pitchClass: pitchClass,
            fret: fret,
            name: this.toname(pitchClass),
            x: (this.fretpos(fret - 1) + this.fretpos(fret)) / 2,
            key: "n" + string + "_" + fret,
          };
          if (normalized_notes.includes(pitchClass)) {
            visible.push(note);
          } else {
            hidden.push(note);
          }
        }
        if (tuning != undefined) {
          result.push({
            nr: string,
            y: string * this.string_spacing,
            tuning: this.toname(tuning),
            visible: visible,
            hidden: hidden,
          });
        }
      });
      return result;
    },
    fret_lines: function () {
      let lines = [];
      for (let i = 1; i < this.frets; i++) {
        lines.push({
          nr: i,
          x: this.fretpos(i),
        });
      }
      return {
        y1: this.height == 0 ? -this.string_spacing / 4 : 0,
        y2: this.height == 0 ? this.string_spacing / 4 : this.height,
        lines: lines,
      };
    },
    inlay_polys: function () {
      if (this.showInlays == "no") return [];
      let result = [];
      if (!this.tuning.length) return result;
      for (let fret of this.inlays) {
        if (fret >= this.frets) break;

        const resize_x = 0.6;
        const resize_y = fret == 12 ? 0.8 : 0.6;

        let height = this.height;
        let top = height / 2 - (height * resize_y) / 2;
        if (height < 10) {
          height = 10;
          top = -5;
        }
        const bottom = top + height * resize_y;
        const left = this.fretpos(fret - 1);
        const right = this.fretpos(fret);
        const width = right - left;

        const nleft = left + width / 2 - (width * resize_x) / 2;
        const nright = nleft + width * resize_x;

        let points;
        if (fret == 12) {
          points = [
            [nleft, top],
            [nright, top],
            [nright, bottom],
            [nleft, bottom],
          ];
        } else {
          points = [
            [left + width / 2, top],
            [nleft, height / 2],
            [left + width / 2, bottom],
            [nright, height / 2],
          ];
        }

        let pointsstr = "";
        for (let point of points) {
          pointsstr += point[0].toString() + "," + point[1].toString() + " ";
        }

        result.push({
          fret: fret,
          points: pointsstr,
        });
      }
      return result;
    },
  },

  methods: {
    fretpos(n) {
      // https://www.liutaiomottola.com/formulae/fret.htm
      if (n <= 20) {
        const s = 1300;
        let d = s - s / Math.pow(2, n / 12);
        return Math.round(d * 1000) / 1000;
      } else {
        let p19 = this.fretpos(19);
        let p20 = this.fretpos(20);
        return p20 + (p20 - p19) * (n - 20);
      }
    },
    toname(x) {
      if (this.sharps == "pc") {
        return Midi.pcset([x])[0];
      } else {
        return Midi.midiToNoteName(x, {
          sharps: this.sharps,
          pitchClass: true,
        });
      }
    },
    normalize(notes) {
      return notes.map((x) => x % 12);
    },
    shouldEmphasize(stringNumber, fret, pitchClass) {
      if (!this.emphasize) return false;
      if (this.emphasize.type == "frets") {
        return this.emphasize.data[stringNumber].includes(fret);
      } else {
        return this.emphasize.data == pitchClass;
      }
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.list-enter-active,
.list-leave-active {
  transition: all 0.4s ease;
}
.list-enter, .list-leave-to /* .list-leave-active below version 2.1.8 */ {
  opacity: 0;
  transform: translateX(20px);
}
</style>
