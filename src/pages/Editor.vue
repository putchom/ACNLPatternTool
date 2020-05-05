<template v-for="i in 1">
  <div class="editor">
    <main>
      <div class="left">
        <div class="previews">
          <div class="2D">
            <canvas class="fordrawing" ref="canvas2" width="128" height="128"/>
            <canvas class="fordrawing" ref="canvas3" width="64" height="64"/>
            <div class="pattern-info">
              <div class="pattern_title">
                {{$t("editor.title")}}: {{patTitle}}
              </div>
              <div class="pattern_author">
                {{$t("editor.author")}}: {{patAuthor}}
              </div> 
              <div class="pattern_town">
                {{$t("editor.town")}}: {{patTown}}
              </div>
              <div class="pattern_typename">
                {{$t("editor.type")}}: {{patTypeName}}
              </div>
            </div>
            <button class="editInfo" @click="patInfoModal=true">
              {{$t("editor.change")}}
            </button>
          </div>
          <div class="render-preview">
            <ThreeDRender :width="196" :height="300" :drawing-tool="drawingTool"/>
          </div>
        </div>
      </div>
      <div class="center">
        <Palette
          ref="palette"
          :drawing-tool="drawingTool"
          @changed-current-color="onChangedCurrentColor"/>
        <button @click="openColorPicker">
          {{$t("editor.open_color_editor")}}
        </button>
        <canvas class="fordrawing" ref="canvas1" width="512" height="512"/>
        <div class="colorPicker-menu" ref="colorPickerMenu">
          <ColorPicker
            ref="colorPicker"
            :drawing-tool="drawingTool"
            @color-picked="onColorPicked"/>
          <button @click="closeColorPicker">
            {{$t("editor.close")}}
          </button>
        </div>
      </div>
      <div class="right">
        <div class="tools-and-colors">
          <ToolSelector @newtool="toolChange" @newtoolalt="toolChangeAlt" />
          <div class="tool-buttons">
            <button @click="convertImage = true">
              {{$t("editor.convert")}}
            </button>
            <button @click="onQROpen">
              {{$t("editor.generate_qr_code")}}
            </button>
          </div>
        </div>
      </div>
    </main>
    <ModalContainer v-if="qrCode" @modal-close="qrCode=false">
      <div class="modal">
        <div class="modal-header">
          {{$t("editor.generate_qr_code")}}
        </div>
        <div class="modal-window modal-centered">
          <ACNLQRGenerator :pattern="qrCode" />
          <button @click="downPNG">
            {{$t("editor.save_image")}}
          </button>
        </div>
      </div>
    </ModalContainer>
    <ModalContainer v-if="convertImage" @modal-close="convertImage=false">
      <div class="modal">
        <div class="modal-header">
          {{$t("editor.convert")}}
        </div>
        <ImageLoader class="modal-window" :pattern-type="patType" @converted="onConvert" />
      </div>
    </ModalContainer>
    <ModalContainer v-if="patInfoModal" @modal-close="patInfoSave">
      <div class="modal">
        <div class="modal-header">
          {{$t("editor.edit_pattern_details")}}
        </div>
        <div class="modal-window" id="change-info-modal">
          <div class="edit-info">
              <span>{{$t("editor.title")}}: <input type="text" maxlength="20" v-model="patTitle"></span>
              <span>{{$t("editor.author")}}: <input type="text" maxlength="9" v-model="patAuthor"></span>
              <span>{{$t("editor.town")}}: <input type="text" maxlength="9" v-model="patTown"></span>
              <span>{{$t("editor.type")}}:
                <select v-model="patType">
                  <option
                    v-for="(ti, no) in allTypes"
                    :key="no"
                    :value="no">{{ti.name}}
                  </option>
                </select>
              </span>
          </div>
          <div class="edit-buttons">
            <button @click="patInfoModal=false; onLoad()">
              {{$t("editor.cancel")}}
            </button>
            <button @click="patInfoSave(false)">
              {{$t("editor.save")}}
            </button>
          </div>
        </div>
      </div>
    </ModalContainer>
  </div>
</template>

<script>
import ColorPicker from '/components/ColorPicker.vue';
import Palette from '/components/Palette.vue';
import ThreeDRender from '/components/ThreeDRender.vue';
import FileLoader from '/components/FileLoader.vue';
import ImageLoader from '/components/ImageLoader.vue';
import ACNLQRGenerator from '/components/ACNLQRGenerator.vue';
import IconGenerator from '/components/IconGenerator.vue';
import ModalContainer from '/components/ModalContainer.vue';
import ToolSelector from '/components/ToolSelector.vue';
import DrawingTool from '/libs/DrawingTool';
import ACNLFormat from '/libs/ACNLFormat';
import origin from '/libs/origin';
import logger from '/utils/logger';
import lzString from 'lz-string';
import { saveAs } from 'file-saver';
import generateACNLQR from "/libs/ACNLQRGenerator";

export default {
  name: "Editor",
  components: {
    ColorPicker,
    Palette,
    ThreeDRender,
    FileLoader,
    ImageLoader,
    ACNLQRGenerator,
    IconGenerator,
    ModalContainer,
    ToolSelector,
  },
  beforeRouteUpdate: function (to, from, next) {
    if (to.hash.length > 1) {
      if (to.hash.startsWith("#H:")){
        origin.view(to.hash.substring(3)).then((r)=>{
          this.drawingTool.load(r);
        });
        next();
        return;
      }
      let newHash = lzString.compressToEncodedURIComponent(this.drawingTool.toString());
      if (to.hash !== "#" + newHash) {
        this.drawingTool.load(lzString.decompressFromEncodedURIComponent(to.hash.substring(1)));
      }
    }
    next();
  },
  data: function() {
    return {
      drawingTool: new DrawingTool(),
      qrCode: false,
      patTitle: "Empty",
      patAuthor: "Unknown",
      patTown: "Unknown",
      allTypes: [],
      storedAuthorHuman: false,
      patInfoModal: false,
      fragment: "",
      patType: 9,
      patTypeName: "",
      pickPatterns: false,
      multiName: "Local storage",
      allowMoveToLocal: true,
      convertImage: false,
      pubStyleA: "",
      pubStyleB: "",
      pubStyleC: "",
      pubTypeA: "",
      pubTypeB: "",
      pubTypeC: "",
      pubNSFW: "",
      publishModal: false,
      origin,
    };
  },
  methods: {
    async downPNG(){
      const img = await generateACNLQR(this.drawingTool);
      saveAs(img, this.drawingTool.title+".png");
    },
    patInfoSave(publish=false){
      const patTitle = this.patTitle.trim();
      const patTown = this.patTown.trim();
      const patAuthor = this.patAuthor.trim();
      const titleCheck = patTitle && patTitle !== 'Empty';
      const townCheck = patTown && patTown !== 'Unknown';
      const nameCheck = patAuthor && patAuthor !== 'Unknown';
      if (titleCheck && townCheck && nameCheck){
        this.drawingTool.title = this.patTitle;
        if (this.drawingTool.creator[0] !== this.patAuthor) this.drawingTool.creator = this.patAuthor;
        if (this.drawingTool.town[0] !== this.patTown) this.drawingTool.town = this.patTown;
        if (this.drawingTool.patternType !== this.patType){
          this.drawingTool.patternType = this.patType;
          this.patTypeName = this.drawingTool.typeInfo.name;
        }
        this.patInfoModal = false;
        if (publish) this.onPublish();
        return;
      }
      alert('Please provide a valid pattern name, town name, and player name for this pattern.');
    },
    toolChange(newTool){
      this.drawingTool.drawHandler = newTool;
    },
    toolChangeAlt(newTool){
      this.drawingTool.drawHandlerAlt = newTool;
    },
    onColorPicked: function(color) {
      const currentColor = this.drawingTool.currentColor;
      if (this.drawingTool.getPalette(currentColor) === color) return;
      this.drawingTool.pushUndo();
      this.drawingTool.setPalette(this.drawingTool.currentColor, color);
      logger.info(`color picked: ${color}`);
    },
    onChangedCurrentColor: function(idx) {
      if (this.drawingTool.currentColor === idx) return;
      this.drawingTool.currentColor = idx;
      this.drawingTool.onColorChange();
      logger.info(`changed current color: ${idx}`);
    },
    onLoad: async function(t){
      let patStr = this.drawingTool.toString();
      this.patType = this.drawingTool.patternType;
      this.patTypeName = this.drawingTool.typeInfo.name;
      this.patTitle = this.drawingTool.title;
      this.patAuthor = this.drawingTool.creator[0];
      this.patTown = this.drawingTool.town[0];

      await this.$nextTick();
      await this.$nextTick();

      return;
    },
    extLoad: function(data) {
      this.drawingTool.load(data);
    },
    onConvert: function(patterns){
      this.convertImage = false;
      if (patterns.length == 1){
        this.extLoad(patterns[0]);
      }else{
        this.multiName = "Conversion result";
        this.pickPatterns = patterns;
        this.allowMoveToLocal = true;
      }
    },
    extMultiLoad: function(data) {
      this.multiName = "Load which?";
      this.pickPatterns = data;
      this.allowMoveToLocal = true;
    },
    onQROpen: function() {
      const patStr = this.drawingTool.toString();
      this.qrCode = patStr;
    },
    pickPattern: function(p){
      this.extLoad(p);
      this.pickPatterns = false;
    },
    closePicks: function() {
      this.pickPatterns = false;
    },
    openColorPicker: function() {
      if (this.drawingTool.currentColor !== 15) {
        this.$data.colorPickerMenu = !this.$data.colorPickerMenu;
        this.$refs.colorPickerMenu.style.height = ((!this.$data.colorPickerMenu)?0:315)+'px';
        return;
      }
      alert('This one has to stay transparent. :)');
    },
    closeColorPicker: function() {
      this.$refs.colorPickerMenu.style.height = '0px';
    }
  },
  mounted: function() {
    if (localStorage.getItem("author_acnl")){
      this.drawingTool.authorStrict = localStorage.getItem("author_acnl");
      this.storedAuthorHuman = this.drawingTool.creator[0]+" / "+this.drawingTool.town[0];
    }
    this.drawingTool.addCanvas(this.$refs.canvas1, {grid:true});
    this.drawingTool.addCanvas(this.$refs.canvas2);
    this.drawingTool.addCanvas(this.$refs.canvas3);
    this.allTypes = this.drawingTool.allTypes;
    this.drawingTool.onLoad(this.onLoad);
    if (this.$router.currentRoute.hash.length > 1){
      const hash = this.$router.currentRoute.hash.substring(1);
      if (hash.startsWith("H:")){
        origin.view(hash.substring(2)).then((r)=>{
          this.drawingTool.load(r);
        });
      }else{
        this.drawingTool.load(lzString.decompressFromEncodedURIComponent(hash));
      }
    }
    else{
      this.onLoad();
      this.drawingTool.render();
    }

    document.addEventListener('keydown', (e) => {
      if (e.ctrlKey && e.key === 'Z'){
        this.drawingTool.redo();
        e.preventDefault();
        return;
      }
      if (e.ctrlKey && e.key === 'z'){
        this.drawingTool.undo();
        e.preventDefault();
        return;
      }
    });
  },
}
</script>

<style lang="scss" scoped>
button, input[type="button"] {
  border: none;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  font-size: 13px;
  font-weight: 800;
  text-transform: uppercase;
  padding: 10px 14px;
}
button.editInfo {
  margin: 0 0 5px;
}
.editor {
  user-select: none;
}
.modal {
  flex-direction: column;
  align-items: center;
  justify-content: center;
  display:flex;
  margin:50px auto;
  max-width: 80%;
}
.modal-centered{
  flex-direction: column;
  align-items: center;
  justify-content: center;
  display:flex;
}
.modal-header {
  background-color: #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 10px 30px 0;
  min-width: 300px;
}
.modal-window {
  background-color: #ffffff;
  padding: 25px;
  min-width: 600px;
}
.modal-info {
  display: flex;
  justify-content: center;
  width: 100%;
  position: fixed;
  top: 55%;
  color: #ffffff;
  font-size: 32px;
  text-align: center;
}
.info-text {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: rgba(21, 50, 69, 0.7);
  border-radius: 35px;
  padding: 40px;
  min-width: 800px;
  min-height: 150px;
  max-width: 60%;
  max-height: 60%;
}
.info-text p {
  width: 350px;
  display: inline-block;
  line-height: 40px;
}
.info-text span{
  color: #00C3C9;
}
.topbar-buttons {
  display: inline-flex;
  align-items: center;
  justify-content: space-evenly;
  height: 62px;
}
main {
  display: flex;
  flex-direction: row;
  justify-content: center;
}
main .left, .center, .right {
  display: flex;
  flex-direction: column;
}
main .left {
  padding-right: 40px;
}
main .center canvas, main .left canvas {
  box-shadow: 0px 12px 12px -3px rgba(0,0,0,0.3);
}
main .center .colorPicker-menu {
  height: 0;
  position: fixed;
  z-index: 1;
  overflow: hidden;
  transition: 0.5s;
  top: 60px;
  left: 50%;
  transform: translate(-50%);
}
main .center .colorPicker-menu button {
  display: block;
  margin: 0 auto;
}
.bottom-buttons {
  padding: 20px 10% 0 0;
  text-align: right;
}
.svg {
  padding: 5px;
  pointer-events: none;
}
.svg-small {
  height: 10px;
  width: 10px;
  padding: 5px;
  pointer-events: none;
}
.svg.nav {
  border-radius: 100%;
  margin-right: 5px;
  height: 25px;
  width: 25px;
}
.svg.toolbar{
  height: 50px;
  width: 50px;
}
.svg.white-circle {
  background-color: #ffffff;
}
.svg.brown-circle {
  background-color: #EFF1D8;
}
.previews {
  display: inline-flex;
  flex-direction: column;
  justify-content: space-between;
  padding-top: 62px;
  height: 512px;
}
.tools-and-colors {
  height: 512px;
  display: inline-flex;
  flex-direction: row;
  align-items: center;
  border-radius: 0 35px 35px 0;
}
.tool-buttons {
  display: inline-flex;
  flex-direction: column;
  align-items: right;
}
.tool-buttons button{
  margin:5px;
}
canvas.fordrawing{
  background: repeating-linear-gradient(-45deg, #ddd, #ddd 5px, #fff 5px, #fff 10px);
}
.pattern-info{
  margin: 10px 0;
  max-width: 196px;
  overflow: hidden;
}
.render-preview{
  border: 3px solid #7e7261;;
  width: 196px;
  height: 300px;
  border-radius: 5px;
}
#change-info-modal.modal-window{
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
#change-info-modal.modal-window .edit-info{
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 500px;
  max-width: 60%;
}
#change-info-modal.modal-window .edit-info span{
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  width: 60%;
}
#change-info-modal.modal-window .edit-notice{
  max-width: 600px;
  margin: 20px;
}
#publish-modal.modal-window{
  display: flex;
  flex-direction: row;
  align-items: center;
}
#publish-modal.modal-window ol {
  list-style: decimal;
  margin: 10px 0;
  padding: 0 0 0 30px;
}
#publish-modal.modal-window .left,
#publish-modal.modal-window .right {
  flex: 1 1 0;
  align-items: center;
  max-width: 400px;
}
#publish-modal.modal-window .dropdowns {
  display: flex;
  flex-direction: column;
  width: 100%;
}
#publish-modal.modal-window .dropdown, #publish-modal.modal-window span {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}
</style>
