<template v-for="i in 1">
  <div class="editor">
    <Palette
      ref="palette"
      v-bind:drawing-tool="drawingTool"
      v-on:changed-current-color="onChangedCurrentColor"/>
    <ColorPicker
      ref="colorPicker"
      v-bind:drawing-tool="drawingTool"
      v-on:color-picked="onColorPicked"/>
    <div>
      <canvas ref="canvas1" width="512" height="512"/>
      <canvas ref="canvas2" width="128" height="128"/>
      <canvas ref="canvas3" width="64" height="64"/>
    </div>
  </div>
</template>

<script>
import ColorPicker from "/components/ColorPicker.vue";
import Palette from "/components/Palette.vue";
import DrawingTool from "/libs/DrawingTool";
import logger from "/utils/logger";

export default {
  name: "Editor",
  components: {
    ColorPicker,
    Palette
  },
  data: function() {
    return {
      drawingTool: new DrawingTool(),
    };
  },
  methods: {
    onColorPicked: function(color) {
      this.drawingTool.setPalette(this.drawingTool.currentColor, color);
      this.$refs.palette.palChange();
    },
    onChangedCurrentColor: function(idx) {
      this.drawingTool.currentColor = idx;
      this.$refs.colorPicker.forceCheck();
    }
  },
  mounted: function() {
    this.drawingTool.addCanvas(this.$refs.canvas1);
    this.drawingTool.addCanvas(this.$refs.canvas2);
    this.drawingTool.addCanvas(this.$refs.canvas3);
    this.drawingTool.render();
  }
}
</script>

<style scoped>
.editor {
  user-select: none;
}
</style>