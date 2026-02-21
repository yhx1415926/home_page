<template>
  <div class="weather" v-if="weatherData.adCode.city && weatherData.weather.weather">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{
        weatherData.weather.winddirection?.endsWith("风")
          ? weatherData.weather.winddirection
          : weatherData.weather.winddirection + "风"
      }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { getAdcode, getIpGeoLocation, getOtherWeather, getRegeoByLocation, getWeather } from "@/api";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY;
const geoWaitTimeout = 20000;

// 天气数据
const weatherData = reactive({
  adCode: {
    city: null, // 城市
    adcode: null, // 城市编码
  },
  weather: {
    weather: null, // 天气现象
    temperature: null, // 实时气温
    winddirection: null, // 风向描述
    windpower: null, // 风力级别
  },
});

// Promise 超时控制
const withTimeout = (promise, timeout, timeoutMessage) => {
  return Promise.race([
    promise,
    new Promise((_, reject) => {
      setTimeout(() => reject(timeoutMessage), timeout);
    }),
  ]);
};

// 取出天气平均值
const getTemperature = (min, max) => {
  try {
    // 计算平均值并四舍五入
    const average = (Number(min) + Number(max)) / 2;
    return Math.round(average);
  } catch (error) {
    console.error("计算温度出现错误：", error);
    return "NaN";
  }
};

// 查询高德天气并校验结果
const getValidatedWeather = async (cityOrAdcode) => {
  const result = await getWeather(mainKey, cityOrAdcode);
  if (!result?.lives?.[0]) {
    throw "天气信息为空";
  }
  return result.lives[0];
};

// 使用自定义 IP 服务解析城市信息
const resolveByIpGeoLocation = async () => {
  const geoData = await withTimeout(
    getIpGeoLocation(),
    geoWaitTimeout,
    "IP 地理位置接口请求超时（10s）",
  );

  const city = geoData?.cityName;
  const longitude = geoData?.longitude;
  const latitude = geoData?.latitude;

  if (longitude && latitude) {
    const regeo = await getRegeoByLocation(mainKey, longitude, latitude);
    if (regeo?.infocode === "10000" && regeo?.regeocode?.addressComponent?.adcode) {
      const addressComponent = regeo.regeocode.addressComponent;
      return {
        city: addressComponent.city || addressComponent.province || city || "未知地区",
        adcode: String(addressComponent.adcode),
      };
    }
  }

  if (!city) {
    throw "IP 地理位置缺少城市信息";
  }

  // 当无法拿到 adcode 时，使用城市名查询天气
  return {
    city,
    adcode: city,
  };
};

// 获取天气数据
const getWeatherData = async () => {
  try {
    // 获取地理位置信息
    if (!mainKey) {
      console.log("未配置，使用备用天气接口");
      const result = await getOtherWeather();
      const data = result.result;
      weatherData.adCode = {
        city: data.city.City || "未知地区",
      };
      weatherData.weather = {
        weather: data.condition.day_weather,
        temperature: getTemperature(data.condition.min_degree, data.condition.max_degree),
        winddirection: data.condition.day_wind_direction,
        windpower: data.condition.day_wind_power,
      };
      return;
    }

    let locationData = null;
    let weather = null;

    // 先尝试高德 IP 定位；有数据则不等待自定义接口
    const adCode = await getAdcode(mainKey);
    if (adCode?.infocode === "10000" && adCode?.adcode) {
      locationData = {
        city: adCode.city,
        adcode: adCode.adcode,
      };
      weatherData.adCode = locationData;

      try {
        weather = await getValidatedWeather(locationData.adcode);
      } catch (error) {
        console.warn("高德定位天气为空，改用自定义 IP 定位重试", error);
      }
    }

    // 当高德没有定位信息，或定位后的天气为空时，最多等待 10 秒使用自定义 IP 定位
    if (!weather) {
      locationData = await resolveByIpGeoLocation();
      weatherData.adCode = locationData;
      weather = await getValidatedWeather(locationData.adcode);
    }

    weatherData.weather = {
      weather: weather.weather,
      temperature: weather.temperature,
      winddirection: weather.winddirection,
      windpower: weather.windpower,
    };
  } catch (error) {
    console.error("天气信息获取失败:", error);
    onError("天气信息获取失败");
  }
};

// 报错信息
const onError = (message) => {
  ElMessage({
    message,
    icon: h(Error, {
      theme: "filled",
      fill: "#efefef",
    }),
  });
  console.error(message);
};

onMounted(() => {
  // 调用获取天气
  getWeatherData();
});
</script>
