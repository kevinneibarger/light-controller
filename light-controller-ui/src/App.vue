<script setup lang="ts">
import { ref, onMounted, Ref } from 'vue'
import axios from 'axios'

const trafficLightSequence = ['green', 'yellow', 'red', 'green']
type RoadDirection = 'north' | 'south' | 'east' | 'west'

interface TrafficLightController {
  directions: RoadDirection[]
  roadNum: number
  trafficLights: Record<RoadDirection, Ref<string>>
  sequenceIndex: number
  startTrafficCycleNew: () => void
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

  let sequenceIndex = 0

  const trafficLightState = (direction: RoadDirection, color: string) => {
    axios.post('http://localhost:8080/intersections', {
      road: roadNum,
      direction,
      activeLight: color
    })
      .then(console.log)
      .catch(console.error)
  }

  const startTrafficCycleNew = () => {
    let activeRoad: 'NS' | 'EW' = 'NS'
    const nsDirections: RoadDirection[] = ['north', 'south']
    const ewDirections: RoadDirection[] = ['east', 'west']

    // Start the traffic cycle with active direction being North/South and inactive being East/West
    const cycle = () => {
      const activeDirections = activeRoad === 'NS' ? nsDirections : ewDirections
      const inactiveDirections = activeRoad === 'NS' ? ewDirections : nsDirections

      // Set the active direction north/south to green
      activeDirections.forEach(dir => {
        trafficLights[dir].value = 'green'
        trafficLightState(dir, 'green')
      })
      // Set the inactive direction east/west to red
      inactiveDirections.forEach(dir => {
        trafficLights[dir].value = 'red'
        trafficLightState(dir, 'red')
      })

      setTimeout(() => {
        // Step 2: Yellow
        activeDirections.forEach(dir => {
          trafficLights[dir].value = 'yellow'
          trafficLightState(dir, 'yellow')
        })

        // In order to make this look like a real traffic light sequence,
        // we want to make the red transition (where all lights are red) a short 1 second pause
        setTimeout(() => {
          // Step 3: Red for all (brief transition)
          activeDirections.forEach(dir => {
            trafficLights[dir].value = 'red'
            trafficLightState(dir, 'red')
          })

          // Switch active road
          activeRoad = activeRoad === 'NS' ? 'EW' : 'NS'

          // Next cycle
          setTimeout(cycle, 1000) // brief pause before switching
        }, 2000) // yellow duration
      }, 3000) // green duration
    }

    cycle()
  }

  return { directions, roadNum, trafficLights, sequenceIndex, startTrafficCycleNew }
}

const northSouthRoad = createTrafficLightController(1, ['north', 'south'])
const eastWestRoad = createTrafficLightController(2, ['east', 'west'])

onMounted(() => {
  northSouthRoad.startTrafficCycleNew()
  eastWestRoad.startTrafficCycleNew()
})

const activeLightEast = eastWestRoad.trafficLights.east
const activeLightWest = eastWestRoad.trafficLights.west
const activeLightNorth = northSouthRoad.trafficLights.north
const activeLightSouth = northSouthRoad.trafficLights.south
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
          <p>Traffic Light 1 - Road 1: {{ activeLightNorth }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightNorth" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightNorth" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightNorth" /> Red</label>
          </div>
        </div>

        <div class="light-controller west">
          <p>Traffic Light 2 - Road 1: {{ activeLightWest }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightWest" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightWest" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightWest" /> Red</label>
          </div>
        </div>

        <!-- Bottom Row: Road 2 -->
        <div class="light-controller east">
          <p>Traffic Light 1 - Road 2: {{ activeLightEast }}</p>
          <div class="light">
            <label><input type="radio" value="green" class="green" v-model="activeLightEast" /> Green</label>
            <label><input type="radio" value="yellow" class="yellow" v-model="activeLightEast" /> Yellow</label>
            <label><input type="radio" value="red" class="red" v-model="activeLightEast" /> Red</label>
          </div>
        </div>

        <div class="light-controller south">
          <p>Traffic Light 2 - Road 2: {{ activeLightSouth }}</p>
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
