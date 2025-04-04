<template>
  <div class="map-view" id="map"></div>
  <div class="popup-container" v-if="feature" @click="closeFeature()">
    <div
      class="feature-container"
      @click="stopPropagation"
      :class="{ 'has-pointcloud': pointcloudUrl }"
    >
      <table>
        <tr class="feature" v-for="item of feature.items">
          <td class="title">{{ displayLastPortion(item.key) }}</td>
          <td>
            <div
              :id="'div-' + item.key"
              v-if="!item.hasFocus"
              @click="setItemInput(item)"
            >
              {{ item.value }}
            </div>
            <input
              v-else
              :id="'btn-' + item.key"
              type="text"
              v-model="item.value"
              @blur="item.hasFocus = false"
              :style="getInputStyle()"
              @input="changeObject(item)"
            />
          </td>
        </tr>
      </table>
      <iframe v-if="pointcloudUrl" :src="pointcloudUrl"></iframe>
      <div class="insert" v-if="showAddLabel()">
        <button @click="addLabel()">Add Label</button>
      </div>
      <div class="close"><button @click="closeFeature()">Close</button></div>
    </div>
  </div>
  <div class="loader" v-if="total > 0">
    <div class="title">Loading data</div>
    <div class="container">
      <div class="progress" :style="{ width: getProgressWidth() }"></div>
    </div>
  </div>
</template>

<style lang="scss">
.leaflet-pane.leaflet-shadow-pane {
  display: none;
}
</style>

<style lang="scss" scoped>
.map-view {
  background-color: #4a5568;
  height: 100%;
  color: white;
  resize: both;
  overflow: hidden;
  position: relative;
  z-index: 1;
}

.loader {
  position: absolute;
  z-index: 1000;
  bottom: 10px;
  left: 10px;
  height: 30px;
  display: flex;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.5);
  color: white;

  .title {
    position: absolute;
    left: 0px;
    z-index: 1;
    padding-left: 5px;
  }

  .container {
    width: 200px;
    padding: 2px;
    height: 100%;

    .progress {
      background-color: transparentize($color: green, $amount: 0.5);
      height: 100%;
    }
  }
}

.popup-container {
  position: absolute;
  z-index: 1000;
  left: 0px;
  top: 0px;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: rgba(0, 0, 0, 0.5);

  .feature-container {
    background-color: white;
    color: black;
    border-radius: 5px;
    padding: 20px;
    box-shadow: 0px 0px 10px 0px #a3a3a3;
    display: flex;
    flex-direction: column;

    &.has-pointcloud {
      width: calc(100% - 20px);
      height: calc(100% - 20px);
    }

    table {
      border-spacing: 0px;

      .feature {
        word-break: break-all;

        &:nth-child(odd) {
          background-color: #f2f2f2;
        }

        .title {
          // width: 20%;
          padding-right: 30px;
          padding-left: 10px;
          padding-top: 5px;
          padding-bottom: 5px;
        }

        td {
          margin: 0;
          border: none;

          div {
            min-height: 24px;
          }

          input {
            padding: 5px;
            font-size: 15px;
            margin: 0;
          }
        }
      }
    }

    iframe {
      margin-top: 10px;
      border: none;
      flex: 1;
    }
  }

  .insert {
    margin-top: 10px;
  }

  .close {
    display: flex;
    justify-content: center;
    margin-top: 20px;
  }
}
</style>

<script>
import L from "leaflet";
import { getFeature } from "../services/api-geo";
import {
  subscribeFeatureVisibiltyChange,
  subscribeFeatureClick,
  subscribeFeatureDoubleClick,
  updateFeature,
} from "../services/geo";

export default {
  data() {
    return {
      map: null,
      feature: null,
      mapObjList: {},
      featuresToDisplay: {},
      unsubscribe: [],
      pointcloudUrl: null,
      total: 0,
      actual: 0,
      inputMinWidth: 0,
    };
  },
  methods: {
    initMap() {
      const mbAttr =
        'Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, ' +
        'Imagery © <a href="https://www.mapbox.com/">Mapbox</a>';
      const mbUrl =
        "https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token=pk.eyJ1IjoiZmVkZXJkaXNwaSIsImEiOiJjbGdreWlncHcwd3F0M2hsdnhscGg5Yzc1In0.Ud5vRdMf9cbtUUd5ufgKXQ";

      const rUrl = "https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png";

      const topplusUrl = "http://sgx.geodatenzentrum.de/wms_topplus_web_open";
      const topplusAttr =
        '&copy Bundesamt für Kartographie und Geodäsie 2017, <a href="http://sg.geodatenzentrum.de/web_public/Datenquellen_TopPlus_Open.pdf">Datenquellen</a>';

      const grayscale = L.tileLayer(mbUrl, {
          id: "mapbox/light-v9",
          tileSize: 512,
          zoomOffset: -1,
          attribution: mbAttr,
        }),
        streets = L.tileLayer(mbUrl, {
          id: "mapbox/streets-v11",
          tileSize: 512,
          zoomOffset: -1,
          attribution: mbAttr,
        }),
        relief = L.tileLayer(rUrl, {
          maxZoom: 17,
          attribution:
            'Map data: &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, <a href="http://viewfinderpanoramas.org">SRTM</a> | Map style: &copy; <a href="https://opentopomap.org">OpenTopoMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)',
        }),
        topplus = L.tileLayer.wms(topplusUrl, {
          layers: "web",
          format: "image/png",
          transparent: true,
          attribution: topplusAttr,
        });

      // Definition of the map, and setup of details : coordinates of the center, zoom level and layers that appear on loading
      this.map = L.map("map", {
        center: [49.99, 8.24],
        zoom: 6,
        minZoom: 2,
        fullscreenControl: true,
        layers: [streets],
        preferCanvas: true,
      });

      this.map.attributionControl.setPrefix(false);

      // Definition of layers
      const baseLayers = {
        Grayscale: grayscale,
        Streets: streets,
        Relief: relief,
        Topplus: topplus,
      };

      // Add layers and points layers to the map
      L.control.layers(baseLayers).addTo(this.map);

      this.unsubscribe.push(
        ...[
          subscribeFeatureVisibiltyChange(
            this.onFeatureVisibilityChange.bind(this)
          ),
          subscribeFeatureClick(this.onFeatureClick.bind(this)),
          subscribeFeatureDoubleClick(this.displayFeature.bind(this)),
        ]
      );
    },
    async displayFeature(featureId, catalogId) {
      const res = await getFeature(featureId, catalogId);
      this.feature = {
        id: featureId,
        items: res
          .filter((x) => x.metadatas?.key && x.metadatas?.value)
          .map(({ metadatas: { key, value } }) => ({
            key,
            value,
          })),
      };

      this.pointcloudUrl = res.filter((x) => x.pointcloudUrl)[0]?.pointcloudUrl;
    },
    closeFeature() {
      this.feature = null;
      this.pointcloudUrl = null;
    },

    createMapObj(feature) {
      let mapObj;

      if (feature.wkt.type === "LINESTRING") {
        mapObj = new L.Polyline(
          feature.wkt.geo.map(([lng, lat]) => new L.LatLng(lat, lng)),
          {
            color: "blue",
            weight: 3,
            opacity: 0.5,
            smoothFactor: 1,
          }
        );
      } else if (feature.wkt.type === "MULTILINESTRING") {
        mapObj = new L.Polyline(
          feature.wkt.geo.map((x) =>
            x.map(([lng, lat]) => new L.LatLng(lat, lng))
          ),
          {
            color: "blue",
            weight: 3,
            opacity: 0.5,
            smoothFactor: 1,
          }
        );
      } else if (feature.wkt.type === "POLYGON") {
        mapObj = new L.Polygon(
          feature.wkt.geo.map(([lng, lat]) => new L.LatLng(lat, lng)),
          {
            color: "blue",
            weight: 3,
            opacity: 0.5,
            smoothFactor: 1,
          }
        );
      } else if (feature.wkt.type === "POINT") {
        // mapObj = new L.marker(
        //   new L.LatLng(feature.wkt.geo[1], feature.wkt.geo[0])
        // );
        mapObj = L.circleMarker(
          new L.LatLng(feature.wkt.geo[1], feature.wkt.geo[0]),
          {
            color: "#3388ff",
          }
        );
      }

      if (mapObj) {
        mapObj.on("click", (event) => {
          this.displayFeature(
            event.target.spalodId,
            event.target.spalodCatalogId
          );
        });
        mapObj.spalodId = feature.id;
        mapObj.spalodCatalogId = feature.catalogId;
        mapObj.visible = false;

        this.mapObjList[mapObj.spalodId] = mapObj;
        return mapObj;
      } else {
        throw new Error("Unkown feature type");
      }
    },

    onFeatureVisibilityChange(features, remove, zoomObjs = []) {
      if (features.length > 0) {
        features
          .filter(({ visible }) => visible)
          .forEach((feature) => {
            if (!this.featuresToDisplay[feature.id]) {
              this.featuresToDisplay[feature.id] = feature;
              this.total += 1;
            } else {
              delete this.featuresToDisplay[feature.id];
              this.total -= 1;
            }
          });

        features
          .filter(({ visible }) => !visible)
          .forEach((feature) => {
            if (this.featuresToDisplay[feature.id]) {
              delete this.featuresToDisplay[feature.id];
              this.total -= 1;
            } else {
              this.featuresToDisplay[feature.id] = feature;
              this.total += 1;
            }
          });
      }

      const chunkSize = 100;
      const chunkFeatures = Object.keys(this.featuresToDisplay)
        .map((x) => this.featuresToDisplay[x])
        .slice(0, chunkSize);

      if (chunkFeatures.length === 0) {
        if (this.actual >= this.total) {
          this.actual = 0;
          this.total = 0;
        }

        if (zoomObjs.length > 0) {
          this.map.options.maxZoom = 17;
          this.fitBounds(zoomObjs.map((x) => this.getObjBounds(x)).flat(1));
        }

        return;
      }

      for (const feature of chunkFeatures) {
        this.actual += 1;
        let mapObj = this.mapObjList[feature.id];

        if (!mapObj && feature.wkt) {
          mapObj = this.createMapObj(feature);
        }

        if (mapObj) {
          if (feature.visible && !remove) {
            mapObj.addTo(this.map);
            mapObj.visible = true;
            zoomObjs.push(mapObj);
          } else {
            mapObj.removeFrom(this.map);
            mapObj.visible = false;

            if (remove) {
              delete this.mapObjList[feature.id];
            }
          }
        }

        delete this.featuresToDisplay[feature.id];
      }

      setTimeout(() => {
        this.onFeatureVisibilityChange([], remove, zoomObjs);
      }, 10);
    },

    getObjBounds(obj) {
      if (typeof obj.getBounds === "function") {
        return obj.getBounds();
      } else if (typeof obj.getLatLng === "function") {
        const latLngs = [obj.getLatLng()];
        const markerBounds = L.latLngBounds(latLngs);
        return markerBounds;
      }
    },

    onFeatureClick(featureId) {
      const mapObj = this.mapObjList[featureId];

      this.fitBounds(this.getObjBounds(mapObj));
    },

    fitBounds(bounds) {
      const previousZoom = this.map.options.maxZoom;
      this.map.options.maxZoom = 15;
      this.map.fitBounds(bounds);
      this.map.options.maxZoom = previousZoom;
    },

    stopPropagation(event) {
      event.stopPropagation();
    },

    getProgressWidth() {
      return Math.floor((this.actual / this.total) * 100) + "%";
    },
    displayLastPortion(item) {
      return item.replace(/.*\//, "").replace(/.*#/, "");
    },

    setItemInput(item) {
      item.hasFocus = true;
      this.inputMinWidth = document.getElementById(
        `div-${item.key}`
      ).offsetWidth;
      setTimeout(() => {
        document.getElementById(`btn-${item.key}`).focus();
      });
    },

    getInputStyle() {
      return {
        "min-width": this.inputMinWidth + "px",
      };
    },

    changeObject(item) {
      updateFeature(this.feature.id, item.key, item.value, item.new);
      item.new = false;
    },

    showAddLabel() {
      return !this.feature?.items?.some(
        (x) =>
          x.key === "rdfs:label" ||
          x.key === "http://www.w3.org/2000/01/rdf-schema#label"
      );
    },

    addLabel() {
      this.feature.items.push({
        key: "http://www.w3.org/2000/01/rdf-schema#label",
        value: "",
        new: true,
      });
    },
  },
  mounted() {
    this.initMap();
  },
  unmounted() {
    this.unsubscribe.forEach((x) => x());
    // clearFeatures();
  },
};
</script>
