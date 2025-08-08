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
    north: ref('off'),
    south: ref('off'),
    east: ref('off'),
    west: ref('off')
  }

  const isTrafficCycleRunning = ref(false)
  let timeoutId: number | null = null // Holds the timeout internal so it can be canceled in the case all lights are red.
  let activeRoad: 'NS' | 'EW' = 'NS'

  // This const controls the state of the light's color. Kinda like useState in React.js
  const trafficLightState = (direction: RoadDirection, color: string) => {
    axios.post('http://localhost:8080/intersections', {
      road: roadNum,
      direction,
      activeLight: color
    }).catch(console.error)
  }

  // Define the traffic cycle of green > yellow > red > ... for the appropriate direction north/south or east/west
  const cycle = () => {
    // Execute the cycle if the "running" flag is true
    if (!isTrafficCycleRunning.value) return

    const nsDirections: RoadDirection[] = ['north', 'south']
    const ewDirections: RoadDirection[] = ['east', 'west']
    const activeDirections = activeRoad === 'NS' ? nsDirections : ewDirections
    const inactiveDirections = activeRoad === 'NS' ? ewDirections : nsDirections

    // Set the light state as green for the active road, either north/south or east/west
    activeDirections.forEach(dir => {
      trafficLights[dir].value = 'green'
      trafficLightState(dir, 'green')
    })

    // Set the light state as red for the inactive road, either north/south or east/west
    inactiveDirections.forEach(dir => {
      trafficLights[dir].value = 'red'
      trafficLightState(dir, 'red')
    })

    // This logic sets the duration that the light is yellow and red
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

        timeoutId = window.setTimeout(cycle, 1000)
      }, 1000)
    }, 3000)
  }

  // This will triggered when the user clicks the button to start the Traffic Light Cycle
  const startTrafficCycle = () => {
    if (!isTrafficCycleRunning.value) {
      isTrafficCycleRunning.value = true
      cycle() // Run the cycle of lights for each traffic light and direction
    }
  }

  // This trigger will stop the traffic signal cycle and shut all lights off
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

// New controller which holds the state of each light and determines if the cycle is running
const controller = createTrafficLightController(1, ['north', 'south', 'east', 'west'])
// Expose the variables for states of light color and cycle
const activeLightNorth = controller.trafficLights.north
const activeLightSouth = controller.trafficLights.south
const activeLightEast = controller.trafficLights.east
const activeLightWest = controller.trafficLights.west
const isTrafficCycleRunning = controller.isTrafficCycleRunning

// When the ACTIVATE or DEACTIVE Traffic Lights button is clicked, perform the appropriate action
const toggleCycle = () => {
  if (isTrafficCycleRunning.value) {
    controller.stopTrafficCycle()
  } else {
    controller.startTrafficCycle()
  }
}

// The follow logic tests each light individually and manually, one color at a time (click)
const selectedDirection = ref<RoadDirection | null>(null)
const lightCycle = ['green', 'yellow', 'red', 'off'] as const
type LightColor = typeof lightCycle[number]

const testSelectedLight = () => {
  if (!selectedDirection.value) return

  const dir = selectedDirection.value
  const current = controller.trafficLights[dir].value as LightColor

  const currentIndex = lightCycle.indexOf(current)
  const nextIndex = (currentIndex + 1) % lightCycle.length
  const next = lightCycle[nextIndex]

  controller.trafficLights[dir].value = next

  axios.post('http://localhost:8080/intersections', {
    road: controller.roadNum,
    direction: dir,
    activeLight: next
  }).catch(console.error)
}

// Logic to test the sequence function of each Traffic light selected by a radio button
const selectedDirection1 = ref<RoadDirection | null>(null)
const isTesting = ref(false)
const individualTimeoutId = ref<number | null>(null)
type LightState = 'green' | 'yellow' | 'red' | 'off'
const lightStates: LightState[] = ['green', 'yellow', 'red', 'off']
let currentStateIndex = 0

const testSelectedLightSequence = () => {
  if (!selectedDirection1.value) return

  if (isTesting.value) {
    // Stop the test
    isTesting.value = false
    clearTimeout(individualTimeoutId.value!)
    individualTimeoutId.value = null
    controller.trafficLights[selectedDirection1.value].value = 'off'
    sendLightState(selectedDirection1.value, 'off')
    return
  }

  // Start the test
  isTesting.value = true
  currentStateIndex = 0
  cycleLight()
}

function cycleLight() {
  if (!isTesting.value || !selectedDirection1.value) return

  const dir = selectedDirection1.value
  const state = lightStates[currentStateIndex]

  controller.trafficLights[dir].value = state
  sendLightState(dir, state)

  // Advance to next state
  currentStateIndex = (currentStateIndex + 1) % lightStates.length

  const delay = state === 'green' ? 3000 : 1000

  individualTimeoutId.value = window.setTimeout(() => {
    cycleLight()
  }, delay)
}

const sendLightState = (dir: RoadDirection, color: string) => {
  axios.post('http://localhost:8080/intersections', {
    road: controller.roadNum,
    direction: dir,
    activeLight: color
  }).catch(console.error)
}


</script>

<template>
  <header>
    <div class="wrapper">
      <h1>Intersection Light Controller</h1>
    </div>
  </header>

    <main>
      <div class="intersection-layout">
      <div class="controller-panel">
        <button @click="toggleCycle" :class="{ stop: isTrafficCycleRunning }">
          {{ isTrafficCycleRunning ? 'DEACTIVATE' : 'ACTIVATE' }} Traffic Lights
        </button>

        <div class="test-light-panel">
          <button @click="testSelectedLight" :disabled="!selectedDirection">
            Test Light Functionality (Manual)
          </button>
            <div class="direction-selector">
              <label><input type="radio" value="north" v-model="selectedDirection" /> North</label>
              <label><input type="radio" value="south" v-model="selectedDirection" /> South</label>
              <label><input type="radio" value="east" v-model="selectedDirection" /> East</label>
              <label><input type="radio" value="west" v-model="selectedDirection" /> West</label>
            </div>
        </div>

        <div class="test-light-panel">
          <button
            @click="testSelectedLightSequence" :class="{ stop: isTesting }"
            :disabled="!selectedDirection1">{{ isTesting ? 'Stop Test' : 'Test Light Functionality (Cycle)' }}
          </button>
          <div class="direction-selector">
            <label><input type="radio" value="north" v-model="selectedDirection1" /> North</label>
            <label><input type="radio" value="south" v-model="selectedDirection1" /> South</label>
            <label><input type="radio" value="east" v-model="selectedDirection1" /> East</label>
            <label><input type="radio" value="west" v-model="selectedDirection1" /> West</label>
          </div>
        </div>
      </div>

      <div class="diamond-grid">
        <!-- Top Row: Road 1 -->
        <div class="light-controller north">
          <p>Northbound Traffic Light</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightNorth" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightNorth" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightNorth" /> Red</label>
          </div>
        </div>

        <div class="light-controller west">
          <p>Westbound Traffic Light</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightWest" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightWest" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightWest" /> Red</label>
          </div>
        </div>

        <!-- Bottom Row: Road 2 -->
        <div class="light-controller east">
          <p>Eastbound Traffic Light</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightEast" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightEast" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightEast" /> Red</label>
          </div>
        </div>

        <div class="light-controller south">
          <p>Southbound Traffic Light</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightSouth" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightSouth" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightSouth" /> Red</label>
          </div>
        </div>
      </div>
      </div>
    </main>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.intersection-layout {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 2rem;
  padding: 1rem;
  flex-wrap: wrap; /* optional: allows wrapping on smaller screens */
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
  transform: scale(2.5); /* Increase size by 1.5x */
  margin-right: 8px;     /* Add spacing between radio and label text */
}

input[type='radio'].yellow {
  accent-color: #e7b416;
  transform: scale(2.5); /* Increase size by 1.5x */
  margin-right: 8px;     /* Add spacing between radio and label text */
}

input[type='radio'].green {
  accent-color: #2dc937;
  transform: scale(2.5); /* Increase size by 1.5x */
  margin-right: 8px;     /* Add spacing between radio and label text */
}

.traffic-radio input[type="radio"] {
  appearance: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: 2px solid #333;
  margin-right: 8px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.traffic-radio input[type="radio"]:checked {
  background-color: green; /* You can dynamically change this based on direction or state */
}

.traffic-radio input[type="radio"]:hover {
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
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

.controller-panel {
  flex: 0 0 300px; /* fixed width */
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  border: 2px solid #ccc;
  padding: 1rem;
  border-radius: 12px;
  background-color: #f9f9f9;
  box-shadow: 0 0 8px rgba(0, 0, 0, 0.1);
}

.light-buttons button {
  padding: 0.5rem 1rem;
  font-weight: bold;
  border: none;
  border-radius: 6px;
  background-color: #444;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.light-buttons button:hover {
  background-color: #666;
}

.test-light-panel {
  border: 2px dashed #aaa;
  padding: 1rem;
  border-radius: 10px;
  margin-top: 2rem;
  text-align: center;
  background-color: #f0f0f0;
}

.direction-selector {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.direction-selector label {
  font-weight: bold;
  cursor: pointer;
  color: #181818;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

</style>
