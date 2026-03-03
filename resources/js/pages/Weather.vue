<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import axios from 'axios';

const city = ref('');
const weatherData = ref(null);
const airQuality = ref(null);
const province = ref('');
const loading = ref(false);
const currentTime = ref('');
let timer = null;
const apiKey = import.meta.env.VITE_OPENWEATHER_API_KEY;

const updateLocalTime = () => {
    if (!weatherData.value) return;

    const offsetSeconds = weatherData.value.city.timezone;
    const utcDate = new Date();
    const localDate = new Date(
        utcDate.getTime() +
            utcDate.getTimezoneOffset() * 60000 +
            offsetSeconds * 1000,
    );

    currentTime.value = localDate.toLocaleTimeString('en-US', {
        hour: '2-digit',
        minute: '2-digit',
        hour12: true,
    });
};

const getWeatherTheme = (condition) => {
    const themes = {
        Clear: 'from-sky-300 to-blue-600',
        Clouds: 'from-slate-400 to-slate-600',
        Rain: 'from-indigo-500 to-slate-700',
        Drizzle: 'from-cyan-600 to-blue-800',
        Thunderstorm: 'from-gray-800 to-purple-800',
        Snow: 'from-blue-50 to-blue-200 text-slate-900',
        Mist: 'from-emerald-400 to-teal-600',
        Haze: 'from-orange-300 to-slate-500',
    };
    return themes[condition] || 'from-blue-400 to-indigo-400';
};

const bgGradient = computed(() => {
    const condition =
        weatherData.value?.list?.[0]?.weather?.[0]?.main || 'Default';
    return getWeatherTheme(condition);
});

const getAQIDescription = (aqi) => {
    const levels = {
        1: { text: 'Good', color: 'text-green-300' },
        2: { text: 'Fair', color: 'text-yellow-200' },
        3: { text: 'Moderate', color: 'text-orange-300' },
        4: { text: 'Poor', color: 'text-red-400' },
        5: { text: 'Very Poor', color: 'text-purple-400' },
    };
    return levels[aqi] || { text: 'Unknown', color: 'text-white' };
};

const fetchAllData = async (lat, lon, cityName, stateName) => {
    try {
        const [weatherRes, airRes] = await Promise.all([
            axios.get(
                `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&units=metric&appid=${apiKey}`,
            ),
            axios.get(
                `https://api.openweathermap.org/data/2.5/air_pollution?lat=${lat}&lon=${lon}&appid=${apiKey}`,
            ),
        ]);

        weatherData.value = weatherRes.data;
        airQuality.value = airRes.data.list[0].main.aqi;
        city.value = cityName;
        province.value = stateName || '';
        updateLocalTime();
    } catch (err) {
        alert('Error fetching weather details.');
    }
};

const fetchWeather = async () => {
    if (!city.value) return;
    loading.value = true;
    try {
        const geoRes = await axios.get(
            `https://api.openweathermap.org/geo/1.0/direct?q=${city.value}&limit=1&appid=${apiKey}`,
        );
        if (geoRes.data.length === 0) throw new Error('City not found');
        const { lat, lon, name, state } = geoRes.data[0];
        await fetchAllData(lat, lon, name, state);
    } catch (err) {
        alert(err.message);
    } finally {
        loading.value = false;
    }
};

const handleLocationClick = () => {
    if (!navigator.geolocation) return alert('GPS not supported');
    loading.value = true;
    navigator.geolocation.getCurrentPosition(async (pos) => {
        const { latitude, longitude } = pos.coords;
        try {
            const geoRes = await axios.get(
                `https://api.openweathermap.org/geo/1.0/reverse?lat=${latitude}&lon=${longitude}&limit=1&appid=${apiKey}`,
            );
            const { name, state } = geoRes.data[0] || {
                name: 'Current Location',
                state: '',
            };
            await fetchAllData(latitude, longitude, name, state);
        } catch (err) {
            alert('Error fetching location.');
        } finally {
            loading.value = false;
        }
    });
};

onMounted(() => {
    timer = setInterval(updateLocalTime, 60000);
});

onUnmounted(() => {
    clearInterval(timer);
});

const dailyForecast = computed(() => {
    return (
        weatherData.value?.list?.filter((_, i) => i % 8 === 0).slice(0, 5) || []
    );
});

const hourlyForecast = computed(() => {
    return weatherData.value?.list?.slice(0, 8) || [];
});
</script>
<template>
    <div
        :class="[
            'flex min-h-screen flex-col items-center bg-gradient-to-br p-6 font-sans text-white transition-all duration-1000',
            bgGradient,
        ]"
    >
        <component is="style">
            .glass-scroll::-webkit-scrollbar { height: 4px; }
            .glass-scroll::-webkit-scrollbar-track { background: rgba(255, 255,
            255, 0.1); border-radius: 10px; }
            .glass-scroll::-webkit-scrollbar-thumb { background: rgba(255, 255,
            255, 0.3); border-radius: 10px; }
            .glass-scroll::-webkit-scrollbar-thumb:hover { background: rgba(255,
            255, 255, 0.5); }
        </component>

        <div class="relative z-10 mt-4 mb-8 w-full max-w-2xl md:mt-10">
            <form @submit.prevent="fetchWeather" class="group relative">
                <input
                    v-model="city"
                    type="text"
                    placeholder="Search city..."
                    class="w-full rounded-2xl border border-white/30 bg-white/20 px-6 py-4 text-white placeholder-white/70 shadow-xl backdrop-blur-lg focus:ring-2 focus:ring-white/50 focus:outline-none"
                />
                <div class="absolute top-1.5 right-3 flex items-center gap-1">
                    <button
                        type="button"
                        @click="handleLocationClick"
                        class="cursor-pointer rounded-xl p-2 text-xl transition-colors hover:bg-white/20"
                    >
                        <i class="fa-solid fa-location-crosshairs"></i>
                    </button>
                    <button
                        type="submit"
                        class="cursor-pointer rounded-xl border border-white/20 bg-white/20 px-4 py-1.5 transition-all hover:bg-white/40"
                    >
                        {{ loading ? '...' : 'Search' }}
                    </button>
                </div>
            </form>
        </div>

        <div
            v-if="weatherData"
            class="relative z-10 w-full max-w-2xl space-y-6"
        >
            <div
                class="rounded-[2.5rem] border border-white/30 bg-white/10 p-8 shadow-2xl backdrop-blur-xl transition-transform hover:scale-[1.01] hover:bg-white/20"
            >
                <div
                    class="flex flex-col items-center justify-between md:flex-row"
                >
                    <div class="text-center md:text-left">
                        <div class="flex flex-col">
                            <div
                                class="flex items-center justify-center space-x-2 md:justify-start"
                            >
                                <h1
                                    class="text-5xl font-extrabold tracking-tight drop-shadow-md"
                                >
                                    {{ weatherData.city.name }}
                                </h1>
                                <span
                                    class="rounded-xl border border-white/20 bg-white/20 px-3 py-1 text-lg font-medium"
                                    >{{ weatherData.city.country }}</span
                                >
                            </div>
                            <p
                                v-if="province"
                                class="mt-1 text-lg font-semibold tracking-wide opacity-70"
                            >
                                {{ province }}
                            </p>
                        </div>
                        <p
                            class="mt-4 text-xl font-medium capitalize opacity-90"
                        >
                            {{ weatherData.list[0].weather[0].description }}
                        </p>

                        <div
                            class="mt-4 flex flex-col items-center md:items-start"
                        >
                            <div
                                class="flex gap-4 text-xs font-bold tracking-widest uppercase opacity-60"
                            >
                                <span
                                    >High:
                                    {{
                                        Math.round(
                                            weatherData.list[0].main.temp_max,
                                        )
                                    }}°</span
                                >
                                <span
                                    >Low:
                                    {{
                                        Math.round(
                                            weatherData.list[0].main.temp_min,
                                        )
                                    }}°</span
                                >
                            </div>
                            <div
                                class="mt-2 flex items-center gap-2 text-lg font-semibold text-white/90"
                            >
                                <i class="fa-regular fa-clock opacity-70"></i>
                                <span>{{ currentTime }}</span>
                            </div>
                        </div>
                    </div>

                    <div class="mt-6 flex flex-col items-center md:mt-0">
                        <img
                            :src="`https://openweathermap.org/img/wn/${weatherData.list[0].weather[0].icon}@4x.png`"
                            class="h-32 w-32 animate-pulse drop-shadow-2xl"
                        />
                        <p class="text-7xl font-thin drop-shadow-sm">
                            {{ Math.round(weatherData.list[0].main.temp) }}°
                        </p>
                    </div>
                </div>

                <div
                    class="mt-8 grid grid-cols-2 gap-4 border-t border-white/10 pt-6 md:grid-cols-4"
                >
                    <div
                        class="flex flex-col items-center rounded-2xl bg-white/5 px-2 py-3 backdrop-blur-sm"
                    >
                        <span
                            class="mb-1 text-[10px] font-black uppercase opacity-50"
                            >Feels Like</span
                        >
                        <div class="flex items-center gap-2">
                            <i
                                class="fa-solid fa-temperature-three-quarters text-sm text-orange-300"
                            ></i>
                            <span class="text-lg font-bold"
                                >{{
                                    Math.round(
                                        weatherData.list[0].main.feels_like,
                                    )
                                }}°</span
                            >
                        </div>
                    </div>
                    <div
                        class="flex flex-col items-center rounded-2xl bg-white/5 px-2 py-3 backdrop-blur-sm"
                    >
                        <span
                            class="mb-1 text-[10px] font-black uppercase opacity-50"
                            >Humidity</span
                        >
                        <div class="flex items-center gap-2">
                            <i
                                class="fa-solid fa-droplet text-sm text-cyan-300"
                            ></i>
                            <span class="text-lg font-bold"
                                >{{ weatherData.list[0].main.humidity }}%</span
                            >
                        </div>
                    </div>
                    <div
                        class="flex flex-col items-center rounded-2xl bg-white/5 px-2 py-3 backdrop-blur-sm"
                    >
                        <span
                            class="mb-1 text-[10px] font-black uppercase opacity-50"
                            >Air Quality</span
                        >
                        <div class="flex items-center gap-2">
                            <i
                                class="fa-solid fa-mask-face text-sm text-emerald-300"
                            ></i>
                            <span
                                :class="[
                                    'text-sm font-bold',
                                    getAQIDescription(airQuality).color,
                                ]"
                                >{{ getAQIDescription(airQuality).text }}</span
                            >
                        </div>
                    </div>
                    <div
                        class="flex flex-col items-center rounded-2xl bg-white/5 px-2 py-3 backdrop-blur-sm"
                    >
                        <span
                            class="mb-1 text-[10px] font-black uppercase opacity-50"
                            >Rain Chance</span
                        >
                        <div class="flex items-center gap-2">
                            <i
                                class="fa-solid fa-cloud-showers-heavy text-sm text-blue-300"
                            ></i>
                            <span class="text-lg font-bold"
                                >{{
                                    Math.round(weatherData.list[0].pop * 100)
                                }}%</span
                            >
                        </div>
                    </div>
                </div>
            </div>
            <div
                class="rounded-[2rem] border border-white/20 bg-white/10 p-6 backdrop-blur-xl hover:scale-[1.01] hover:bg-white/20"
            >
                <p
                    class="mb-4 text-[10px] font-black tracking-[0.2em] uppercase opacity-60"
                >
                    Next 24 Hours
                </p>
                <div class="glass-scroll flex gap-4 overflow-x-auto pb-4">
                    <div
                        v-for="(hour, i) in hourlyForecast"
                        :key="i"
                        class="min-w-[90px] rounded-2xl border border-white/5 bg-white/5 p-3 text-center"
                    >
                        <p class="text-[10px] font-bold opacity-70">
                            {{ new Date(hour.dt * 1000).getHours() }}:00
                        </p>
                        <img
                            :src="`https://openweathermap.org/img/wn/${hour.weather[0].icon}.png`"
                            class="mx-auto w-10"
                        />
                        <p class="text-lg font-bold">
                            {{ Math.round(hour.main.temp) }}°
                        </p>
                        <p class="text-[9px] font-bold text-blue-300">
                            <i class="fa-solid fa-droplet text-[8px]"></i>
                            {{ Math.round(hour.pop * 100) }}%
                        </p>
                    </div>
                </div>
            </div>

            <div
                class="rounded-[2.5rem] border border-white/20 bg-white/10 p-8 backdrop-blur-xl hover:scale-[1.01] hover:bg-white/20"
            >
                <p
                    class="mb-6 text-[10px] font-black tracking-[0.2em] uppercase opacity-60"
                >
                    5-Day Forecast
                </p>
                <div class="space-y-5">
                    <div
                        v-for="(day, i) in dailyForecast"
                        :key="i"
                        class="flex items-center justify-between border-b border-white/10 pb-4 last:border-0 last:pb-0"
                    >
                        <p class="w-20 text-sm font-bold">
                            {{
                                i === 0
                                    ? 'TODAY'
                                    : new Date(day.dt * 1000)
                                          .toLocaleDateString('en', {
                                              weekday: 'short',
                                          })
                                          .toUpperCase()
                            }}
                        </p>
                        <div
                            class="flex flex-1 items-center justify-center gap-6"
                        >
                            <div class="flex items-center gap-2">
                                <img
                                    :src="`https://openweathermap.org/img/wn/${day.weather[0].icon}.png`"
                                    class="w-10"
                                />
                                <span
                                    class="text-[10px] font-black uppercase opacity-50"
                                    >{{ day.weather[0].main }}</span
                                >
                            </div>
                            <span class="text-[10px] font-bold text-blue-300"
                                ><i class="fa-solid fa-umbrella"></i>
                                {{ Math.round(day.pop * 100) }}%</span
                            >
                        </div>
                        <div class="flex w-24 items-baseline justify-end gap-3">
                            <span class="text-xl font-bold"
                                >{{ Math.round(day.main.temp_max) }}°</span
                            >
                            <span class="text-sm font-medium opacity-40"
                                >{{ Math.round(day.main.temp_min) }}°</span
                            >
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div
            v-else
            class="mt-20 max-w-sm rounded-[2rem] border border-white/20 bg-white/10 p-10 text-center backdrop-blur-md"
        >
            <p class="text-2xl font-bold opacity-80">Welcome!</p>
            <p class="mt-2 text-sm opacity-60">
                Search for a city or use your GPS to see the weather, AQI, and
                forecast details.
            </p>
        </div>
    </div>
</template>
