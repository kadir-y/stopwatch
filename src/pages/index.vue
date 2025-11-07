<template>
  <v-container class="text-center">
    <v-row justify="center">
        <v-col cols="11" md="6" xl="5">
          <div class="py-10 display">
            <div class="text-h3 pb-2">{{ displayOverallTime }}</div>
            <div class="text-medium-emphasis text-h4">{{ laps.length > 1 ? displayLapTime : '00 : 00 .00' }}</div>
          </div>
          <div class="d-flex justify-space-evenly pb-12">
            <template v-if="!stopped && !isStarted">
              <v-btn class="me-2 rounded-pill" size="x-large" @click="startHandler" color="deep-purple-darken-3">Start</v-btn>
              <v-btn class="rounded-pill" size="x-large"  disabled>Lap</v-btn>
            </template>

            <template v-if="!stopped && isStarted">
              <v-btn class="rounded-pill" size="x-large" color="red-accent-4" @click="stopHandler">Stop</v-btn>
              <v-btn class="rounded-pill" size="x-large" @click="lapHandler">Lap</v-btn>
            </template>

            <template v-if="stopped">
              <v-btn class="rounded-pill" size="x-large" @click="resumeHandler" color="blue-grey-darken-1">Resume</v-btn>
              <v-btn class="rounded-pill" size="x-large" @click="resetHandler" variant="outlined" color="red-darken-4">Reset</v-btn>
            </template>
          </div>
        </v-col>
        <v-col col="11" md="6" xl="5">
          <v-data-table-virtual
            :headers="table.headers"
            :items="displayLaps"
            no-data-text="No laps to show"
            item-value="index"
            class="stopwatch-table rounded-lg"
          >
            <template v-slot:item.index="{ item }">
              <span :class="'text-' + item.color + ' font-weight-bold'">{{ item.index }}</span>
            </template>
            <template v-slot:item.lapTime="{ item }">
              <span class="text-medium-emphasis">{{ item.lapTime }}</span>
            </template>
            <template v-slot:item.overallTime="{ item }">
              <span class="text-high-emphasis">{{ item.overallTime }}</span>
            </template>
          </v-data-table-virtual>
        </v-col>
    </v-row>
  </v-container>
</template>

<script>
import dateParser from '../libs/dateParser'
export default {
  name: '',
  data () {
    return {
      displayOverallTime: '00 : 00 .00',
      displayLapTime: '00 : 00 .00',
      overallTime: 0,
      isStarted: false,
      stopped: false,
      table: {
        headers: [
          { title: "Lap", align: "center", key: "index" },
          { title: "Lap times", align: "center", key: "lapTime" },
          { title: "Overall time", align: "center", key: "overallTime" }
        ]
      },
      laps: [
      /*
        {
          index: ...,
          lapTime: ...,
          overallTime: ...,
          logs: [
            {
              startingTime: ...,
              endingTime: ...,
            }
          ]
        }
      */
      ]
    }
  },
  methods: {
    fitToTime (time) {
      const parser = new dateParser()
      const { hours, minutes, seconds, miliseconds } = parser.mixed(time)
      return  (hours === 0 ? '' : (hours < 10 ? '0' + hours.toString() : hours.toString()) + ' : ') +
        (minutes < 10 ? '0' + minutes.toString() : minutes) +
        (' : ' + (seconds < 10 ? '0' + seconds.toString() : seconds)) +
        (' .' + (miliseconds < 100 ? '0' + miliseconds.toString().slice(0, 1) : miliseconds.toString().slice(0, 2)))
      
    },
    displayStopwatch () {
      const lastLap = this.laps.at(-1)
      const lastLog = lastLap.logs.at(-1)
      const lastLogElapsedTime = Date.now() - lastLog.startTime
      this.displayLapTime = this.fitToTime(lastLap.lapTime + lastLogElapsedTime)
      this.displayOverallTime = this.fitToTime(this.overallTime + lastLogElapsedTime)
    },
    runStopwatchInterval () {
      this.stopwatchInterval = setInterval(() => {
        this.displayStopwatch()
      }, 50)
    },
    pauseStopwatchInterval () {
      clearInterval(this.stopwatchInterval)
    },
    // handlers
    startHandler () {
      this.addLap()
      this.runStopwatchInterval()
      this.isStarted = true

      this.saveToHistory()
    },
    stopHandler () {
      this.stopped = true

      this.endTheLog()
      this.pauseStopwatchInterval()

      this.saveToHistory()
    },
    resetHandler () {
      this.stopped = false
      this.isStarted = false

      this.laps = []
      this.overallTime = 0

      this.displayOverallTime = this.fitToTime(0)
      this.displayLapTime= this.fitToTime(0)

      this.saveToHistory()
    },
    resumeHandler () {
      this.stopped = false

      this.addLog()
      this.runStopwatchInterval()

      this.saveToHistory()
    },
    lapHandler () {
      this.endTheLog()
      this.finishToLap()
      this.addLap()

      this.saveToHistory()
    },   
    addLap () {
      const index = this.laps.length + 1 
      this.laps.push({ 
        index: index >= 10 ? index : `0${index}`,  
        logs: [{ startTime: Date.now() }],
        lapTime: 0
      })
    },
    finishToLap () {
      this.laps.at(-1).overallTime = this.overallTime
    },
    addLog () {
      this.laps.at(-1).logs.push({ 
        startTime: Date.now(),
      })
    },
    endTheLog () { 
      this.laps.at(-1).logs.at(-1).endTime = Date.now()

      // calculating elapsed time
      const lastLap = this.laps.at(-1)
      const lastLog = lastLap.logs.at(-1)
      lastLap.lapTime += lastLog.endTime - lastLog.startTime
      this.overallTime += lastLog.endTime - lastLog.startTime
    },
    saveToHistory () {
      if (this.laps.length >= 50) this.laps.shift() 
      
      const unsavedHistory = {
        overallTime: this.overallTime,
        displayOverallTime: this.displayOverallTime,
        displayLapTime: this.displayLapTime,
        isStarted: this.isStarted,
        stopped: this.stopped,
        laps: this.laps
      }
      localStorage.setItem("stopwatchHistory", JSON.stringify(unsavedHistory))
    },
    loadFromHistory () {
      // check history and create if not created 
      const historyJSON = localStorage.getItem("stopwatchHistory")
      if (historyJSON) {
        // main load process
        const parsedHistory = JSON.parse(historyJSON) 
        this.overallTime = parsedHistory.overallTime || 0
        this.displayOverallTime = parsedHistory.displayOverallTime || this.fitToTime(0)
        this.displayLapTime = parsedHistory.displayLapTime || this.fitToTime(0)
        this.isStarted = parsedHistory.isStarted || false
        this.stopped = parsedHistory.stopped || false
        this.laps = parsedHistory.laps || []
        if (!parsedHistory.stopped && parsedHistory.isStarted) {
          this.runStopwatchInterval()
        } else {
          // if you use after mounted the page this line can be usefull, otherwise not necessary at mounted
          this.pauseStopwatchInterval()
        }
      }
    }
  },
  computed: {
    displayLaps () {
      let lastLap;
      return this.laps
      .slice(0, -1)
      .map(lap => {
        const lapObj = {
          lapTime: this.fitToTime(lap.lapTime),
          overallTime: this.fitToTime(lap.overallTime),
          index: lap.index,
        }
        const difference = lastLap ? lap.lapTime - lastLap.lapTime : 0
        const toleranceValue = (lap.lapTime + (lastLap ? lastLap.lapTime : 0)) / 10
        if (Math.abs(difference) > toleranceValue)
        lapObj.color = difference < 0 ? "deep-purple-darken-3" : "deep-orange-accent-4" 
        lastLap = lap
        return lapObj
      })      
      .reverse()
    }
  },
  mounted () {
    this.loadFromHistory();
  }
}
</script>

<style lang="scss">
  @media (max-width: 960px) {
    .stopwatch-table {
      height:calc(100vh - 28.875rem)
    }
  }
  @media (min-width: 961px) {
    .display {  
      margin-top: 5rem;
    }
    .stopwatch-table {
      height:calc(100vh - 10.5rem)
    }
  }
</style>