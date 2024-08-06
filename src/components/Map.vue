<template>
  <input type="checkbox" v-model="isRandomColorTile" /> Рандомный цвет тайла
  <input type="checkbox" v-model="isShowTitle" /> Показать надпись тайтла
  <div id="cesiumContainer"></div>
</template>

<script>
//import Cesium from "cesium/Source/Cesium";
//import { Viewer, Cesium3DTileset, HeadingPitchRange } from "cesium";
import * as Cesium from "cesium";

import "cesium/Source/Widgets/widgets.css";
//import { Viewer } from 'cesium';

export default {
  data() {
    return {
      isRandomColorTile: false,
      isShowTitle: false,
      viewer: null,
    };
  },
  methods: {
    eventColorPick() {
      // Обработка событий наведения мыши
      //создаем обьект для хранения информации предыдущего тайла
      let currentFeature = {
        feature: undefined,
        originalColor: new Cesium.Color(),
      };

      const handler = new Cesium.ScreenSpaceEventHandler(this.viewer.canvas);

      handler.setInputAction((movement) => {
        if (!this.isRandomColorTile) return;
        const pickedFeature = this.viewer.scene.pick(movement.endPosition);

        if (
          currentFeature.feature === pickedFeature ||
          !Cesium.defined(pickedFeature)
        ) {
          currentFeature.feature.content.tile.color = Cesium.Color.clone(
            currentFeature.originalColor,
            currentFeature.feature.color
          );
          return;
        }

        if (!pickedFeature?.content?.tile) return; // проверка что получили  tile  а не title

        if (
          Cesium.defined(currentFeature.feature) &&
          currentFeature.feature !== pickedFeature
        ) {
          // Восстанавливаем оригинальный цвет для тайла, который больше не выбран
          currentFeature.feature.content.tile.color = Cesium.Color.clone(
            currentFeature.originalColor,
            currentFeature.feature.color
          );
          currentFeature.feature = undefined;
        }
        if (
          Cesium.defined(pickedFeature) &&
          pickedFeature !== currentFeature.feature
        ) {
          currentFeature.feature = pickedFeature;
          currentFeature.originalColor = pickedFeature.content.tile.color;

          Cesium.Color.clone(pickedFeature.color, currentFeature.originalColor);
          const darkeningFactor = 0.1;

          const nowColor = pickedFeature.content.tile.color;
          //получаем темный цвет тайла
          const darkColor = {
            red: nowColor.red - darkeningFactor,
            green: nowColor.green - darkeningFactor,
            blue: nowColor.blue - darkeningFactor,
            alpha: nowColor.alpha,
          };
          // Выделяем новый выбранный тайл
          pickedFeature.content.tile.tileset.outlineColor = new Cesium.Color(
            1.0,
            0.0,
            0.0,
            1
          );
          pickedFeature.content.tile.color = darkColor;
        }
      }, Cesium.ScreenSpaceEventType.MOUSE_MOVE);
    },

    createTitles() {
      console.log("test");



      let options = {
        day: "2-digit", // День месяца в формате 02
        month: "2-digit", // Месяц в формате 02
        year: "numeric", // Год в формате 2023
        locale: "ru-RU", // Русская локаль для форматирования
      };

      const dateText = new Date().toLocaleString("ru-RU", options)

      this.viewer.entities.removeAll(); // чистим все записи дат
      const allTail = this.viewer.scene.primitives.get(0)._selectedTiles; // получаю все тайтлы на сцене
      allTail.forEach((title) => {
        const { center, radius } = title.boundingSphere;

        //формирую тайтл
        const titleBody = {
          //disableDepthTestDistance: Number.POSITIVE_INFINITY,
          position: center,
          point: { pixelSize: 10 },

          
            font: "24px Helvetica",
            show: true,
            text:  dateText,
            fillColor: Cesium.Color.WHITE.withAlpha(0.7),
            outlineColor: Cesium.Color.BLACK,
            outlineWidth: 2,
          
        };
        this.viewer.entities.add(titleBody);
      });
    },

    async eventTitle() {
      this.viewer.camera.moveEnd.addEventListener(() => {
        this.createTitles();
      });
    },

    async initMap() {
      // Grant CesiumJS access to your ion assets
      Cesium.Ion.defaultAccessToken =
        "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJjMTA1OGM2ZS0zMTVjLTQyM2EtOTBmYy01MmNhNGJhYWQ2YTMiLCJpZCI6MjMyMjM2LCJpYXQiOjE3MjI1ODYwOTl9.eFDC9edSCJ_L2KPTNqGvha0ie8fVLwt7-AZjyAHbIeg";

      this.viewer = new Cesium.Viewer("cesiumContainer");
      this.viewer.infoBox.frame.removeAttribute("sandbox");
      this.viewer.scene.debugShowFramesPerSecond = true;

      // Загрузка 3D тайлов
      try {
        const tileset = await Cesium.createGooglePhotorealistic3DTileset(); //await Cesium.Cesium3DTileset.fromIonAssetId(354307);
        this.viewer.scene.primitives.add(tileset);
        await this.viewer.zoomTo(tileset);

        // Apply the default style if it exists
        const extras = tileset.asset.extras;
        if (
          Cesium.defined(extras) &&
          Cesium.defined(extras.ion) &&
          Cesium.defined(extras.ion.defaultStyle)
        ) {
          tileset.style = new Cesium.Cesium3DTileStyle(extras.ion.defaultStyle);
        }
      } catch (error) {
        console.log(error);
      }

      // Загрузка 3D тайлов
      // try {
      //   //const tileset = await Cesium.Cesium3DTileset.fromIonAssetId(96188)
      //   //const tileset = await Cesium.Cesium3DTileset.fromIonAssetId(2275207);
      //   // this.viewer.scene.primitives.add(tileset);
      //   const tileset = await Cesium.Cesium3DTileset.fromIonAssetId(354307);
      //   viewer.scene.primitives.add(tileset);
      //   await viewer.zoomTo(tileset);

      //   //const tileset = await Cesium.Cesium3DTileset.fromIonAssetId(96188);
      //   this.viewer.scene.primitives.add(tileset);
      //   await viewer.zoomTo(tileset);
      // } catch (error) {
      //   console.log(error);
      // }
    },
  },

  async mounted() {
    await this.initMap();
    this.eventColorPick();
    this.eventTitle();
  },
  watch: {
    isRandomColorTile(val) {
      const terrainPrimitives = this.viewer.scene.primitives.get(0);
      if (val) {
        terrainPrimitives.debugColorizeTiles = true;
      } else {
        terrainPrimitives.debugColorizeTiles = false;
      }
    },

    isShowTitle(val) {
      console.log("this.viewer.entities", this.viewer.entities);
      if (val) {
        this.createTitles();
      } else {
        this.viewer.entities.removeAll(); // чистим все записи дат
      }
    },
  },
};
</script>

<style scoped>
#cesiumContainer {
  width: 100%;
  height: 100vh;
}
</style>
