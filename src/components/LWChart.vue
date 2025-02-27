<script>
import { createChart, LineStyle, LineType } from 'lightweight-charts';

let series;
let chart;
let averageSeries;
let volumeSeries;

function getChartSeriesConstructorName(type) {
	return `add${type.charAt(0).toUpperCase() + type.slice(1)}Series`;
}

const calculateAverageData = (data) => {
	if (!data || data.length === 0) return [];
	const average = data.reduce((sum, point) => sum + point.value, 0) / data.length;
	return data.map(point => ({ time: point.time, value: average }));
};

// const addSeriesAndData = (type, seriesOptions, data, volumeData) => {
// 	const seriesConstructor = getChartSeriesConstructorName(type);
// 	series = chart[seriesConstructor](seriesOptions);
// 	series.setData(data);

// 	const averageData = calculateAverageData(data);
// 	averageSeries = chart.addLineSeries({ color: 'red', lineWidth: 2 });
// 	averageSeries.setData(averageData);

// 	volumeSeries = chart.addHistogramSeries({ color: 'rgba(0, 150, 136, 0.5)', priceScaleId: '' });
// 	volumeSeries.setData(volumeData);
// };

const addSeriesAndData = (type, seriesOptions, data, volumeData) => {
	const seriesConstructor = getChartSeriesConstructorName(type);
	series = chart[seriesConstructor](seriesOptions);
	series.setData(data);

	// Thêm đường Average
	const averageData = calculateAverageData(data);
	averageSeries = chart.addLineSeries({ color: '#fff', lineWidth: 2, LineType: LineType.Dashed, LineStyle: LineStyle.Dashed });
	averageSeries.setData(averageData);

	// Thêm Volume Series với priceScale riêng
	volumeSeries = chart.addHistogramSeries({
		color: 'rgba(0, 150, 136, 0.5)',
		priceScaleId: 'volume', // Gán priceScale riêng cho Volume
	});

	volumeSeries.setData(volumeData);

	// Thêm priceScale bên trái cho Volume
	chart.priceScale('volume').applyOptions({
		position: 'left', // Hiển thị cột giá trị volume bên trái
		scaleMargins: {
			top: 0.8, // Để biểu đồ volume nhỏ hơn phía dưới
			bottom: 0,
		},
		borderVisible: false, // Ẩn đường viền giữa giá & volume
	});
};
const createTooltip = () => {
	const tooltip = document.createElement('div');
	tooltip.style.position = 'absolute';
	tooltip.style.background = 'rgba(0, 0, 0, 0.8)';
	tooltip.style.color = '#ffffff';
	tooltip.style.padding = '8px';
	tooltip.style.borderRadius = '4px';
	tooltip.style.display = 'none';
	tooltip.style.pointerEvents = 'none';
	tooltip.style.fontSize = '12px';
	tooltip.style.zIndex = '999';
	document.body.appendChild(tooltip);
	return tooltip;
};

const tooltip = createTooltip();

const handleCrosshairMove = (param) => {
	if (!param || !param.time || !series || !volumeSeries) {
		tooltip.style.display = 'none';
		return;
	}

	// Lấy giá trị price và volume từ seriesData
	const priceDataPoint = param.seriesPrices?.get(series) || {};
	const volumeDataPoint = param.seriesPrices?.get(volumeSeries) || {};

	// Kiểm tra nếu không có giá trị hợp lệ thì ẩn tooltip
	if (priceDataPoint === undefined || volumeDataPoint === undefined) {
		tooltip.style.display = 'none';
		return;
	}

	tooltip.style.display = 'block';
	tooltip.innerHTML = `
		<b>Date:</b> ${param.time ? new Date(param.time).toISOString().split('T')[0] : ''}<br>
		<b>Price:</b> ${priceDataPoint.toFixed(2)}<br>
		<b>Volume:</b> ${volumeDataPoint}
	`;
	// Định vị tooltip theo vị trí chuột
	const { x, y } = param.point;
	tooltip.style.left = `${x + 10}px`;
	tooltip.style.top = `${y + 10}px`;
};


const resizeHandler = container => {
	if (!chart || !container) return;
	const dimensions = container.getBoundingClientRect();
	chart.resize(dimensions.width, dimensions.height);
};

export default {
	props: {
		type: {
			type: String,
			default: 'baseline',
		},
		data: {
			type: Array,
			required: true,
		},
		volumeData: {
			type: Array,
			required: true,
		},
		autosize: {
			default: true,
			type: Boolean,
		},
		chartOptions: {
			type: Object,
		},
		seriesOptions: {
			type: Object,
		},
		timeScaleOptions: {
			type: Object,
		},
		priceScaleOptions: {
			type: Object,
		},
	},
	mounted() {
		chart = createChart(this.$refs.chartContainer, { 
			layout: { background: { color: '#121212' }, textColor: '#ffffff' },
			grid: { vertLines: { color: '#333' }, horzLines: { color: '#333' }, borderColor: '#333' },
			...this.chartOptions 
		});
		addSeriesAndData(this.type, this.seriesOptions, this.data, this.volumeData);

		if (this.priceScaleOptions) {
			chart.priceScale().applyOptions(this.priceScaleOptions);
		}

		if (this.timeScaleOptions) {
			chart.timeScale().applyOptions(this.timeScaleOptions);
		}

		chart.timeScale().fitContent();

		if (this.autosize) {
			window.addEventListener('resize', () => resizeHandler(this.$refs.chartContainer));
		}
		chart.subscribeCrosshairMove(handleCrosshairMove);
	},
	unmounted() {
		if (chart) {
			chart.remove();
			chart = null;
		}
		if (series) {
			series = null;
		}
		if (averageSeries) {
			averageSeries = null;
		}
		if (volumeSeries) {
			volumeSeries = null;
		}
		window.removeEventListener('resize', resizeHandler);

		chart.unsubscribeCrosshairMove(handleCrosshairMove);
		document.body.removeChild(tooltip);

	},
	watch: {
		data(newData) {
			if (!series || !averageSeries || !volumeSeries) return;
			series.setData(newData);
			averageSeries.setData(calculateAverageData(newData));
		},
		volumeData(newData) {
			if (!volumeSeries) return;
			volumeSeries.setData(newData);
		},
	},
	methods: {
		fitContent() {
			if (!chart) return;
			chart.timeScale().fitContent();
		},
		getChart() {
			return chart;
		},
	},
	expose: ['fitContent', 'getChart'],
};
</script>

<template>
	<div class="lw-chart" ref="chartContainer" id="chart"></div>
</template>

<style scoped>
.lw-chart {
	height: 100%;
	background: #121212;
}
</style>
