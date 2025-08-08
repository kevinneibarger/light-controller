<script setup lang="ts">
import { ref, onMounted, Ref } from 'vue'
import axios from 'axios'

type RoadDirection = 'north' | 'south' | 'east' | 'west'

interface TrafficLightController {
  directions: RoadDirection[]
  roadNum: number
  trafficLights: Record<RoadDirection, Ref<string>>
  startTrafficCycle: () => void
  stopTrafficCycle: () => void
  isTrafficCycleRunning: ref<boolean>
}

const createTrafficLightController = (
  roadNum: number,
  directions: RoadDirection[]
): TrafficLightController => {
  const trafficLights: Record<RoadDirection, Ref<string>> = {
    north: ref('red'),
    south: ref('red'),
    east: ref('green'),
    west: ref('green')
  }

  const isTrafficCycleRunning = ref(false)
  let timeoutId: number | null = null
  let activeRoad: 'NS' | 'EW' = 'NS'

  // This const controls the state of the light's color. Kinda like useState in React.js
  const trafficLightState = (direction: RoadDirection, color: string) => {
    axios.post('http://localhost:8080/intersections', {
      road: roadNum,
      direction,
      activeLight: color
    }).catch(console.error)
  }

  const cycle = () => {
    // Execute the cycle if the "running" flag is true
    if (!isTrafficCycleRunning.value) return

    const nsDirections: RoadDirection[] = ['north', 'south']
    const ewDirections: RoadDirection[] = ['east', 'west']
    const activeDirections = activeRoad === 'NS' ? nsDirections : ewDirections
    const inactiveDirections = activeRoad === 'NS' ? ewDirections : nsDirections

    activeDirections.forEach(dir => {
      trafficLights[dir].value = 'green'
      trafficLightState(dir, 'green')
    })
    inactiveDirections.forEach(dir => {
      trafficLights[dir].value = 'red'
      trafficLightState(dir, 'red')
    })

    timeoutId = window.setTimeout(() => {
      activeDirections.forEach(dir => {
        trafficLights[dir].value = 'yellow'
        trafficLightState(dir, 'yellow')
      })

      timeoutId = window.setTimeout(() => {
        activeDirections.forEach(dir => {
          trafficLights[dir].value = 'red'
          trafficLightState(dir, 'red')
        })

        activeRoad = activeRoad === 'NS' ? 'EW' : 'NS'

        timeoutId = window.setTimeout(cycle, 2000)
      }, 2000)
    }, 3000)
  }

  // This will triggered when the user clicks the button to start the Traffic Light Cycle
  const startTrafficCycle = () => {
    if (!isTrafficCycleRunning.value) {
      isTrafficCycleRunning.value = true
      cycle() // Run the cycle of lights for each traffic light and direction
    }
  }

  const stopTrafficCycle = () => {
    isTrafficCycleRunning.value = false
    if (timeoutId !== null) {
      clearTimeout(timeoutId)
      timeoutId = null
    }

    // When this button is clicked we want to turn off all lights.
    Object.keys(trafficLights).forEach(dir => {
      const direction = dir as RoadDirection
      trafficLights[direction].value = 'off'
      trafficLightState(direction, 'off')
    })
  }

  return {
    directions,
    roadNum,
    trafficLights,
    startTrafficCycle,
    stopTrafficCycle,
    isTrafficCycleRunning
  }
}

const controller = createTrafficLightController(1, ['north', 'south', 'east', 'west'])

const activeLightNorth = controller.trafficLights.north
const activeLightSouth = controller.trafficLights.south
const activeLightEast = controller.trafficLights.east
const activeLightWest = controller.trafficLights.west
const isTrafficCycleRunning = controller.isTrafficCycleRunning

const toggleCycle = () => {
  if (isTrafficCycleRunning.value) {
    controller.stopTrafficCycle()
  } else {
    controller.startTrafficCycle()
  }
}
</script>

<template>
  <header>
    <div class="wrapper">
      <h1>Intersection Light Controller</h1>
    </div>
  </header>

    <main>
      <div>
        <button
          @click="toggleCycle"
          :class="{ stop: isTrafficCycleRunning }">
          {{ isTrafficCycleRunning ? 'Stop' : 'Start' }} Traffic Light Cycle
        </button>
      </div>
      <div class="diamond-grid">
        <!-- Top Row: Road 1 -->
        <div class="light-controller north">
          <p>Northbound Traffic Light: {{ activeLightNorth === 'off' ? 'OFF' : activeLightNorth }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightNorth" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightNorth" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightNorth" /> Red</label>
          </div>
        </div>

        <div class="light-controller west">
          <p>Westbound Traffic Light: {{ activeLightWest === 'off' ? 'OFF' : activeLightWest }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightWest" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightWest" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightWest" /> Red</label>
          </div>
        </div>

        <!-- Bottom Row: Road 2 -->
        <div class="light-controller east">
          <p>Eastbound Traffic Light: {{ activeLightEast === 'off' ? 'OFF' : activeLightEast }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightEast" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightEast" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightEast" /> Red</label>
          </div>
        </div>

        <div class="light-controller south">
          <p>Southbound Traffic Light: {{ activeLightSouth === 'off' ? 'OFF' : activeLightSouth }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightSouth" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightSouth" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightSouth" /> Red</label>
          </div>
        </div>
      </div>
    </main>
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

button {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  font-weight: bold;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background-color: #2dc937; /* green for start */
  color: white;
  transition: background-color 0.3s ease;
  margin: 1rem auto;
  display: block;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

button:hover {
  background-color: #28a745;
}

button.stop {
  background-color: #cc3232; /* red for stop */
}

button.stop:hover {
  background-color: #b02a2a;
}
</style>
