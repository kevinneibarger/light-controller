<script setup lang="ts">
import { ref, onMounted } from 'vue'
import axios from 'axios'

// lights cycle through colors in order: Green -> Yellow -> Red -> Green
const trafficLightSequence = ['green', 'yellow', 'red', 'green']

// We have two roads at an intersection, with four traffic lights
// So, we have active lights on 4 sides, 2 for each road
const activeLightRoad1TL1 = ref<string>('green')
const activeLightRoad1TL2 = ref<string>('red')
const activeLightRoad2TL1 = ref<string>('green')
const activeLightRoad2TL2 = ref<string>('red')


// This const will handle each active light change by calling the Spring boot rest api and setting to the current road and light
const handleActiveLightChange = (road : number, trafficLight : string) => {
  axios.post('http://localhost:8080/intersections', { road, activeLight: activeLightRoad1TL1.value })
    .then(console.log)
    .catch(console.error)
}

// This const will iterate through the light cycle for each interval and set the light value to the next color in the sequence
const cycleTrafficLight = (lightRef: typeof activeLightRoad1TL1, road: number, interval: number) => {
  let index = 0
  setInterval(() => {
    index = (index + 1) % trafficLightSequence.length
    lightRef.value = trafficLightSequence[index]
    handleActiveLightChange(road, lightRef.value)
  }, interval)
}

onMounted(() => {
  cycleTrafficLight(activeLightRoad1TL1, 1, 3000) // Road 1 Traffic Light 1 will change every 5 seconds
  cycleTrafficLight(activeLightRoad1TL2, 1, 5000)
  cycleTrafficLight(activeLightRoad2TL1, 2, 3000)
  cycleTrafficLight(activeLightRoad2TL2, 2, 5000)
})

</script>

<template>
  <header>
    <div class="wrapper">
      <h1>Intersection Light Controller</h1>
    </div>
  </header>

    <main>

      <div class="diamond-grid">
        <!-- Top Row: Road 1 -->
        <div class="light-controller north">
          <p>Traffic Light 1 - Road 1: {{ activeLightRoad1TL1 }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightRoad1TL1" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightRoad1TL1" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightRoad1TL1" /> Red</label>
          </div>
        </div>

        <div class="light-controller west">
          <p>Traffic Light 2 - Road 1: {{ activeLightRoad1TL2 }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightRoad1TL2" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightRoad1TL2" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightRoad1TL2" /> Red</label>
          </div>
        </div>

        <!-- Bottom Row: Road 2 -->
        <div class="light-controller east">
          <p>Traffic Light 1 - Road 2: {{ activeLightRoad2TL1 }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightRoad2TL1" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightRoad2TL1" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightRoad2TL1" /> Red</label>
          </div>
        </div>

        <div class="light-controller south">
          <p>Traffic Light 2 - Road 2: {{ activeLightRoad2TL2 }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightRoad2TL2" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightRoad2TL2" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightRoad2TL2" /> Red</label>
          </div>
        </div>
      </div>
    </main>


<!--  <main>-->
<!--    <div class="light-controller">-->
<!--      <p>Active light Road 1: {{ activeLightRoad1 }}</p>-->
<!--      <div class="light">-->
<!--        <label>-->
<!--          <input type="radio" value="red" class="red" v-model="activeLightRoad1" name="light"-->
<!--            @change="handleActiveLightChange" /> Red-->
<!--        </label>-->
<!--        <label>-->
<!--          <input type="radio" value="yellow" class="yellow" v-model="activeLightRoad1" name="light"-->
<!--            @change="handleActiveLightChange" /> Yellow-->
<!--        </label>-->
<!--        <label>-->
<!--          <input type="radio" value="green" class="green" v-model="activeLightRoad1" name="light"-->
<!--            @change="handleActiveLightChange" /> Green-->
<!--        </label>-->
<!--      </div>-->
<!--    </div>-->
<!--    <div class="light-controller">-->
<!--      <p>Active light Road 2: {{ activeLightRoad2 }}</p>-->
<!--      <div class="light">-->
<!--        <label>-->
<!--          <input type="radio" value="red" class="red" v-model="activeLightRoad2" name="light"-->
<!--                 @change="handleActiveLightChange" /> Red-->
<!--        </label>-->
<!--        <label>-->
<!--          <input type="radio" value="yellow" class="yellow" v-model="activeLightRoad2" name="light"-->
<!--                 @change="handleActiveLightChange" /> Yellow-->
<!--        </label>-->
<!--        <label>-->
<!--          <input type="radio" value="green" class="green" v-model="activeLightRoad2" name="light"-->
<!--                 @change="handleActiveLightChange" /> Green-->
<!--        </label>-->
<!--      </div>-->
<!--    </div>-->
<!--  </main>-->
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    margin: calc(var(--section-gap) / 4);
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}

/*.intersection-grid {
//  display: grid;
//  grid-template-columns: repeat(2, 1fr);
//  grid-template-rows: repeat(2, auto);
//  gap: 2rem;
//  margin-top: 2rem;
}*/

.diamond-grid {
  display: grid;
  grid-template-areas:
    ". north ."
    "west center east"
    ". south .";
  grid-template-columns: 1fr 1fr 1fr;
  grid-template-rows: auto auto auto;
  gap: 2rem;
  justify-items: center;
  align-items: center;
  margin-top: 2rem;
}

.north {
  grid-area: north;
}
.south {
  grid-area: south;
}
.east {
  grid-area: east;
}
.west {
  grid-area: west;
}
.intersection-label {
  grid-area: center;
  font-size: 2rem;
  text-align: center;
}

.light-controller {
  display: grid;
  place-items: center;
  gap: 1rem;

  .light {
    display: grid;
    gap: .5rem;
  }

}

input[type='radio'].red {
  accent-color: #cc3232;
}

input[type='radio'].yellow {
  accent-color: #e7b416;
}

input[type='radio'].green {
  accent-color: #2dc937;
}
</style>
