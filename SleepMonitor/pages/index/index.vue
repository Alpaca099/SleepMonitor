<template>
	<view class="content">
		<view class="header">
			<text class="title">睡眠监测数据</text>
		</view>
		
		<view class="charts-container">
			<view class="chart-item">
				<text class="chart-title">心率数据 (次/分)</text>
				<view class="chart-box">
					<view class="echarts" id="heartRateChart"></view>
				</view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">血氧数据 (%)</text>
				<view class="chart-box">
					<view class="echarts" id="oxygenChart"></view>
				</view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">温度数据 (°C)</text>
				<view class="chart-box">
					<view class="echarts" id="temperatureChart"></view>
				</view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">鼾声数据 (dB)</text>
				<view class="chart-box">
					<view class="echarts" id="snoreChart"></view>
				</view>
			</view>
		</view>
		
		<view class="nav-button" @click="goToConsult">
			<text>睡眠咨询</text>
		</view>
	</view>
</template>

<script>
	import * as echarts from 'echarts';
	
	export default {
		data() {
			return {
				charts: {},
				chartData: {
					heartRate: {
						times: [],
						values: []
					},
					oxygen: {
						times: [],
						values: []
					},
					temperature: {
						times: [],
						values: []
					},
					snore: {
						times: [],
						values: []
					}
				}
			}
		},
		onLoad() {
			// 先加载数据
			this.loadData();
		},
		onReady() {
			// 确保DOM已经渲染完成后再初始化图表
			setTimeout(() => {
				this.initCharts();
				// 初始化后立即更新数据
				this.updateCharts();
			}, 300);
		},
		methods: {
			loadData() {
				// 生成更真实的模拟数据
				const generateData = (base, range, count) => {
					return Array.from({length: count}, () => 
						Math.round((base + (Math.random() - 0.5) * range) * 10) / 10
					);
				};
				
				const timePoints = Array.from({length: 24}, (_, i) => 
					`${String(i).padStart(2, '0')}:00`
				);
				
				// 更新数据
				this.chartData.heartRate = {
					times: timePoints,
					values: generateData(75, 10, 24) // 心率在65-85之间波动
				};
				
				this.chartData.oxygen = {
					times: timePoints,
					values: generateData(98, 2, 24) // 血氧在97-99之间波动
				};
				
				this.chartData.temperature = {
					times: timePoints,
					values: generateData(36.5, 0.5, 24) // 体温在36.2-36.7之间波动
				};
				
				this.chartData.snore = {
					times: timePoints,
					values: generateData(30, 15, 24) // 鼾声在22.5-37.5之间波动
				};
			},
			
			initCharts() {
				const chartConfig = {
					grid: {
						top: 40,
						right: 20,
						bottom: 40,
						left: 60,
						containLabel: true
					},
					tooltip: {
						trigger: 'axis',
						axisPointer: {
							type: 'cross',
							label: {
								backgroundColor: '#6a7985'
							}
						}
					},
					xAxis: {
						type: 'category',
						boundaryGap: false,
						data: [],
						axisLine: {
							lineStyle: {
								color: '#999'
							}
						},
						axisLabel: {
							color: '#666',
							rotate: 45
						}
					},
					yAxis: {
						type: 'value',
						axisLine: {
							show: true,
							lineStyle: {
								color: '#999'
							}
						},
						axisLabel: {
							color: '#666'
						},
						splitLine: {
							lineStyle: {
								type: 'dashed',
								color: '#ddd'
							}
						}
					},
					series: [{
						type: 'line',
						smooth: true,
						symbol: 'circle',
						symbolSize: 6,
						data: [],
						areaStyle: {
							opacity: 0.1
						},
						lineStyle: {
							width: 2
						}
					}]
				};
				
				// 初始化四个图表
				const initChart = (id, name, color) => {
					const chart = echarts.init(document.getElementById(id));
					chart.setOption({
						...chartConfig,
						series: [{
							...chartConfig.series[0],
							name: name,
							itemStyle: {
								color: color
							}
						}]
					});
					return chart;
				};
				
				this.charts.heartRate = initChart('heartRateChart', '心率', '#1890FF');
				this.charts.oxygen = initChart('oxygenChart', '血氧', '#91CB74');
				this.charts.temperature = initChart('temperatureChart', '温度', '#FAC858');
				this.charts.snore = initChart('snoreChart', '鼾声', '#EE6666');
				
				// 监听窗口大小变化，调整图表大小
				window.addEventListener('resize', this.resizeCharts);
			},
			
			updateCharts() {
				// 更新心率图表
				this.charts.heartRate.setOption({
					xAxis: {
						data: this.chartData.heartRate.times
					},
					series: [{
						data: this.chartData.heartRate.values
					}]
				});
				
				// 更新血氧图表
				this.charts.oxygen.setOption({
					xAxis: {
						data: this.chartData.oxygen.times
					},
					series: [{
						data: this.chartData.oxygen.values
					}]
				});
				
				// 更新温度图表
				this.charts.temperature.setOption({
					xAxis: {
						data: this.chartData.temperature.times
					},
					series: [{
						data: this.chartData.temperature.values
					}]
				});
				
				// 更新鼾声图表
				this.charts.snore.setOption({
					xAxis: {
						data: this.chartData.snore.times
					},
					series: [{
						data: this.chartData.snore.values
					}]
				});
			},
			
			resizeCharts() {
				Object.values(this.charts).forEach(chart => {
					chart && chart.resize();
				});
			},
			
			goToConsult() {
				uni.navigateTo({
					url: '/pages/consult/consult'
				});
			}
		},
		onUnload() {
			// 页面卸载时移除事件监听
			window.removeEventListener('resize', this.resizeCharts);
			// 销毁图表实例
			Object.values(this.charts).forEach(chart => {
				chart && chart.dispose();
			});
		}
	}
</script>

<style>
	.content {
		padding: 20rpx;
	}
	
	.header {
		text-align: center;
		padding: 20rpx 0;
	}
	
	.title {
		font-size: 36rpx;
		font-weight: bold;
		color: #333;
	}
	
	.charts-container {
		display: flex;
		flex-direction: column;
		gap: 30rpx;
	}
	
	.chart-item {
		background-color: #fff;
		border-radius: 12rpx;
		padding: 20rpx;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
	}
	
	.chart-title {
		font-size: 28rpx;
		color: #666;
		margin-bottom: 20rpx;
	}
	
	.chart-box {
		width: 100%;
		height: 400rpx;
	}
	
	.echarts {
		width: 100%;
		height: 100%;
	}
	
	.nav-button {
		position: fixed;
		bottom: 40rpx;
		left: 50%;
		transform: translateX(-50%);
		background-color: #007AFF;
		color: #fff;
		padding: 20rpx 60rpx;
		border-radius: 40rpx;
		font-size: 32rpx;
		box-shadow: 0 4rpx 12rpx rgba(0,122,255,0.3);
	}
</style>
