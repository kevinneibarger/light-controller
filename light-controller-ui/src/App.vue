<script setup lang="ts">
import { ref, Ref } from 'vue'
import '@/assets/traffic-lights.css'
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
