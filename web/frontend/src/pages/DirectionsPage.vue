<template>
  <div class="top-left-card">
    <q-card>
      <search-box
        ref="searchBox"
        hint="From"
        v-model="fromPoi"
        :force-text="fromPoi ? poiDisplayName(fromPoi) : undefined"
        v-on:update:model-value="rewriteUrl"
      ></search-box>
      <search-box
        ref="searchBox"
        hint="To"
        v-model="toPoi"
        :force-text="toPoi ? poiDisplayName(toPoi) : undefined"
        v-on:update:model-value="rewriteUrl"
      ></search-box>
    </q-card>
  </div>
  <div class="bottom-card" ref="bottomCard" v-if="fromPoi && toPoi">
    <q-card>
      <q-card-section class="bg-primary text-white">
        <div class="timeline" ref="timeline" v-on:scroll="scroll">
          <ol>
            <li
              :key="`${item.time}`"
              :class="item.selected ? 'selected' : ''"
              v-for="item in $data.steps"
            >
              <div class="instruction">{{ item.text }}</div>
            </li>
          </ol>
        </div>
        <q-btn
          round
          color="black"
          class="center-left-floating"
          icon="chevron_left"
          v-on:click="scrollLeft"
        />
        <q-btn
          round
          color="black"
          class="center-right-floating"
          icon="chevron_right"
          v-on:click="scrollRight"
        />
      </q-card-section>
    </q-card>
  </div>
</template>

<script lang="ts">
import {
  activeMarkers,
  map,
  setBottomCardAllowance,
} from 'src/components/BaseMap.vue';
import {
  canonicalizePoi,
  decanonicalizePoi,
  POI,
  poiDisplayName,
} from 'src/components/models';
import { defineComponent, Ref, ref } from 'vue';
import SearchBox from 'src/components/SearchBox.vue';
import { decodePath } from 'src/third_party/decodePath';
import { Popup } from 'maplibre-gl';

var toPoi: Ref<POI | undefined> = ref(undefined);
var fromPoi: Ref<POI | undefined> = ref(undefined);

export default defineComponent({
  name: 'DirectionsPage',
  props: {
    mode: String,
    to: String,
    from: String,
  },
  data: function () {
    return {
      steps: [],
      points: [],
      stepCallout: undefined,
      stepCalloutStep: -1,
    };
  },
  components: { SearchBox },
  methods: {
    poiDisplayName,
    scrollLeft() {
      const timeline = this.$refs.timeline as Element;
      if (timeline.scrollWidth - timeline.clientWidth == 0) {
        return;
      }
      const position =
        timeline.scrollLeft / (timeline.scrollWidth - timeline.clientWidth);
      const step = Math.min(
        Math.floor(position * (this.$data.steps.length - 1)),
        this.$data.steps.length - 1
      );
      const newStep = Math.max(Math.ceil(step - 1), 0);
      const newPosition =
        (newStep / (this.$data.steps.length - 1)) *
        (timeline.scrollWidth - timeline.clientWidth);
      timeline.scroll({ behavior: 'smooth', left: newPosition });
    },
    scrollRight() {
      const timeline = this.$refs.timeline as Element;
      if (timeline.scrollWidth - timeline.clientWidth == 0) {
        return;
      }
      const position =
        timeline.scrollLeft / (timeline.scrollWidth - timeline.clientWidth);
      console.log(`position: ${position}`);
      const step = Math.min(
        Math.floor(0.05 + position * (this.$data.steps.length - 1)),
        this.$data.steps.length - 1
      );
      const newStep = Math.min(
        this.$data.steps.length - 1,
        Math.floor(step + 1)
      );
      const newPosition =
        (newStep / (this.$data.steps.length - 1)) *
        (timeline.scrollWidth - timeline.clientWidth);
      timeline.scroll({ behavior: 'smooth', left: newPosition });
    },
    setupPopup() {
      const timeline = this.$refs.timeline as Element;
      if (timeline.scrollWidth - timeline.clientWidth == 0) {
        return;
      }
      const position =
        timeline.scrollLeft / (timeline.scrollWidth - timeline.clientWidth);
      const step = position * (this.$data.steps.length - 1);

      const nearestStepRounded = Math.min(
        Math.round(step),
        this.$data.steps.length - 1
      );

      this.$data.steps.forEach((step, index) => {
        step.selected = index == nearestStepRounded;
      });

      if (
        Math.abs(this.$data.stepCalloutStep - step) <= 0.1 ||
        Math.round(this.$data.stepCalloutStep) == Math.round(step)
      ) {
        return;
      }
      this.$data.stepCalloutStep = step;

      const stepStartPositionIndex =
        this.$data.steps[nearestStepRounded].interval[0];
      const point = this.$data.points[stepStartPositionIndex];
      const beforePoint = this.$data.points[stepStartPositionIndex - 1];
      const afterPoint = this.$data.points[stepStartPositionIndex + 1];

      var neighborAverage = [point[0], point[1]];
      if (afterPoint && beforePoint) {
        neighborAverage = [
          (afterPoint[0] + beforePoint[0]) / 2,
          (afterPoint[1] + beforePoint[1]) / 2,
        ];
      } else if (afterPoint) {
        neighborAverage = afterPoint;
      } else if (beforePoint) {
        neighborAverage = beforePoint;
      }
      var chosenAnchor = 'center';
      if (neighborAverage[0] >= point[0] && neighborAverage[1] >= point[1]) {
        chosenAnchor = 'top-right';
      }
      if (neighborAverage[0] <= point[0] && neighborAverage[1] >= point[1]) {
        chosenAnchor = 'top-left';
      }
      if (neighborAverage[0] >= point[0] && neighborAverage[1] <= point[1]) {
        chosenAnchor = 'bottom-right';
      }
      if (neighborAverage[0] <= point[0] && neighborAverage[1] <= point[1]) {
        chosenAnchor = 'bottom-left';
      }
      setTimeout(() => {
        if (this.$data.stepCallout) {
          this.$data.stepCallout.remove();
        }
        this.$data.stepCallout = new Popup({
          closeOnClick: true,
          anchor: chosenAnchor,
        })
          .setLngLat(this.$data.points[stepStartPositionIndex])
          .setText(this.$data.steps[nearestStepRounded].text)
          .addTo(map);
        this.$data.steps.forEach((step, index) => {
          step.selected = index == nearestStepRounded;
        });
      });
    },
    scroll() {
      setTimeout(() => this.setupPopup());
      const timeline = this.$refs.timeline as Element;
      const position =
        timeline.scrollLeft / (timeline.scrollWidth - timeline.clientWidth);
      const step = Math.min(
        Math.floor(position * (this.$data.steps.length - 1)),
        this.$data.steps.length - 1
      );
      const fraction = position * (this.$data.steps.length - 1) - step;
      const stepStartPositionIndex = this.$data.steps[step].interval[0];
      const stepEndPositionIndex = this.$data.steps[step].interval[1];
      const stepPositionCount = stepEndPositionIndex - stepStartPositionIndex;
      const lerpFraction =
        fraction * stepPositionCount - Math.floor(fraction * stepPositionCount);
      const lerpStart =
        this.$data.points[
          stepStartPositionIndex + Math.floor(fraction * stepPositionCount)
        ];
      const lerpEnd =
        this.$data.points[
          stepStartPositionIndex + Math.ceil(fraction * stepPositionCount)
        ];
      const finalPos: [number, number] = [
        lerpEnd[0] * lerpFraction + lerpStart[0] * (1 - lerpFraction),
        lerpEnd[1] * lerpFraction + lerpStart[1] * (1 - lerpFraction),
      ];
      map?.easeTo({
        center: finalPos,
        duration: 0,
      });
    },
    rewriteUrl: async function () {
      if (!fromPoi.value?.position && !toPoi.value?.position) {
        this.$router.push('/');
        return;
      }
      const fromCanonical = fromPoi.value
        ? canonicalizePoi(fromPoi.value)
        : '_';
      const toCanonical = toPoi.value ? canonicalizePoi(toPoi.value) : '_';
      this.$router.push(
        `/directions/${this.mode}/${toCanonical}/${fromCanonical}`
      );
      if (fromPoi.value?.position && toPoi.value?.position) {
        const response = await fetch(
          `/graphhopper/route?point=${fromPoi.value?.position.lat},${fromPoi.value?.position.long}&point=${toPoi.value?.position.lat},${toPoi.value?.position.long}&profile=${this.$props.mode}`
        );
        const route = await response.json();
        const path = route?.paths[0];
        if (path && map) {
          var totalTime = 0;
          for (const key in path.instructions) {
            totalTime += path.instructions[key].time;
            path.instructions[key].time = totalTime;
          }
          this.$data.steps = path.instructions;
          console.log(this.$data.steps);
          if (map?.getLayer('headway_polyline')) {
            map?.removeLayer('headway_polyline');
          }
          if (map?.getSource('headway_polyline')) {
            map?.removeSource('headway_polyline');
          }
          // min/max
          const bbox: number[] = [1000, 1000, -1000, -1000];
          var points: number[][] = [];
          decodePath(path.points, false).forEach((point) => {
            points.push(point);
            if (point[0] < bbox[0]) {
              bbox[0] = point[0];
            }
            if (point[1] < bbox[1]) {
              bbox[1] = point[1];
            }
            if (point[2] > bbox[2]) {
              bbox[2] = point[2];
            }
            if (point[3] > bbox[3]) {
              bbox[3] = point[3];
            }
          });
          console.log(points);
          this.$data.points = points;
          map?.addSource('headway_polyline', {
            type: 'geojson',
            data: {
              type: 'Feature',
              properties: {},
              geometry: {
                type: 'LineString',
                coordinates: points,
              },
            },
          });
          map?.addLayer({
            id: 'headway_polyline',
            type: 'line',
            source: 'headway_polyline',
            layout: {
              'line-join': 'round',
              'line-cap': 'round',
            },
            paint: {
              'line-color': '#1976D2',
              'line-width': 6,
            },
          });
          map?.flyTo({
            center: points[0] as [number, number],
            zoom: 16,
          });
          this.setupPopup();
        }
      } else {
        if (map?.getLayer('headway_polyline')) {
          map?.removeLayer('headway_polyline');
        }
        if (map?.getSource('headway_polyline')) {
          map?.removeSource('headway_polyline');
        }
        if (this.$data.stepCallout) {
          this.$data.stepCallout.remove();
          this.$data.stepCallout = undefined;
        }
      }
    },
    resizeMap() {
      if (this.$refs.bottomCard && this.$refs.bottomCard) {
        setBottomCardAllowance(this.$refs.bottomCard.offsetHeight);
      } else {
        setBottomCardAllowance(0);
      }
    },
  },
  watch: {
    to: {
      immediate: true,
      deep: true,
      handler(newValue) {
        setTimeout(async () => {
          toPoi.value = await decanonicalizePoi(newValue);
          await this.rewriteUrl();
          this.resizeMap();
        });
      },
    },
    from: {
      immediate: true,
      deep: true,
      handler(newValue) {
        setTimeout(async () => {
          fromPoi.value = await decanonicalizePoi(newValue);
          await this.rewriteUrl();
          this.resizeMap();
        });
      },
    },
  },
  mounted: async function () {
    setTimeout(async () => {
      toPoi.value = await decanonicalizePoi(this.$props.to as string);
      fromPoi.value = await decanonicalizePoi(this.$props.from as string);
      await this.rewriteUrl();
      this.resizeMap();
    });
  },
  unmounted: function () {
    activeMarkers.forEach((marker) => marker.remove());
    activeMarkers.length = 0;
    if (map?.getLayer('headway_polyline')) {
      map?.removeLayer('headway_polyline');
    }
    if (map?.getSource('headway_polyline')) {
      map?.removeSource('headway_polyline');
    }
    if (this.$data.stepCallout) {
      this.$data.stepCallout.remove();
      this.$data.stepCallout = undefined;
    }
  },
  setup: function () {
    return {
      toPoi,
      fromPoi,
    };
  },
});
</script>
