<script setup>
import { ref, onMounted } from "vue";

import LWChart from "./components/LWChart.vue";
import { createChart } from "lightweight-charts";

/**
 * Generates sample data for the lightweight chart
 * @param  {Boolean} ohlc Whether generated dat should include open, high, low, and close values
 * @returns {Array} sample data
 */

const seriesOptions = ref({
  color: "rgb(45, 77, 205)",
});

function generateSampleData(ohlc) {
  const randomFactor = 25 + Math.random() * 25;
  function samplePoint(i) {
    return (
      i *
        (0.5 +
          Math.sin(i / 10) * 0.2 +
          Math.sin(i / 20) * 0.4 +
          Math.sin(i / randomFactor) * 0.8 +
          Math.sin(i / 500) * 0.5) +
      200
    );
  }

  const res = [];
  let date = new Date(Date.UTC(2018, 0, 1, 0, 0, 0, 0));
  const numberOfPoints = ohlc ? 100 : 500;
  for (var i = 0; i < numberOfPoints; ++i) {
    const time = date.getTime() / 1000;
    const value = samplePoint(i);
    if (ohlc) {
      const randomRanges = [-1 * Math.random(), Math.random(), Math.random()].map((i) => i * 10);
      const sign = Math.sin(Math.random() - 0.5);
      res.push({
        time,
        low: value + randomRanges[0],
        high: value + randomRanges[1],
        open: value + sign * randomRanges[2],
        close: samplePoint(i + 1),
      });
    } else {
      res.push({
        time,
        value,
      });
    }

    date.setUTCDate(date.getUTCDate() + 1);
  }
  const average =
    res.reduce((s, c) => {
      return s + c.value;
    }, 0) / res.length;
  seriesOptions.value = { baseValue: { type: "price", price: average } };

  return res;
}
const generateVolumeData = (priceData, minVolume = 10, maxVolume = 100) => {
	return priceData.map(point => ({
		time: point.time, // Giữ nguyên ngày tháng từ priceData
		value: Math.floor(Math.random() * (maxVolume - minVolume + 1)) + minVolume
	}));
};
const chartOptions = ref({
  backgroundColor: "#121212",
});
const data = ref(generateSampleData(false));

const chartType = ref("baseline");
const lwChart = ref();
const volumeData = generateVolumeData(data.value);
console.log(volumeData)
</script>

<template>
  <div class="chart-container">
    <LWChart
      :type="chartType"
      :data="data"
      :volumeData="volumeData"
      :autosize="true"
      :chart-options="chartOptions"
      :series-options="seriesOptions"
      ref="lwChart"
    />
  </div>
</template>
<style scoped>
.chart-container {
  height: calc(100% - 3.2em);
  width: calc(100% - 5em);
  margin-top: 2.5em;
  margin-left: auto;
  margin-right: auto;
}
</style>
