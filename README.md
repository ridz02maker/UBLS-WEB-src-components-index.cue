# UBLS Bus Tracker

A bus location system that displays routes and real-time vehicle positions
for multiple transit agencies.

## Preview
<template>
  <div>
    <util-header ref="headerRef"/>
    <div class="flex flex-col relative top-12">
      <!--Notification-->
      <div v-show="headerRef?.notificationText && headerRef?.notificationText.trim() !== ''" class="animate-slide-in-top flex items-center mt-4 mx-4 lg:mx-80 p-2 text-xl lg:text-2xl text-black border rounded-lg bg-white dark:text-black dark:border-blue-800 font-NotoSansJP" role="alert">
        <p>{{headerRef?.notificationText}}</p>
      </div>
      <!--main page-->
      <p v-if="agencyClock" class="m-2 lg:ml-72 text-sm tabular-nums">
      {{ headerRef?.isEnglish ? 'Local time' : '現地時刻' }}
    （{{ headerRef?.isEnglish ? agencyClock.englishName : agencyClock.japaneseName }}）
      <time :datetime="clockNow.toISOString()">{{ agencyLocalTime }}</time>
      </p>
      <h2 v-if="!headerRef?.isEnglish" class="m-2 lg:ml-72 max-lg:text-lg lg:text-base flex items-center">◎地図上でバスを探す</h2>
      <h2 v-else class="m-2 lg:ml-72 max-lg:text-lg lg:text-base flex items-center">Find on maps</h2>
      <!-- 地図表示ボタン（運行中バスがいる場合のみ表示） -->
      <div v-if="filteredTrips.size !== 0 && sortedMaps.length > 0" class="flex flex-col">
        <a v-for="map in sortedMaps" :key="map.map_id"
            class="text-white py-3 max-lg:mx-4 lg:mx-80 mb-2 text-3xl text-center font-weight-medium font-bold rounded flex justify-center"
            :style="{ backgroundColor: '#' + (map.map_button_color || '22c55e') }"
            :href="getMapURLByMapId(map.map_id)">
          {{ map.map_name }}
        </a>
      </div>
      <div v-else-if="filteredTrips.size === 0 && !headerRef?.isEnglish" class="bg-red-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-2 text-2xl text-center font-weight-medium rounded flex justify-center" role="alert">
        <p class="font-medium">運行中のバスはありません</p>
      </div>
      <div v-else-if="filteredTrips.size === 0 && headerRef?.isEnglish" class="bg-red-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-2 text-2xl text-center font-weight-medium rounded flex justify-center" role="alert">
        <p class="font-medium">Services not found</p>
      </div>
      <div v-else-if="filteredTrips.size !== 0 && sortedMaps.length === 0 && !headerRef?.isEnglish" class="bg-yellow-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-2 text-2xl text-center font-weight-medium rounded flex justify-center" role="alert">
        <p class="font-medium">地図表示ボタンが設定されていません</p>
      </div>
      <div v-else-if="filteredTrips.size !== 0 && sortedMaps.length === 0 && headerRef?.isEnglish" class="bg-yellow-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-2 text-2xl text-center font-weight-medium rounded flex justify-center" role="alert">
        <p class="font-medium">Map display buttons not configured</p>
      </div>
      <h2 v-if="filteredTrips.size !== 0 && !headerRef?.isEnglish" class="mx-2 mt-2 max-lg:text-lg lg:text-base flex items-center lg:ml-72">◎運行中のバスを選ぶ</h2>
      <h2 v-else-if="filteredTrips.size !== 0 && headerRef?.isEnglish" class="mx-2 mt-2 max-lg:text-lg lg:text-base flex items-center lg:ml-72">Select a bus in operation</h2>
      
      <!-- 地域別表示 -->
      <div v-if="filteredTrips.size !== 0 && areas.length > 0" class="flex flex-col">
        <div v-for="area in sortedAreas" :key="area.area_id" class="mb-4">
          <template v-if="getTripsForArea(area.area_id).length > 0">
            <h3 v-if="!headerRef?.isEnglish" class="mx-4 lg:mx-80 text-lg font-semibold text-gray-700 mb-2 border-b pb-1">
              {{ area.area_name }}
            </h3>
            <h3 v-else class="mx-4 lg:mx-80 text-lg font-semibold text-gray-700 mb-2 border-b pb-1">
              {{ area.area_name }}
            </h3>
            
            <div v-if="!headerRef?.isEnglish" class="flex flex-col">
              <a v-for="trip in getTripsForArea(area.area_id)"
                :key="trip.agency_str + '-' + trip.description.tripId.ja"
                :style="{
                  backgroundColor: '#' + getRouteColor(trip),
                  color: '#' + getTextColor(getRouteColor(trip))
                }"
                class="py-2 max-lg:mx-4 max-lg:my-1 lg:mx-80 lg:my-1 max-lg:text-xl lg:text-base rounded text-center whitespace-pre-wrap font-bold"
                :href="mapURL+trip.agency_str+'/map/'+trip.description.tripId.ja"
              >{{ getJPDescByTrip(trip) }}</a>
            </div>
            <div v-else class="flex flex-col">
              <a v-for="trip in getTripsForArea(area.area_id)"
                :key="trip.agency_str + '-' + trip.description.tripId.ja"
                :style="{
                  backgroundColor: '#' + getRouteColor(trip),
                  color: '#' + getTextColor(getRouteColor(trip))
                }"
                class="py-2 max-lg:mx-4 max-lg:my-1 lg:mx-80 lg:my-1 max-lg:text-xl lg:text-base rounded text-center whitespace-pre-wrap font-bold"
                :href="mapURL+trip.agency_str+'/map/'+trip.description.tripId.ja"
              >{{ getENDescByTrip(trip) }}</a>
            </div>
          </template>
        </div>
        
        <!-- 地域未設定の路線（ヘッダーなし） -->
        <template v-if="getTripsWithoutArea().length > 0">
          <div v-if="!headerRef?.isEnglish" class="flex flex-col">
            <a v-for="trip in getTripsWithoutArea()"
              :key="trip.agency_str + '-' + trip.description.tripId.ja"
              :style="{
                backgroundColor: '#' + getRouteColor(trip),
                color: '#' + getTextColor(getRouteColor(trip))
              }"
              class="py-2 max-lg:mx-4 max-lg:my-1 lg:mx-80 lg:my-1 max-lg:text-xl lg:text-base rounded text-center whitespace-pre-wrap"
              :href="mapURL+trip.agency_str+'/map/'+trip.description.tripId.ja"
            >{{ getJPDescByTrip(trip) }}</a>
          </div>
          <div v-else class="flex flex-col">
            <a v-for="trip in getTripsWithoutArea()"
              :key="trip.agency_str + '-' + trip.description.tripId.ja"
              :style="{
                backgroundColor: '#' + getRouteColor(trip),
                color: '#' + getTextColor(getRouteColor(trip))
              }"
              class="py-2 max-lg:mx-4 max-lg:my-1 lg:mx-80 lg:my-1 max-lg:text-xl lg:text-base rounded text-center whitespace-pre-wrap"
              :href="mapURL+trip.agency_str+'/map/'+trip.description.tripId.ja"
            >{{ getENDescByTrip(trip) }}</a>
          </div>
        </template>
      </div>
      
      <!-- 地域設定がない場合の従来表示 -->
      <div v-else-if="filteredTrips.size !== 0 && !headerRef?.isEnglish" class="flex flex-col">
        <a v-for="[trip] in filteredTrips"
          :key="trip.agency_str + '-' + trip.description.tripId.ja"
          :style="{
            backgroundColor: '#' + getRouteColor(trip),
            color: '#' + getTextColor(getRouteColor(trip))
          }"
          class="py-2 max-lg:mx-4 max-lg:my-3 lg:mx-80 lg:my-2 max-lg:text-xl lg:text-base rounded text-center whitespace-pre-wrap"
        :href="mapURL+trip.agency_str+'/map/'+trip.description.tripId.ja"
        >{{ getJPDescByTrip(trip) }}</a>
      </div>
      <div v-else-if="filteredTrips.size !== 0 && headerRef?.isEnglish" class="flex flex-col">
        <a v-for="[trip] in filteredTrips"
          :key="trip.agency_str + '-' + trip.description.tripId.ja"
          :style="{
            backgroundColor: '#' + getRouteColor(trip),
            color: '#' + getTextColor(getRouteColor(trip))
          }"
          class="py-2 max-lg:mx-4 max-lg:my-3 lg:mx-80 lg:my-2 max-lg:text-xl lg:text-base rounded text-center whitespace-pre-wrap"
          :href="mapURL+trip.agency_str+'/map/'+trip.description.tripId.ja"
        >{{ getENDescByTrip(trip) }}</a>
      </div>
      <!-- Manual page link -->
      <h2 v-if="!headerRef?.isEnglish" class="m-2 lg:ml-72 max-lg:text-lg lg:text-base flex items-center">◎使い方</h2>
      <h2 v-else class="m-2 lg:ml-72 max-lg:text-lg lg:text-base flex items-center">How to use</h2>
      <a v-if="!headerRef?.isEnglish"
         class="bg-green-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-4 text-2xl text-center font-weight-medium font-bold rounded flex justify-center cursor-pointer"
         :href="manualUrl">使い方はこちら</a>
      <a v-else
         class="bg-green-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-4 text-2xl text-center font-weight-medium font-bold rounded flex justify-center cursor-pointer"
         :href="manualUrl">How to use</a>
      
      <template v-if="headerRef?.surveyUrl">
        <h2 v-if="!headerRef?.isEnglish" class="m-2 lg:ml-72 max-lg:text-lg lg:text-base flex items-center">◎お知らせ</h2>
        <h2 v-else class="m-2 lg:ml-72 max-lg:text-lg lg:text-base flex items-center">Information</h2>
        <a v-if="headerRef?.surveyUrl && !headerRef?.isEnglish"
           class="bg-green-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-4 text-2xl text-center font-weight-medium font-bold rounded flex justify-center cursor-pointer"
           :href="headerRef.surveyUrl"
           target="_blank"
           rel="noopener noreferrer">利用者アンケートはこちら</a>
        <a v-else-if="headerRef?.surveyUrl && headerRef?.isEnglish"
           class="bg-green-500 text-white py-3 max-lg:mx-4 lg:mx-80 mb-4 text-2xl text-center font-weight-medium font-bold rounded flex justify-center cursor-pointer"
           :href="headerRef.surveyUrl"
           target="_blank"
           rel="noopener noreferrer">User Survey</a>
      </template>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, watchEffect, computed } from "vue"
import utilHeader from '../components/header.vue'
import { useRoute, useRouter } from 'vue-router'
import axios from "axios"

const apiMap = ref(new Map())
const reloadMs = ref(60000)
const mapURL = ref("/")
const trips = ref(new Map())
const allTrips = ref(new Map())
const displayRoutes = ref(new Map()) // agency_str -> route_ids with view=true
const areas = ref([]) // 地域データ
const sortedAreas = ref([]) // ソート済み地域データ
const routeAreaMap = ref(new Map()) // route_id -> area_id のマッピング
const maps = ref([]) // 地図表示ボタンデータ
const sortedMaps = ref([]) // ソート済み地図表示ボタンデータ
const headerRef = ref()
const route = useRoute()
const router = useRouter()

const agencyClocks = new Map([
  ['kitami', {
    timeZone: 'Asia/Tokyo',
    englishName: 'Kitami',
    japaneseName: '北見',
  }],
  ['natori', {
    timeZone: 'Asia/Tokyo',
    englishName: 'Natori',
    japaneseName: '名取',
  }],
  ['gopi', {
    timeZone: 'Asia/Kuala_Lumpur',
    englishName: 'GOPI',
    japaneseName: 'GOPI',
  }],
  ['dole', {
    timeZone: 'Europe/Paris',
    englishName: 'Dole',
    japaneseName: 'ドール',
  }],
  ['kat', {
    timeZone: 'America/New_York',
    englishName: 'Knoxville',
    japaneseName: 'ノックスビル',
  }],
  ['lmt', {
    timeZone: 'Asia/Colombo',
    englishName: 'Sri Lanka',
    japaneseName: 'スリランカ',
  }],
])

const clockNow = ref(new Date())

const agencyClock = computed(() => {
  const agency = route.params.agency || headerRef.value?.currentAgency || 'kitami'
  console.log('[agency clock]', JSON.stringify(agency), agencyClocks.has(agency))
  return agencyClocks.get(agency)
})

const clockFormatter = computed(() => {
  if (!agencyClock.value) return null

  return new Intl.DateTimeFormat(
    headerRef.value?.isEnglish ? 'en-GB' : 'ja-JP',
    {
      timeZone: agencyClock.value.timeZone,
      dateStyle: 'medium',
      timeStyle: 'medium',
      hourCycle: 'h23',
    },
  )
})

const agencyLocalTime = computed(() => {
  return clockFormatter.value?.format(clockNow.value) ?? ''
})

let clockTimer

onMounted(() => {
  clockTimer = window.setInterval(() => {
    clockNow.value = new Date()
  }, 1000)
})

onUnmounted(() => {
  window.clearInterval(clockTimer)
})

// display_routesのviewが有効な路線のみ表示
const filteredTrips = computed(() => {
  const filtered = new Map()
  console.log("=== FILTERING TRIPS ===")
  console.log("allTrips size:", allTrips.value.size)
  console.log("displayRoutes:", Object.fromEntries(displayRoutes.value))
  
  for (const [trip] of allTrips.value.entries()) {
    const agencyStr = trip.agency_str
    const routeId = trip.description.routeId.ja
    console.log(`Checking trip: agency_str=${agencyStr}, routeId=${routeId}`)
    
    const agencyRoutes = displayRoutes.value.get(agencyStr)
    console.log(`Routes for ${agencyStr}:`, agencyRoutes ? Array.from(agencyRoutes) : 'undefined')
    
    if (agencyRoutes && agencyRoutes.has(routeId)) {
      console.log(`✅ Trip INCLUDED: ${agencyStr} - ${routeId}`)
      filtered.set(trip, agencyStr)
    } else {
      console.log(`❌ Trip EXCLUDED: ${agencyStr} - ${routeId} (agencyRoutes=${!!agencyRoutes}, hasRoute=${agencyRoutes?.has(routeId)})`)
    }
  }
  
  console.log("=== FILTERING COMPLETE ===")
  console.log("Filtered trips size:", filtered.size)
  return filtered
})

// マニュアルページのURL
const manualUrl = computed(() => {
  const agency = route.params.agency || headerRef.value?.currentAgency || 'kitami'
  return `/${agency}/manual`
})

onMounted(async () => {
  setFavicon()
  setAPIMap()

  mapURL.value = "/"
  
  await getDisplayRoutes()
  await getAreas()
  await getMaps()
  await getTrips()
  watchEffect(() => {
    document.title = headerRef.value.title
  })
})
onMounted(() => {
  setInterval(getTrips, reloadMs.value)
})

function setAPIMap() {
  const apiCollectionKeys = import.meta.env.VITE_API_COLLECTION_KEYS.split(',');
  const apiCollection = {};

  apiCollectionKeys.forEach(key => {
    const value = import.meta.env[`VITE_API_COLLECTION_${key.toUpperCase()}`];
    apiCollection[key] = value;
    apiMap.value.set(key, value)
  });
  if (apiCollection.length === 0) {
    console.error("API_COLLECTION NOT FOUND.")
  }
}

async function getDisplayRoutes() {
  displayRoutes.value.clear()
  
  // URLパスから事業者を取得
  const requestedAgency = route.params.agency
  
  // 特定のagencyが指定されているが存在しない場合は404ページにリダイレクト
  if (requestedAgency && !apiMap.value.has(requestedAgency)) {
    console.log(`Agency '${requestedAgency}' not found in apiMap, redirecting to 404`)
    router.push({ name: 'NotFound' })
    return
  }
  
  // URLパスから事業者を取得（指定がない場合はkitamiをデフォルトとする）
  let currentAgency = requestedAgency || 'kitami'
  
  // kitamiも存在しない場合は利用可能な最初のagencyを使用
  if (!apiMap.value.has(currentAgency) && apiMap.value.size > 0) {
    currentAgency = Array.from(apiMap.value.keys())[0]
    console.log(`kitami not found, using first available agency: ${currentAgency}`)
  }
  
  console.log("Current agency from route:", currentAgency)
  console.log("Available agencies in apiMap:", Array.from(apiMap.value.keys()))
  
  // 特定の事業者パスでアクセスしている場合は、その事業者のdisplay_routesのみを取得
  if (currentAgency && apiMap.value.has(currentAgency)) {
    const url = apiMap.value.get(currentAgency)
    console.log(`Processing display routes for agency: ${currentAgency}, URL: ${url}`)
    try {
      const response = await axios.get(url + '/display_routes')
      console.log(`Display routes response for ${currentAgency}:`, response.data)
      
      // 各ルートレコードをagency_strごとにグループ化
      const agencyRoutes = new Map()
      response.data.forEach(route => {
        console.log(`Route: agency_str=${route.agency_str}, route_id=${route.route_id}, view=${route.view}`)
        if (route.view) {
          if (!agencyRoutes.has(route.agency_str)) {
            agencyRoutes.set(route.agency_str, new Set())
          }
          agencyRoutes.get(route.agency_str).add(route.route_id)
        }
      })
      
      // displayRoutesに各agency_strの路線を追加
      for (const [agencyStr, routes] of agencyRoutes.entries()) {
        displayRoutes.value.set(agencyStr, routes)
        console.log(`Display routes for agency_str ${agencyStr}:`, routes)
      }
    } catch (error) {
      console.log("Display Routes Error for " + currentAgency + ": " + error)
    }
  } else {
    console.log("Processing fallback case")
    // フォールバック：kitamiが存在しない場合は全ての事業者を処理
    for (const [agency, url] of apiMap.value.entries()) {
      try {
        const response = await axios.get(url + '/display_routes')
        console.log(`Display routes response for ${agency}:`, response.data)
        
        // 各ルートレコードをagency_strごとにグループ化
        const agencyRoutes = new Map()
        response.data.forEach(route => {
          if (route.view) {
            if (!agencyRoutes.has(route.agency_str)) {
              agencyRoutes.set(route.agency_str, new Set())
            }
            agencyRoutes.get(route.agency_str).add(route.route_id)
          }
        })
        
        // displayRoutesに各agency_strの路線を追加
        for (const [agencyStr, routes] of agencyRoutes.entries()) {
          displayRoutes.value.set(agencyStr, routes)
          console.log(`Display routes for agency_str ${agencyStr}:`, routes)
        }
      } catch (error) {
        console.log("Display Routes Error for " + agency + ": " + error)
      }
    }
  }
  console.log("All display routes:", displayRoutes.value)
}

async function getAreas() {
  areas.value = []
  routeAreaMap.value.clear()
  
  // URLパスから事業者を取得
  const requestedAgency = route.params.agency
  
  // 特定のagencyが指定されているが存在しない場合は処理を中断（404は既にgetDisplayRoutesで処理済み）
  if (requestedAgency && !apiMap.value.has(requestedAgency)) {
    console.log(`Agency '${requestedAgency}' not found in apiMap, skipping getAreas`)
    return
  }
  
  // URLパスから事業者を取得（指定がない場合はkitamiをデフォルトとする）
  let currentAgency = requestedAgency || 'kitami'
  
  // kitamiも存在しない場合は利用可能な最初のagencyを使用
  if (!apiMap.value.has(currentAgency) && apiMap.value.size > 0) {
    currentAgency = Array.from(apiMap.value.keys())[0]
    console.log(`kitami not found, using first available agency: ${currentAgency}`)
  }
  
  console.log("Current agency from route:", currentAgency)
  
  // 特定の事業者パスでアクセスしている場合は、その事業者の地域データのみを取得
  if (currentAgency && apiMap.value.has(currentAgency)) {
    const url = apiMap.value.get(currentAgency)
    console.log(`Processing areas for agency: ${currentAgency}, URL: ${url}`)
    try {
      // 地域データを取得
      const areasResponse = await axios.get(url + '/areas')
      areas.value = areasResponse.data || []
      console.log(`Areas response for ${currentAgency}:`, areas.value)
      
      // 地域情報付き路線データを取得
      const routesResponse = await axios.get(url + '/display_routes_with_area')
      console.log(`Routes with area response for ${currentAgency}:`, routesResponse.data)
      
      // route_id -> area_id のマッピングを作成
      routesResponse.data.forEach(route => {
        if (route.area_id) {
          const key = `${route.agency_str}-${route.route_id}`
          routeAreaMap.value.set(key, route.area_id)
        }
      })
    } catch (error) {
      console.log("Areas Error for " + currentAgency + ": " + error)
    }
  } else {
    console.log("Processing areas for fallback case")
    // フォールバック：kitamiが存在しない場合は最初の事業者の地域データを使用
    for (const [agency, url] of apiMap.value.entries()) {
      try {
        // 地域データを取得
        const areasResponse = await axios.get(url + '/areas')
        areas.value = areasResponse.data || []
        console.log(`Areas response for ${agency}:`, areas.value)
        
        // 地域情報付き路線データを取得
        const routesResponse = await axios.get(url + '/display_routes_with_area')
        console.log(`Routes with area response for ${agency}:`, routesResponse.data)
        
        // route_id -> area_id のマッピングを作成
        routesResponse.data.forEach(route => {
          if (route.area_id) {
            const key = `${route.agency_str}-${route.route_id}`
            routeAreaMap.value.set(key, route.area_id)
          }
        })
        
        // 最初の事業者のデータのみ使用
        break
      } catch (error) {
        console.log("Areas Error for " + agency + ": " + error)
      }
    }
  }
  
  // 地域を優先度順でソート
  sortedAreas.value = [...areas.value].sort((a, b) => {
    if (a.area_priority !== b.area_priority) {
      return a.area_priority - b.area_priority
    }
    return a.area_name.localeCompare(b.area_name)
  })
  
  console.log("Areas loaded:", areas.value)
  console.log("Route area mapping:", Object.fromEntries(routeAreaMap.value))
}

async function getMaps() {
  maps.value = []
  
  // URLパスから事業者を取得
  const requestedAgency = route.params.agency
  
  // 特定のagencyが指定されているが存在しない場合は処理を中断
  if (requestedAgency && !apiMap.value.has(requestedAgency)) {
    console.log(`Agency '${requestedAgency}' not found in apiMap, skipping getMaps`)
    return
  }
  
  // URLパスから事業者を取得（指定がない場合はkitamiをデフォルトとする）
  let currentAgency = requestedAgency || 'kitami'
  
  // kitamiも存在しない場合は利用可能な最初のagencyを使用
  if (!apiMap.value.has(currentAgency) && apiMap.value.size > 0) {
    currentAgency = Array.from(apiMap.value.keys())[0]
    console.log(`kitami not found, using first available agency: ${currentAgency}`)
  }
  
  console.log("Current agency from route:", currentAgency)
  
  // 特定の事業者パスでアクセスしている場合は、その事業者の地図データのみを取得
  if (currentAgency && apiMap.value.has(currentAgency)) {
    const url = apiMap.value.get(currentAgency)
    console.log(`Processing maps for agency: ${currentAgency}, URL: ${url}`)
    try {
      const response = await axios.get(url + '/maps')
      maps.value = response.data || []
      console.log(`Maps response for ${currentAgency}:`, maps.value)
    } catch (error) {
      console.log("Maps Error for " + currentAgency + ": " + error)
    }
  } else {
    console.log("Processing maps for fallback case")
    // フォールバック：kitamiが存在しない場合は最初の事業者の地図データを使用
    for (const [agency, url] of apiMap.value.entries()) {
      try {
        const response = await axios.get(url + '/maps')
        maps.value = response.data || []
        console.log(`Maps response for ${agency}:`, maps.value)
        break
      } catch (error) {
        console.log("Maps Error for " + agency + ": " + error)
      }
    }
  }
  
  // 地図を優先度順でソート
  sortedMaps.value = [...maps.value].sort((a, b) => {
    if (a.map_priority !== b.map_priority) {
      return a.map_priority - b.map_priority
    }
    return a.map_name.localeCompare(b.map_name)
  })
  
  console.log("Maps loaded:", maps.value)
}

async function getTrips() {
  allTrips.value.clear()
  
  // 常に全ての事業者のtripsを取得
  console.log("Getting trips from all agencies")
  for (const [agency, url] of apiMap.value.entries()) {
    let currentOperation = null
    try {
      currentOperation = await axios.get(url.replace("v1", "v4") + '/operation')
      trips.value.delete(agency)
      trips.value.set(agency, currentOperation.data)
      const tripArray = JSON.parse(JSON.stringify(currentOperation.data))
      console.log(`Trips for ${agency}:`, tripArray.length)
      tripArray.forEach((element) => {
        console.log(`Trip: agency_str=${element.agency_str}, routeId=${element.description.routeId.ja}`)
        allTrips.value.set(element, element.agency_str)
      })
    } catch (error) {
      console.log("Trips Error : " + error)
    }
  }
  console.log("Total trips loaded:", allTrips.value.size)
}

function getJPDescByTrip(trip) {
  if (
    trip.agency_str === 'lmt' &&
    /^LMT_CM\d+_[01]_/.test(trip.tripId)
  ) {
    return 'GPS位置情報・出発時刻不明\n' + trip.description.routeName.ja
  }

  if (trip.description.stopHeadsign.ja === "") {
    return trip.description.departureTime.slice(0, -3) + "発 " + trip.description.routeName.ja + '\n' + trip.description.departureName.ja + " ⇒ " + trip.description.arrivalName.ja
  } else {
    return trip.description.departureTime.slice(0, -3) + "発 " + trip.description.routeName.ja + '\n' + trip.description.stopHeadsign.ja + "方面"
  }
}

function getENDescByTrip(trip) {
  if (
  trip.agency_str === 'lmt' &&
  /^LMT_CM\d+_[01]_/.test(trip.tripId)
  ) {
  return 'Live GPS · departure time unavailable\n' +
    (trip.description.routeName.en || trip.description.routeName.ja)
  }
  if (trip.description.routeName.en === "") {
    return trip.description.departureTime.slice(0, -3) + " " + trip.description.routeName.ja + '\n' + trip.description.departureName.en + " ⇒ " + trip.description.arrivalName.en
  } else {
    if (trip.description.stopHeadsign.en === "") {
      return trip.description.departureTime.slice(0, -3) + " " + trip.description.routeName.en + '\n' + trip.description.departureName.en + " ⇒ " + trip.description.arrivalName.en
    } else {
      return trip.description.departureTime.slice(0, -3) + " " + trip.description.routeName.en + '\n' + trip.description.stopHeadsign.en
    }
  }
}

function getRouteColor(trip) {
  if (trip.description.routeColor !== "FFFFFF") {
    return trip.description.routeColor
  }
  return "9D5B8B"
}

function getTextColor(routeColor) {
  let r = parseInt(routeColor.substr(0, 2), 16)
  let g = parseInt(routeColor.substr(2, 2), 16)
  let b = parseInt(routeColor.substr(4, 2), 16)
  r *= 0.299
  g *= 0.587
  b *= 0.114
  const midLuminance = 138
  if (r + g + b > midLuminance) {
    return "000000"
  } else {
    return "FFFFFF"
  }
}

function getMapURL() {
  const currentAgency = route.params.agency
  if (currentAgency) {
    return `/${currentAgency}/map`
  } else {
    return '/map'
  }
}

function setFavicon() {
  const link = document.createElement('link')
  link.rel = 'icon'
  link.href = import.meta.env.VITE_DEPLOY_URL + '/assets/buslocationicon.png'
  document.head.appendChild(link)
}

// 地域別のtrip取得関数
function getTripsForArea(areaId) {
  const tripsForArea = []
  
  for (const [trip] of filteredTrips.value.entries()) {
    const routeKey = `${trip.agency_str}-${trip.description.routeId.ja}`
    const tripAreaId = routeAreaMap.value.get(routeKey)
    
    if (tripAreaId === areaId) {
      tripsForArea.push(trip)
    }
  }
  
  return tripsForArea
}

// 地域未設定のtrip取得関数
function getTripsWithoutArea() {
  const tripsWithoutArea = []
  
  for (const [trip] of filteredTrips.value.entries()) {
    const routeKey = `${trip.agency_str}-${trip.description.routeId.ja}`
    const tripAreaId = routeAreaMap.value.get(routeKey)
    
    if (!tripAreaId) {
      tripsWithoutArea.push(trip)
    }
  }
  
  return tripsWithoutArea
}

function getMapURLByMapId(mapId) {
  const currentAgency = route.params.agency
  if (currentAgency) {
    return `/${currentAgency}/map?mapId=${mapId}`
  } else {
    return `/map?mapId=${mapId}`
  }
}
</script>

<style scoped>
* {
  font-family: "Noto Sans JP",sans-serif;
}
.font-weight-medium {
  font-family: "Noto Sans JP medium",sans-serif;
}
</style>

## Add code
Add the clock logic

import { onMounted, onUnmounted, ref, watchEffect, computed } from "vue"

