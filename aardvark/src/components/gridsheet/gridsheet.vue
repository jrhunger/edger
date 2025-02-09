<template>
  <div class="grid" @mousemove="headerResizeMove" @mouseup="onMouseUpSvg()">
    <svg :width="positionLeft(headerObj.length + 1) + 1" height="24">
      <g
        v-for="(col, ci) in headerObj"
        :key="ci"
        :transform="translateCol(ci)"
        @mousedown="startColumnSelect(ci)"
        @mousemove="changeColumnSelect(ci)"
        @mouseup="endColumnSelect"
      >
        <rect :height="rowHeight" :width="widthAt(ci)" class="col-header" x="0" y="0" />

        <text
          :height="rowHeight"
          :width="widthAt(ci)"
          :x="widthAt(ci) / 2"
          class="col-header__text"
          text-anchor="middle"
          y="12"
        >{{ col.name }}
        </text>

        <rect
          :class="{'active': ci === headerResizeAt}"
          :height="rowHeight"
          :width="5"
          :x="widthAt(ci) - 5"
          :y="0"
          class="col-header__resize"
          @mousedown.stop="headerResizeStart(ci)"
        />
      </g>
    </svg>

    <div
      ref="wrapper"
      style="height: 400px;
  overflow: scroll;
  position: relative;"
    >
      <svg :height="data.length * 24" :width="positionLeft(data.length + 1) + 1">
        <g
          v-for="(row, ri) in data"
          :key="ri"
          :transform="translateRow(ri)"
        >
          <g
            v-for="(col, ci) in data[ri]"
            v-if="col.length != 0"
            :key="ci"
            :fill="getColor(ci,ri)"
            :transform="translateCol(ci)"
            @mousedown="onMouseDownCell(ci, ri)"
            @mousemove="onMouseMoveCell(ci, ri)"
          >

            <rect :height="rowHeight" :width="widthAt(ci)" x="0" y="0" />
            <text :height="rowHeight" :width="widthAt(ci)" x="2" y="12">{{ col }}</text>
          </g>
        </g>
        <rect
          :height="selectionCount.h * rowHeight"
          :transform="selectionTransform"
          :width="selectionSize.w"
          class="selection"
          x="0"
          y="0"
        />
      </svg>
      <div :style="editorStyleObj" class="editor__frame">
        <input
          ref="hiddenInput"
          v-model="editingText"
          :class="{'editor--visible': editing}"
          autofocus
          class="editor__textarea"
          @blur="onBlur"
          @mousedown="onMouseDownCell(selection.c, selection.r)"
        >
      </div>
    </div>
  </div>
</template>

<script>
import "bootstrap-reboot-import";
import Vue from "vue";
import * as math from "lodash-es";


export default {

  name: "VueGridSheet",
  props: {
    data: Array,
    format: Array,
    state: Array,
  },

  data () {
    return {

      selection: {
        c: 0,
        r: 0,
        sc: 0,
        sr: 0,
      },

      editingText: "",
      editing: true,
      rowHeight: 24,
      selectionMode: false,
      selectionModeColumn: false,
      headerResizeAt: -1,
    };
  },

  computed: {
    editorStyleObj () {
      return {
        left: this.positionLeft(this.selection.c) + "px",
        top: this.selection.r * 24 + "px",
        width: this.selectionSize.w + "px",
      };
    },
    selectionTransform () {
      return `translate(${this.positionLeft(this.selectionCount.c)},  ${this
        .selectionCount.r * 24})`;
    },
    selectionCount () {
      return {
        r:
            this.selection.r <= this.selection.sr
              ? this.selection.r
              : this.selection.sr,
        c:
            this.selection.c <= this.selection.sc
              ? this.selection.c
              : this.selection.sc,
        w: Math.abs(this.selection.sc - this.selection.c) + 1,
        h: Math.abs(this.selection.sr - this.selection.r) + 1,
      };
    },
    selectionSize () {
      return {
        r: this.positionLeft(this.selectionCount.r),
        c: this.positionLeft(this.selectionCount.c),
        w:
            this.positionLeft(this.selectionCount.c + this.selectionCount.w) -
            this.positionLeft(this.selectionCount.c),
        h:
            this.positionLeft(this.selectionCount.c + this.selectionCount.h) -
            this.positionLeft(this.selectionCount.c),
      };
    },
    headerObj () {
      const width = 1000;
      const colWidth = math.floor(width / 17)
      const firstColWidth = width - colWidth * 16
      let myHeader = [];
      if (myHeader.length === 0) {
        myHeader.push({name: "", width: firstColWidth})
        for (let j = 0; j < 16; j++) {
          myHeader.push({name: j, width: colWidth});
        }
      }
      return myHeader
    },
  },

  methods: {

    getColor (ci, ri) {
      if (ci === 0) {
        return "green";
      }
      //console.log(this.state)
      for (let k = 0; k < this.state.length; k++) {
        // console.log(this.state[k])

        if (this.state[k][0] === ci && this.state[k][1] === ri) {
          return "red"
        }
      }
      return "lightgray"
    },
    startColumnSelect (c) {

      this.selection.sr = this.data.length - 1;
      this.selection.r = 0;
      this.selection.sc = c;
      this.selection.c = c;
      this.selectionModeColumn = true;
    },
    changeColumnSelect (c) {
      if (this.selectionModeColumn) {
        this.selection.c = c;
      }
    },
    endColumnSelect () {
      this.selectionModeColumn = false;
    },
    headerResizeStart (c) {
      this.headerResizeAt = c;
    },
    headerResizeEnd () {
      this.headerResizeAt = -1;
    },
    headerResizeMove (e) {
      const headerRect = e.target.parentNode.parentNode.getBoundingClientRect();
      const headerMouseX = e.clientX - headerRect.left;
      if (this.headerResizeAt >= 0) {
        const updateWidth =
            headerMouseX - this.positionLeft(this.headerResizeAt);
        this.headerObj[this.headerResizeAt].width =
            updateWidth > 30 ? updateWidth : 30;
      }
    },
    widthAt (index) {
      return this.headerObj[index].width;
    },
    positionLeft (index) {
      return this.headerObj
        .slice(0, index)
        .map(it => it.width)
        .reduce((a, b) => a + b, 0)
    },

    setDataAt (ci, ri, value) {


      let notfound = true
      for (let k = 0; k < this.state.length; k++) {

        if (this.state[k][0] === ci && this.state[k][1] === ri) {
          notfound = false
          break
        }
      }
      if (notfound) {
        // eslint-disable-next-line vue/no-mutating-props
        this.state.push([ci, ri])
      }
      Vue.set(this.data[ri], ci, value);
      this.$emit("update", this.data);
    },
    getDataAt (c, r) {
      return this.data[r][c];
    },
    onBlur () {
      this.editing = false;
      this.setDataAt(this.selection.c, this.selection.r, this.editingText);
    },
    translateCol (ci) {
      return `translate(${this.positionLeft(ci)}, 0)`;
    },
    translateRow (ri) {
      return `translate(0, ${ri * 24})`;
    },
    onMouseDownCell (c, r) {
      if (this.editing) {
        this.onBlur();
      }
      if (
        this.selectionCount.c === c &&
          this.selectionCount.r === r &&
          this.selectionCount.w === 1 &&
          this.selectionCount.h === 1
      ) {
        this.editHere();
        return;
      }
      this.setSelectionStart(c, r);
      Vue.nextTick(() => {
        this.$refs["hiddenInput"].focus();
      });
    },
    setSelectionSingle (c, r) {
      this.selection.c = c;
      this.selection.r = r;
      this.selection.sc = c;
      this.selection.sr = r;
      this.setEditingText();
    },
    setSelectionStart (c, r) {
      this.setSelectionSingle(c, r);
      this.selectionMode = true;
    },
    onMouseMoveCell (c, r) {
      if (this.selectionMode) {
        this.setSelectionEnd(c, r);
      }
    },
    onMouseUpSvg () {
      this.endSelection();
      this.headerResizeEnd();
    },
    setSelectionEnd (c, r) {
      if (this.selectionMode) {
        this.selection.c = c;
        this.selection.r = r;
        this.setEditingText();
      }
    },
    endSelection () {
      this.selectionMode = false;
    },
    editCell (ci, ri) {
      this.editing = true;
      Vue.nextTick(() => {
        this.$refs.hiddenInput.focus();
      });
    },
    editHere () {
      this.editCell(this.selection.c, this.selection.r);
    },
    clearCell (ci, ri) {
      this.setDataAt(ci, ri, "");
    },
    clearSelection () {
      for (let i = 0; i < this.selectionCount.h; i++) {
        for (let j = 0; j < this.selectionCount.w; j++) {
          this.clearCell(this.selectionCount.c + j, this.selectionCount.r + i);
        }
      }
    },
    isInsideTable (c, r) {
      if (c < 0) {
        return false;
      }
      if (r < 0) {
        return false;
      }
      if (c > this.data[0].length - 1) {
        return false;
      }
      if (r > this.data.length - 1) {
        return false;
      }
      return true;
    },
    moveCursor (dc, dr) {
      if (!this.isInsideTable(this.selection.c + dc, this.selection.r + dr)) {
        return;
      }
      if (this.selectionMode) {
        this.setSelectionEnd(this.selection.c + dc, this.selection.r + dr);
        this.fixScroll();
        return;
      }
      if (this.editing) {
        this.onBlur();
      }
      this.setSelectionSingle(this.selection.c + dc, this.selection.r + dr);
      this.fixScroll();
    },
    moveInputCaretToEnd () {
      const el = this.$refs["hiddenInput"];
      el.setSelectionRange(this.editingText.length, this.editingText.length);
    },
    fixScroll () {
      const el = this.$refs["wrapper"];

      if (el.scrollTop > this.selection.r * 24) {
        el.scrollTop = this.selection.r * 24;
      }
      if (el.scrollTop < this.selection.r * 24 - el.clientHeight + 24) {
        el.scrollTop = this.selection.r * 24 - el.clientHeight + 24;
      }

      if (
        el.scrollLeft <
          this.positionLeft(this.selection.c) - el.clientWidth
      ) {
        el.scrollLeft = this.positionLeft(this.selection.c);
      }
    },
    setEditingText () {
      this.editingText = this.getDataAt(this.selection.c, this.selection.r);
    },
  },
  mounted () {
    this.editingText = this.getDataAt(0, 0);
    this.onBlur();

    const target = this.$el;

    target.onkeydown = e => {
      switch (e.keyCode) {
        case 8: //backspace
          if (!this.editing) {
            this.moveInputCaretToEnd();
            this.editHere();
          }
          break;
        case 37: //left
          this.moveCursor(-1, 0);
          e.preventDefault();
          break;
        case 38: //up
          this.moveCursor(0, -1);
          e.preventDefault();
          break;
        case 39: //right
          this.moveCursor(1, 0);
          e.preventDefault();
          break;
        case 40: //down
          this.moveCursor(0, 1);
          e.preventDefault();
          break;
        case 46: //delete
          this.clearSelection();
          break;
        case 13: //enter
          this.moveCursor(0, 1);
          break;
        case 16: //shift
          this.setSelectionStart(this.selection.c, this.selection.r);
          break;
        case 91: //ctrl
          break;
        case 113: //F2
          if (!this.editing) {
            this.moveInputCaretToEnd();
            this.editHere();
          }
          break;
        default:
          if (!this.editing) {
            this.editingText = "";
            this.editHere();
          }
          break;
      }
    };
    target.onkeyup = e => {
      switch (e.keyCode) {
        case 16: //shift
          this.endSelection();
          break;
        default:
      }
    };
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
rect {
  fill: none;
  shape-rendering: crispEdges;
  stroke: #999;
}

.selection {
  fill: none;
  stroke: rgb(158, 55, 255);
  stroke-width: 3px;
}

.col-header {
  fill: #eee;
}

.col-header__resize {
  cursor: col-resize;
  opacity: 0;
}

.col-header__resize:hover {
  cursor: col-resize;
  fill: rgb(158, 55, 255);
  opacity: 0.5;
}

.col-header__resize.active {
  fill: rgb(158, 55, 255);
  opacity: 0.5;
}

.grid {
  background: #eee;
  position: relative;
  width: 100%;
}

.editor__frame {
  overflow: hidden;
  position: absolute;
}

text {
  dominant-baseline: central;
  user-select: none;
}

input {

  box-sizing: border-box;
  margin-left: 2px;
  outline: 0;
}

svg {
  display: block;
}

.editor__textarea {
  opacity: 0;
  width: 100%;
}

.editor--visible {
  opacity: 1;
}
</style>
