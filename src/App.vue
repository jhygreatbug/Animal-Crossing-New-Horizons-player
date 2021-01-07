<template>
  <div class="wrap">
    <template v-if="status === 'ready'">
      <my-audio
        ref="audio"
        :base-path="audioStatus.basePath"
        :hours="audioStatus.hours"
        :weather="audioStatus.weather"
        @ready="handleAudioReady"
      ></my-audio>
      <br />
      <select
        name="hours"
        v-model="audioStatus.hours"
        :disabled="inputPlayMode"
        @change="handleHoursChange"
      >
        <option
          v-for="item in selectHoursOptions"
          :key="item.value"
          :value="item.value"
        >
          {{ item.hours }}：00 {{ item.isAm ? 'a.m.' : 'p.m.' }}
        </option>
      </select>
      <select
        name="weather"
        v-model="audioStatus.weather"
        :disabled="inputPlayMode"
        @change="handleWeatherChange"
      >
        <option value="sunny">Sunny</option>
        <option value="rainy">Rainy</option>
        <option value="snowy">Snowy</option>
      </select>
      <input
        type="checkbox"
        name="simulate"
        v-model="inputPlayMode"
        @change="handlePlayModeChange"
      />
      模拟播放
    </template>
    <template v-else-if="status === 'loading'">
      <div>loading...</div>
    </template>
    <template v-else-if="status === 'prepare'">
      1. 打开 网易云音乐|QQ音乐|酷我音乐|... ，搜索专辑“あつまれ どうぶつの森
      オリジナルサウンドトラック”，下载全部专辑
      <br />
      2. 等待下载完毕，然后手动选择下载资源所在目录
      <br />
      <input
        type="file"
        @change="handleBasePathChange"
        webkitdirectory
        multiple
      />
    </template>
  </div>
</template>

<script lang="ts">
import jsonp from 'jsonp'
import { Component } from 'vue'
import { Options, Vue } from 'vue-class-component'
import MyAudio from '@/components/MyAudio.vue'
import { getTime } from '@/common'
import { weatherMap } from '@/const'

const pad0 = (v: number) => `${v}`.padStart(2, '0')

function dateToHourlyString(d: Date) {
  const date = `${d.getFullYear()}-${pad0(d.getMonth())}-${pad0(d.getDate())}`
  const time = `${pad0(d.getHours())}:00`
  return `${date} ${time}`
}

function getCurrentHours() {
  return new Date().getHours()
}

function requestGeo() {
  return new Promise<TLocationApiPoint>((resolve) => {
    jsonp(
      'https://api.map.baidu.com/location/ip?ak=kf5mCAHrpjyC9ZB1T9IatxcmpYBqF7nD&coor=bd09ll',
      {},
      (err: Error | null, res: TLocationApi) => {
        if (err) {
          throw err
        }
        if (res.status !== 0 || !res.content?.point) {
          // todo: 具体的错误信息
          throw new Error(`${res.status}`)
        }
        resolve(res.content.point)
      },
    )
    // fetch(
    //   'https://api.map.baidu.com/location/ip?ak=kf5mCAHrpjyC9ZB1T9IatxcmpYBqF7nD&coor=bd09ll',
    // )
    //   .then((response) => {
    //     if (!response.ok) {
    //       throw Error(response.statusText)
    //     }
    //     const res = response.json()
    //     return res
    //   })
    //   .then((res: TLocationApi) => {
    //     if (res.status !== 0 || !res.content?.point) {
    //       // todo: 具体的错误信息
    //       throw new Error(`${res.status}`)
    //     }
    //     resolve(res.content.point)
    //   })
    //   .catch(reject)
  })
}

function requestWeather(coords: TLocationApiPoint) {
  return new Promise<TWeatherCache>((resolve) => {
    jsonp(
      `https://api.caiyunapp.com/v2.5/NrwTfgL23ti6WwW3/${coords.x},${coords.y}/hourly.json?hourlysteps=6`,
      {},
      (err: Error | null, res: TWeatherApi) => {
        if (err) {
          throw err
        }
        if (res.status !== 'ok') {
          throw Error(res.error)
        }

        if (!res.result?.hourly) {
          throw Error()
        }

        const { hourly } = res.result
        if (hourly.status !== 'ok') {
          throw Error(res.error)
        }

        if (!Array.isArray(hourly.skycon)) {
          throw Error()
        }

        const weathers: TWeatherCache = {}
        hourly.skycon.forEach(({ datetime, value }) => {
          const date = new Date(datetime)
          const dateStr = dateToHourlyString(date)
          weathers[dateStr] = weatherMap[value]
        })

        resolve(weathers)
      },
    )
  }).catch((e) => {
    console.warn(e)
    return {}
  })
}

function getWeathers() {
  return new Promise<TWeatherCache>((resolve) => {
    requestGeo()
      .then((coords) => {
        requestWeather(coords).then((res) => resolve(res))
      })
      .catch((e) => {
        console.warn('get weather failed', e)
        resolve({})
        // todo: 通知方式
      })
  })
}
@Options({
  components: {
    MyAudio,
  },
  data() {
    const selectHoursOptions = Array.from({ length: 24 }, (v, i) => ({
      value: i,
      ...getTime(i),
    }))
    return {
      selectHoursOptions,
      status: 'prepare',
      ready: false,

      weatherCache: {} as {},
      currentWeather: 'sunny',

      $audio: null,
      audioStatus: {
        basePath: '',
        hours: getCurrentHours(),
        weather: 'sunny',
      },
      inputPlayMode: true,
    }
  },
  computed: {
    playMode() {
      return this.inputPlayMode ? 'simulate' : 'custom'
    },
  },
  created() {
    this.updateWeather()

    const basePath = window.config.audioBasePath
    if (basePath) {
      // todo: 如何抽取重复
      window
        .checkFiles(basePath)
        .then(() => {
          this.audioStatus.basePath = basePath
          window.saveAudioBasePath(basePath)
          this.status = 'ready'
        })
        .catch((e) => {
          console.warn('check file failed', e)
          this.status = 'prepare'
        })
    }
  },
  methods: {
    updateWeather() {
      const hourlyStr = dateToHourlyString(new Date())
      const update = (refWeatherCache: TWeatherCache, refHourlyStr: string) => {
        const weather = refWeatherCache[refHourlyStr] || 'sunny'
        this.currentWeather = weather
        this.audioStatus.weather = weather
      }

      if (hourlyStr in this.weatherCache) {
        update(this.weatherCache, hourlyStr)
        return
      }

      getWeathers().then((weathers) => {
        this.weatherCache = weathers
        update(this.weatherCache, hourlyStr)
      })
    },
    polling() {
      const hours = getCurrentHours()
      if (this.playMode === 'simulate') {
        const seconds = new Date().getSeconds()
        const minutes = new Date().getMinutes()
        if (seconds === 0 && minutes === 0) {
          // 整点更新天气
          this.updateWeather()
        }
        if (hours !== this.audioStatus.hours && !this.$audio.getStatus()) {
          // 整点更新BGM
          this.audioStatus.hours = hours
        }
      }
    },
    reset() {
      this.audioStatus.hours = getCurrentHours()
      this.audioStatus.weather = this.currentWeather
    },
    handleAudioReady($audio: Component) {
      this.$audio = $audio
      // 开始播放
      this.$audio.play()
      setInterval(() => {
        this.polling()
      }, 950)
    },
    handleHoursChange() {
      this.inputPlayMode = false
    },
    handleWeatherChange() {
      this.inputPlayMode = false
    },
    handleResetClick() {
      this.reset()
    },
    handlePlayModeChange() {
      if (this.playMode === 'simulate') {
        this.reset()
      }
    },
    handleBasePathChange({ target }: { target: HTMLInputElement }) {
      if (!target?.files) {
        return
      }
      const file = target.files[0]
      const basePath = file.path.slice(0, file.path.lastIndexOf(file.name))
      // todo: 如何抽取重复
      window
        .checkFiles(basePath)
        .then(() => {
          this.audioStatus.basePath = basePath
          window.saveAudioBasePath(basePath)
          this.status = 'ready'
        })
        .catch((e: unknown) => {
          console.warn('check file failed', e)
          this.status = 'prepare'
        })
    },
  },
})
export default class App extends Vue { }
</script>

<style lang="scss">
html,
body {
  min-height: 100vh;
}

body {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: auto;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica,
    Arial, sans-serif;
  text-align: center;
}

#app {
  padding: 1.5rem 2rem;
}
</style>