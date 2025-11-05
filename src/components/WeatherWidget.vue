<script setup>
import { ref, reactive, onMounted, watch, onUnmounted, computed } from 'vue'
import dayjs from 'dayjs'
import Spinner from './Spinner.vue'

const weatherApiKey = import.meta.env.VITE_WEATHER_API_KEY
const imageApiKey = import.meta.env.VITE_IMAGE_API_KEY
const DEFAULT_BACKGROUND_IMAGE = '/images/default-bg.webp'
const WEATHER_UPDATE_INTERVAL = 30 * 60 * 1000

const props = defineProps({
  latitude: Number,
  longitude: Number
})

const bgURL = ref(DEFAULT_BACKGROUND_IMAGE)

const weather = reactive({
  city: null,
  state: null, //for a picture
  temperature: null,
  date: null,
  description: null,
  icon: null
})

const isLoading = ref(true)
const error = ref(null)

let intervalId = null

function clearWeatherInterval() {
  if (intervalId !== null) {
    clearInterval(intervalId)
    intervalId = null
  }
}

function startWeatherUpdate() {
  isLoading.value = true
  clearWeatherInterval()
  getWeather()
  intervalId = setInterval(getWeather, WEATHER_UPDATE_INTERVAL)
}

async function getBackgroundImageURL(city, state) {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), 5000)

  try {
    const searchQuery = (city + ' ' + state).replaceAll(' ', '%20')
    const response = await fetch(`https://api.unsplash.com/search/photos?page=1&query=${searchQuery}&client_id=${imageApiKey}&per_page=1`, {
      signal: controller.signal
    })

    clearTimeout(timeoutId)

    if (!response.ok) return DEFAULT_BACKGROUND_IMAGE
    
    const imageData = await response.json()
    return imageData?.results?.[0]?.urls?.regular || DEFAULT_BACKGROUND_IMAGE

  } catch {
    clearTimeout(timeoutId)
    return DEFAULT_BACKGROUND_IMAGE
  }
}

function preloadBackgroundImage(imageUrl) {
  const img = new Image();
  img.src = imageUrl;
}

async function getLocationData() {
  const response = await fetch(`https://api.openweathermap.org/geo/1.0/reverse?lat=${props.latitude}&lon=${props.longitude}&limit=1&appid=${weatherApiKey}`)

  if (!response.ok) throw new Error('Geocoding API error')

  try {
    return await response.json()
  } catch {
    throw new Error('JSON Parsing went wrong')
  }
}

async function getWeatherData() {
  const response = await fetch(`https://api.openweathermap.org/data/2.5/weather?lat=${props.latitude}&lon=${props.longitude}&appid=${weatherApiKey}&units=metric`)

  if (!response.ok) throw new Error('Weather API error')
  
  try {
    return await response.json()
  } catch {
    throw new Error('JSON Parsing went wrong')
  }
}

async function getWeather() {
  try {
    const [locationData, weatherData] = await Promise.all([getLocationData(), getWeatherData()])

    Object.assign(weather, {
      city: locationData?.[0]?.name,
      state: locationData?.[0]?.state || '',
      temperature: Math.round(weatherData?.main?.temp),
      date: dayjs(),
      description: weatherData?.weather?.[0]?.description,
      icon: weatherData?.weather?.[0]?.icon
    })

    error.value = null

    bgURL.value = await getBackgroundImageURL(weather.city, weather.state)
  } catch (err) {
    error.value = err.message
    bgURL.value = DEFAULT_BACKGROUND_IMAGE
  } finally {
    isLoading.value = false
  }
}


const formattedWeather = computed(() => {
  if (!weather.city || !weather.date) {
    return {
      city: '', temperature: '--°C', date: '', description: '', icon: ''
    }
  }

  return {
    city: weather.city.toUpperCase(),
    state: weather.state,
    temperature: `${weather.temperature}°C`,
    date: weather.date.format('dddd, D MMMM'),
    description: weather.description,
    icon: `https://openweathermap.org/img/wn/${weather.icon}@2x.png`
  }
})

const temperatureColors = [
  {temp: -20, color: '75, 51, 255'},
  {temp: -5, color: '51, 143, 255'},
  {temp: 15, color: '0, 195, 255'},
  {temp: 25, color: '255, 208, 95'},
  {temp: 35, color: '255, 169, 38'},
  {temp: Infinity, color: '255, 118, 38'},
]

const bgGradientColor = computed(() => {
  const temperature = weather.temperature
  const defaultRGB = '61, 217, 50'

  if (temperature === null) return defaultRGB

  const range = temperatureColors.find(range => temperature <= range.temp)
  return range?.color || defaultRGB
})

watch([() => props.latitude, () => props.longitude], () => {
  startWeatherUpdate()
})

onMounted(() => {
  startWeatherUpdate()
})

onUnmounted(() => {
  clearWeatherInterval()
})
</script>

<template>
  <div v-if="error && !isLoading" class="weather-widget widget-base widget-state--error">
    <div class="widget__message">Something went wrong. Try again.</div>
    <button class="widget__button" @click="startWeatherUpdate">Try Again</button>
  </div>

  <div v-else-if="isLoading" class="weather-widget widget-base widget-state--loading">
    <Spinner />
    <div class="widget__message">Loading weather data...</div>
  </div>

  <div v-else class="weather-widget widget-base weather-widget--content">
    <div class="weather-widget__header"
      :style="{backgroundImage: `linear-gradient(to right,rgba(${bgGradientColor}, 0.25) 0%, rgba(${bgGradientColor}, 0.98) 100%), url(${bgURL})`}"
    >
      <div class="weather-widget__header-wrapper">
        <p class="weather-widget__temperature">{{ formattedWeather.temperature }}</p>
        <h1 class="weather-widget__city">{{ formattedWeather.city }}</h1>
      </div>
    </div>

    <div class="weather-widget__body">
      <p class="weather-widget__date">{{ formattedWeather.date }}</p>
      <p class="weather-widget__description">{{ formattedWeather.description }}</p>
      <div class="weather-widget__icon">
        <img :src="formattedWeather.icon" :alt="formattedWeather.description"/>
      </div>
    </div>
  </div>
</template>

<style scoped>
.widget-base.weather-widget--content {
  padding: 0;
}

.weather-widget--content {
  display: flex;
  flex-direction: column;
}

.weather-widget__header {
  flex: 1 1 auto;
  background-size: cover;
}

.weather-widget__header-wrapper {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
  width: fit-content;
}

.weather-widget__body {
  padding: 1rem;
  display: grid;
  grid-template:
    "date date" auto
    "description icon" auto
    / auto min-content;
  align-items: center;
}

.weather-widget__temperature, .weather-widget__city {
  color: #fff;
}

.weather-widget__city {
  word-break: break-all;
}

.weather-widget__date {
  grid-area: date;
  color: #191919;
}

.weather-widget__description {
  grid-area: description;
  color: #6B6B6B;
}

.weather-widget__icon {
  grid-area: icon;
  width: 50px;
  height: 50px;
  display: flex;
  align-items: center;
  justify-items: center;
  overflow: hidden;
}

.weather-widget__icon img {
  height: 100%;
  width: auto;
  object-position: center;
}
</style>