<template>
	<view class="sleep-container" :class="{'dark-theme': isDarkTheme}">
		<view class="page-header">
			<text class="page-title">睡眠记录</text>
			<view class="theme-switch" @tap="toggleTheme">
				<text class="theme-icon">{{ isDarkTheme ? '🌞' : '🌙' }}</text>
			</view>
		</view>
		<!-- 顶部状态卡片 -->
		<view class="status-card">
			<view class="status-header">
				<text class="status-title">睡眠状态</text>
				<!-- <view class="header-right">
					<view class="theme-switch" @click="toggleTheme">
						<text class="theme-icon">{{ isDarkTheme ? '🌙' : '☀️' }}</text>
					</view>
				</view> -->
			</view>
			<view class="status-content">
				<view class="status-item">
					<text class="status-label">睡眠时长</text>
					<text class="status-value">9:06</text>
				</view>
				<view class="status-item">
					<text class="status-label">入睡时间</text>
					<text class="status-value">23:09</text>
				</view>
				<view class="status-item">
					<text class="status-label">醒来时间</text>
					<text class="status-value">08:15</text>
				</view>
			</view>
		</view>
		
		<!-- 数据图表区域 -->
		<view class="charts-container">
			<view class="chart-item">
				<text class="chart-title">心率变化</text>
				<web-view :src="chartUrl" @message="handleChartMessage" :id="'heartRateChart'" class="chart-webview"></web-view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">呼吸频率</text>
				<web-view :src="chartUrl" @message="handleChartMessage" :id="'breathingRateChart'" class="chart-webview"></web-view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">体温变化</text>
				<web-view :src="chartUrl" @message="handleChartMessage" :id="'temperatureChart'" class="chart-webview"></web-view>
			</view>

			<!-- 鼾声图表 -->
			<view class="chart-item">
				<text class="chart-title">鼾声监测</text>
				<web-view :src="chartUrl" @message="handleChartMessage" :id="'snoreChart'" class="chart-webview"></web-view>
			</view>
		</view>

	</view>
</template>

<script>
	import * as echarts from 'echarts';
	import AudioRecorder from '@/components/audio-recorder/audio-recorder.vue';
	
	export default {
		components: {
			AudioRecorder
		},
		data() {
			return {
				isDarkTheme: false,
				currentTime: '',
				chartUrl: '/pages/sleep/chart.html',
				chartData: {
					heartRate: {
						times: [],
						values: [],
						color: '#1890FF',
						name: '心率',
						unit: '次/分'
					},
					breathingRate: {
						times: [],
						values: [],
						color: '#91CB74',
						name: '呼吸频率',
						unit: '次/分'
					},
					temperature: {
						times: [],
						values: [],
						color: '#FAC858',
						name: '体温',
						unit: '°C'
					},
					snore: {
						times: [],
						values: [],
						color: '#EE6666',
						name: '鼾声',
						unit: 'dB'
					}
				},
				timer: null,
				totalHours: 24,
				displayHours: 8,
				startTimeIndex: 0,
				chartWebviews: {}
			}
		},
		onLoad() {
			// 从本地存储读取主题设置
			const theme = uni.getStorageSync('theme');
			this.isDarkTheme = theme === 'dark';
			
			// 更新时间
			this.updateTime();
			this.timer = setInterval(this.updateTime, 1000);
			
			// 加载数据
			this.loadData();
		},
		onReady() {
			// 等待webview加载完成
			setTimeout(() => {
				this.initCharts();
			}, 1000);
		},
		methods: {
			toggleTheme() {
				this.isDarkTheme = !this.isDarkTheme;
				uni.setStorageSync('theme', this.isDarkTheme ? 'dark' : 'light');
				this.updateChartsTheme();
			},
			
			updateChartsTheme() {
				// 向所有图表webview发送主题更新消息
				Object.keys(this.chartWebviews).forEach(id => {
					const webview = this.chartWebviews[id];
					if (webview) {
						webview.postMessage({
							data: {
								isDark: this.isDarkTheme
							},
							type: 'themeChange'
						});
					}
				});
			},
			
			handleChartMessage(event) {
				const { id } = event.target;
				this.chartWebviews[id] = event.target;
				
				// 发送初始数据
				const chartType = id.replace('Chart', '');
				const data = this.chartData[chartType];
				if (data) {
					event.target.postMessage({
						data: {
							...data,
							type: chartType
						},
						type: 'dataUpdate'
					});
				}
			},
			
			updateTime() {
				const now = new Date();
				this.currentTime = `${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`;
			},
			
			loadData() {
				// 生成模拟数据
				const generateData = (base, range, count) => {
					return Array.from({length: count}, () => 
						Math.round((base + (Math.random() - 0.5) * range) * 10) / 10
					);
				};
				
				// 生成24小时的时间点，每15分钟一个数据点
				const timePoints = Array.from({length: this.totalHours * 4}, (_, i) => {
					const hour = Math.floor(i / 4);
					const minute = (i % 4) * 15;
					return `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`;
				});
				
				// 更新所有图表数据
				this.chartData.heartRate.times = timePoints;
				this.chartData.heartRate.values = generateData(75, 10, this.totalHours * 4);
				
				this.chartData.breathingRate.times = timePoints;
				this.chartData.breathingRate.values = generateData(16, 4, this.totalHours * 4);
				
				this.chartData.temperature.times = timePoints;
				this.chartData.temperature.values = generateData(36.5, 0.5, this.totalHours * 4);
				
				this.chartData.snore.times = timePoints;
				this.chartData.snore.values = generateData(30, 15, this.totalHours * 4);
				
				// 更新图表
				this.updateCharts();
			},
			
			updateCharts() {
				// 向所有图表webview发送数据更新消息
				Object.entries(this.chartWebviews).forEach(([id, webview]) => {
					if (webview) {
						const chartType = id.replace('Chart', '');
						const data = this.chartData[chartType];
						if (data) {
							webview.postMessage({
								data: {
									...data,
									type: chartType
								},
								type: 'dataUpdate'
							});
						}
					}
				});
			},
			
			initCharts() {
				// 图表初始化由webview处理
				this.updateChartsTheme();
				this.updateCharts();
			}
		},
		onUnload() {
			// 清除定时器
			if (this.timer) {
				clearInterval(this.timer);
			}
		}
	}
</script>

<style>
	.sleep-container {
		min-height: 100vh;
		background-color: #f5f5f5;
		transition: all 0.3s ease;
		position: relative;
	}
	
	.sleep-container.dark-theme {
		background-color: #121212;
	}
	
	.page-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 20rpx 30rpx;
		background-color: #ffffff;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
		position: relative;
		z-index: 100;
	}
	
	.sleep-container.dark-theme .page-header {
		background-color: #2c2c2c;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.2);
	}
	
	.page-title {
		font-size: 36rpx;
		font-weight: bold;
		color: #333;
		position: relative;
		z-index: 1;
	}
	
	.sleep-container.dark-theme .page-title {
		color: #ffffff;
	}
	
	.theme-switch {
		width: 80rpx;
		height: 80rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		background-color: #f5f5f5;
		border-radius: 50%;
		cursor: pointer;
		transition: background-color 0.3s ease;
		position: relative;
		z-index: 101;
	}
	
	.theme-switch::after {
		content: '';
		position: absolute;
		top: -10rpx;
		left: -10rpx;
		right: -10rpx;
		bottom: -10rpx;
		z-index: -1;
	}
	
	.sleep-container.dark-theme .theme-switch {
		background-color: #3a3a3a;
	}
	
	.theme-icon {
		font-size: 40rpx;
		position: relative;
		z-index: 1;
	}
	
	.status-card {
		background-color: #ffffff;
		border-radius: 20rpx;
		padding: 20rpx;
		margin: 20rpx;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
		position: relative;
		z-index: 1;
	}
	
	.sleep-container.dark-theme .status-card {
		background-color: #2c2c2c;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
	}
	
	.status-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 30rpx;
	}
	
	.status-title {
		font-size: 32rpx;
		color: #333;
		font-weight: bold;
	}
	
	.sleep-container.dark-theme .status-title {
		color: #ffffff;
	}
	
	.status-time {
		font-size: 28rpx;
		opacity: 0.8;
	}
	
	.status-content {
		display: flex;
		justify-content: space-between;
	}
	
	.status-item {
		text-align: center;
		flex: 1;
	}
	
	.status-label {
		font-size: 24rpx;
		color: #666;
	}
	
	.sleep-container.dark-theme .status-label {
		color: #999;
	}
	
	.status-value {
		font-size: 28rpx;
		color: #333;
		font-weight: bold;
	}
	
	.sleep-container.dark-theme .status-value {
		color: #ffffff;
	}
	
	.charts-container {
		position: relative;
		z-index: 1;
		padding: 20rpx;
	}
	
	.chart-item {
		background-color: #ffffff;
		border-radius: 20rpx;
		padding: 20rpx;
		margin-bottom: 20rpx;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
		height: 660rpx;
		display: flex;
		flex-direction: column;
		position: relative;
		z-index: 1;
	}
	
	.sleep-container.dark-theme .chart-item {
		background-color: #2c2c2c;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
	}
	
	.chart-item:active {
		transform: scale(0.98);
	}
	
	.chart-title {
		font-size: 28rpx;
		color: #333;
		margin-bottom: 20rpx;
		padding: 0 10rpx;
	}
	
	.sleep-container.dark-theme .chart-title {
		color: #ffffff;
	}
	
	.chart-box {
		width: 100%;
		height: 35vw;
		max-height: 350rpx;
		min-height: 280rpx;
		position: relative;
	}
	
	.echarts {
		width: 100%;
		height: 100%;
		cursor: grab;
		touch-action: pan-x pan-y;
		-webkit-tap-highlight-color: transparent;
		user-select: none;
		-webkit-user-select: none;
	}
	
	.echarts:active {
		cursor: grabbing;
	}
	
	.recorder-section {
		margin: 30rpx 0;
		display: flex;
		flex-direction: column;
		margin-bottom: 30rpx;
	}
	
	.snore-chart {
		margin-bottom: 0;
	}
	
	.snore-chart .chart-box {
		height: 30vw;
		max-height: 300rpx;
		min-height: 240rpx;
	}
	
	/* 移除之前的深色模式媒体查询 */
	@media (prefers-color-scheme: dark) {
		.sleep-container {
			background-color: #f8f8f8;
		}
		
		.chart-item {
			background-color: #fff;
			box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.08);
		}
		
		.chart-title {
			color: #666;
		}
	}
	
	.chart-webview {
		width: 100%;
		height: 600rpx;
		margin-bottom: 20rpx;
		position: relative;
		z-index: 1;
	}
</style> 