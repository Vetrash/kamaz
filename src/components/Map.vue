<template>
  <div class="mapContainer">
    <div class="mapMenu">
      <div>
        <input type="checkbox" v-model="isRandomColorTile" /> Рандомный цвет
        тайтла
      </div>
      <div>
        <input type="checkbox" v-model="isShowTitle" /> Показать надпись тайтла
      </div>
    </div>

    <div id="cesiumContainer"></div>
    <div class="loadingOverlay" v-if="!isLoaded">
      <h1>Loading...</h1>
    </div>
  </div>
</template>

<script>
import * as Cesium from "cesium";
import "cesium/Source/Widgets/widgets.css";

export default {
  data() {
    return {
      isRandomColorTile: false,
      isShowTitle: false,
      viewer: null,
      isLoaded: false,
      dateText: "",
      visibleTitle: [],
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

      const silhouetteBlue =
        Cesium.PostProcessStageLibrary.createEdgeDetectionStage();
      silhouetteBlue.uniforms.color = Cesium.Color.LIME;
      silhouetteBlue.uniforms.length = 0.01;
      silhouetteBlue.selected = [];

      this.viewer.scene.postProcessStages.add(
        Cesium.PostProcessStageLibrary.createSilhouetteStage([silhouetteBlue])
      );

      const handler = new Cesium.ScreenSpaceEventHandler(this.viewer.canvas);
      handler.setInputAction((movement) => {
        silhouetteBlue.selected = [];
        if (!this.isRandomColorTile) return;
        const pickedFeature = this.viewer.scene.pick(movement.endPosition);

        if (
          (currentFeature.feature === pickedFeature ||
            !Cesium.defined(pickedFeature)) &&
          currentFeature?.feature?.content?.tile
        ) {
          currentFeature.feature.content.tile.color = Cesium.Color.clone(
            currentFeature.originalColor,
            currentFeature.feature.color
          );
          return;
        }

        if (!pickedFeature?.content?.tile) return; // проверка что получили  tile

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
          pickedFeature.content.tile.color = darkColor;
          // pickedFeature.primitives.outlineColor         = Cesium.Color.RED
          // console.log(pickedFeature)
        }
      }, Cesium.ScreenSpaceEventType.MOUSE_MOVE);
    },

    createTitles() {
      this.viewer.entities.removeAll(); // чистим все записи дат
      this.visibleTitle.forEach((title) => {
        const { center, radius } = title.boundingSphere;

        const titleBody = {
          position: center,
          label: {
            font: "24px Helvetica",
            text: this.dateText,
            fillColor: Cesium.Color.WHITE.withAlpha(0.7),
            outlineColor: Cesium.Color.BLACK,
            outlineWidth: 2,

            heightReference: Cesium.HeightReference.CLAMP_TO_3D_TILE, //RELATIVE_TO_3D_TILE//
            verticalOrigin: Cesium.VerticalOrigin.TOP,

            disableDepthTestDistance: 7500000 * 1.5, //эмпирическое число чтобы были видны подписи до центра земли, а после были скрыты
          },
        };
        this.viewer.entities.add(titleBody);
      });
    },

    async eventTitle() {
      let updateTiles = false; //тригер о необходимости обновления лейблов

      this.viewer.camera.moveStart.addEventListener(() => {
        //очищаем лейблы при движении камеры чтобы не было концентрации лейблов при уменьшении масштаба
        this.viewer.entities.removeAll();
      });

      this.viewer.camera.moveEnd.addEventListener(() => {
        //тригерим обновление если камера двинулась и мы ожидаем подписи тайтлов
        if (this.isShowTitle) {
          updateTiles = true;
        }
      });

      this.viewer.scene.postRender.addEventListener((event) => {
        if (updateTiles) {
          this.visibleTitle = event.primitives.get(0)._selectedTiles; //обновляем коллекцию тайтлов после перерендера.
          this.createTitles(); //обновляем лейблы
          updateTiles = false;
        }
      });
    },

    async initMap() {
      Cesium.Ion.defaultAccessToken =
        "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJjMTA1OGM2ZS0zMTVjLTQyM2EtOTBmYy01MmNhNGJhYWQ2YTMiLCJpZCI6MjMyMjM2LCJpYXQiOjE3MjI1ODYwOTl9.eFDC9edSCJ_L2KPTNqGvha0ie8fVLwt7-AZjyAHbIeg";

      this.viewer = new Cesium.Viewer("cesiumContainer", {
        baseLayerPicker: false,
        timeline: false,
        navigationHelpButton: false,
        homeButton: false,
        animation: false,
      });

      this.viewer.infoBox.frame.removeAttribute("sandbox");

      // Загрузка 3D тайлов
      try {
        const tileset = await Cesium.createGooglePhotorealistic3DTileset();
        this.viewer.scene.primitives.add(tileset);

        this.isLoaded = true;
      } catch (error) {
        console.log(error);
      }
    },
  },

  async mounted() {
    await this.initMap(); //создается карта и загружаем набор 3d тайлов
    const options = {
      day: "2-digit", // День месяца в формате 02
      month: "2-digit", // Месяц в формате 02
      year: "numeric", // Год в формате 2023
      locale: "ru-RU", // Русская локаль для форматирования
    };
    this.dateText = new Date().toLocaleString("ru-RU", options); // сохраняю в стейт чтобы не пересчитывать строку каждый раз
    this.visibleTitle = this.viewer.scene.primitives.get(0)._selectedTiles
    this.eventColorPick(); // создаем событие по раскраске тайтлов
    this.eventTitle(); // создаем событие по добавлению надписи на тайтл
  },

  watch: {
    isRandomColorTile(val) {
      const primitives = this.viewer.scene.primitives.get(0);
      primitives.debugColorizeTiles = val; //переключаем рандомный окрас тайла
    },

    isShowTitle(val) {
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
.mapContainer {
  position: relative;
}
.loadingOverlay {
  position: absolute;
  z-index: 50;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  margin: auto;
  width: 100vw;
  background-color: gray;
}

.mapMenu {
  position: absolute;
  z-index: 40;
  padding: 10px;
  border-radius: 10px;
  background-color: rgba(128, 128, 128, 0.4);
  display: flex;
  flex-direction: column;
  gap: 10px;
  top: 20px;
  left: 20px;
  align-items: start;
}
#cesiumContainer {
  width: 100%;
  height: 100vh;
}
</style>
