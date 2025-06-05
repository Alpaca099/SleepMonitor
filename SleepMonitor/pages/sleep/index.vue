<template>
	<view class="sleep-container" :class="{ 'dark-theme': isDarkTheme }">
		<!-- 顶部状态卡片 -->
		<view class="status-card">
			<view class="status-header">
				<text class="status-title">睡眠状态</text>
				<view class="header-right">
					<view class="theme-switch" @click="toggleTheme">
						<text class="theme-icon">{{ isDarkTheme ? '🌙' : '☀️' }}</text>
					</view>
				</view>
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
				<view class="chart-box">
					<view class="echarts" id="heartRateChart"></view>
				</view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">呼吸频率</text>
				<view class="chart-box">
					<view class="echarts" id="breathingRateChart"></view>
				</view>
			</view>
			
			<view class="chart-item">
				<text class="chart-title">体温变化</text>
				<view class="chart-box">
					<view class="echarts" id="temperatureChart"></view>
				</view>
			</view>

			<!-- 鼾声图表 -->
			<view class="chart-item">
				<text class="chart-title">鼾声监测</text>
				<view class="chart-box">
					<view class="echarts" id="snoreChart"></view>
				</view>
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
				charts: {},
				chartData: {
					heartRate: {
						times: [],
						values: []
					},
					breathingRate: {
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
				},
				timer: null,
				totalHours: 24,
				displayHours: 8,
				startTimeIndex: 0
			}
		},
		onLoad() {
			// 更新时间
			this.updateTime();
			this.timer = setInterval(this.updateTime, 1000);
			
			// 加载数据
			this.loadData();
		},
		onReady() {
			// 确保DOM已经渲染完成后再初始化图表
			setTimeout(() => {
				// 添加移动端触摸样式
				const style = document.createElement('style');
				style.textContent = `
					.echarts {
						-webkit-tap-highlight-color: transparent;
						touch-action: pan-x pan-y;
						user-select: none;
						-webkit-user-select: none;
					}
				`;
				document.head.appendChild(style);
				
				this.initCharts();
				// 初始化后立即更新数据
				this.updateCharts();
			}, 300);
		},
		methods: {
			toggleTheme() {
				this.isDarkTheme = !this.isDarkTheme;
				this.updateChartsTheme();
			},
			
			updateChartsTheme() {
				const theme = this.isDarkTheme ? 'dark' : 'light';
				const textColor = this.isDarkTheme ? '#fff' : '#333';
				const gridColor = this.isDarkTheme ? 'rgba(255,255,255,0.1)' : 'rgba(0,0,0,0.08)';
				const splitLineColor = this.isDarkTheme ? 'rgba(255,255,255,0.05)' : 'rgba(0,0,0,0.03)';
				const splitAreaColor = this.isDarkTheme 
					? ['rgba(255,255,255,0.02)', 'rgba(255,255,255,0.04)']
					: ['rgba(0,0,0,0.01)', 'rgba(0,0,0,0.02)'];
				
				Object.entries(this.charts).forEach(([type, chart]) => {
					if (!chart) return;
					
					chart.setOption({
						title: {
							textStyle: {
								color: textColor
							}
						},
						dataZoom: [{
							type: 'inside',
							start: 0,
							end: (this.displayHours * 4) / (this.totalHours * 4) * 100,
							zoomOnMouseWheel: false,
							moveOnMouseMove: true,
							moveOnMouseWheel: false,
							preventDefaultMouseMove: true,
							throttle: 0,
							rangeMode: ['value', 'value'],
							filterMode: 'filter',
							zoomLock: true,
							minSpan: (this.displayHours * 4) / (this.totalHours * 4) * 100,
							maxSpan: (this.displayHours * 4) / (this.totalHours * 4) * 100
						}],
						xAxis: {
							axisLine: {
								lineStyle: {
									color: gridColor
								}
							},
							axisLabel: {
								color: textColor
							},
							axisTick: {
								lineStyle: {
									color: gridColor
								}
							},
							splitLine: {
								lineStyle: {
									color: splitLineColor
								}
							},
							splitArea: {
								areaStyle: {
									color: splitAreaColor
								}
							}
						},
						yAxis: {
							axisLine: {
								lineStyle: {
									color: gridColor
								}
							},
							axisLabel: {
								color: textColor
							},
							axisTick: {
								lineStyle: {
									color: gridColor
								}
							},
							splitLine: {
								lineStyle: {
									color: splitLineColor
								}
							},
							splitArea: {
								areaStyle: {
									color: splitAreaColor
								}
							}
						}
					});
				});
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
				
				// 生成所有数据
				this.chartData = {
					heartRate: {
						times: timePoints,
						values: generateData(75, 10, this.totalHours * 4)
					},
					breathingRate: {
						times: timePoints,
						values: generateData(16, 4, this.totalHours * 4)
					},
					temperature: {
						times: timePoints,
						values: generateData(36.5, 0.5, this.totalHours * 4)
					},
					snore: {
						times: timePoints,
						values: generateData(30, 15, this.totalHours * 4)
					}
				};
			},
			
			initCharts() {
				const chartConfig = {
					grid: {
						top: 35,
						right: 5,
						bottom: 5,
						left: 5,
						containLabel: true
					},
					dataZoom: [{
						type: 'inside',
						start: 0,
						end: (this.displayHours * 4) / (this.totalHours * 4) * 100,
						zoomOnMouseWheel: false,
						moveOnMouseMove: true,
						moveOnMouseWheel: false,
						preventDefaultMouseMove: true,
						throttle: 0,
						rangeMode: ['value', 'value'],
						filterMode: 'filter',
						zoomLock: true,
						minSpan: (this.displayHours * 4) / (this.totalHours * 4) * 100,
						maxSpan: (this.displayHours * 4) / (this.totalHours * 4) * 100
					}],
					tooltip: {
						trigger: 'axis',
						axisPointer: {
							type: 'line',
							lineStyle: {
								color: 'rgba(0,0,0,0.1)',
								width: 1,
								type: 'solid'
							}
						},
						backgroundColor: 'rgba(255,255,255,0.9)',
						borderColor: 'rgba(0,0,0,0.1)',
						borderWidth: 1,
						textStyle: {
							color: '#666',
							fontSize: 12
						},
						padding: [8, 12]
					},
					xAxis: {
						type: 'category',
						boundaryGap: true,
						data: [],
						axisLine: {
							show: true,
							lineStyle: {
								color: 'rgba(0,0,0,0.08)',
								width: 1
							}
						},
						axisLabel: {
							color: '#999',
							fontSize: 10,
							rotate: 0,
							interval: 'auto',
							formatter: (value) => {
								return value.split(':')[0] + '时';
							}
						},
						axisTick: {
							show: true,
							alignWithLabel: true,
							lineStyle: {
								color: 'rgba(0,0,0,0.08)',
								width: 1
							}
						},
						splitLine: {
							show: true,
							lineStyle: {
								color: 'rgba(0,0,0,0.03)',
								type: 'dashed',
								width: 1
							}
						}
					},
					yAxis: {
						type: 'value',
						axisLine: {
							show: true,
							lineStyle: {
								color: 'rgba(0,0,0,0.08)',
								width: 1
							}
						},
						axisLabel: {
							color: '#999',
							fontSize: 10,
							margin: 8
						},
						axisTick: {
							show: true,
							lineStyle: {
								color: 'rgba(0,0,0,0.08)',
								width: 1
							}
						},
						splitLine: {
							show: true,
							lineStyle: {
								type: 'dashed',
								color: 'rgba(0,0,0,0.03)',
								width: 1
							}
						}
					},
					series: [{
						type: 'line',
						smooth: true,
						symbol: 'circle',
						symbolSize: 4,
						showSymbol: false,
						data: [],
						areaStyle: {
							opacity: 0.15,
							color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{
								offset: 0,
								color: 'rgba(0,0,0,0.2)'
							}, {
								offset: 1,
								color: 'rgba(0,0,0,0.05)'
							}])
						},
						lineStyle: {
							width: 2,
							shadowColor: 'rgba(0,0,0,0.1)',
							shadowBlur: 4
						},
						emphasis: {
							focus: 'series',
							itemStyle: {
								borderWidth: 2
							}
						},
						animation: false,
						zlevel: 1,
						z: 1
					}]
				};
				
				// 初始化图表
				const initChart = (id, name, color) => {
					const dom = document.getElementById(id);
					if (!dom) {
						console.error(`找不到图表容器: ${id}`);
						return null;
					}
					
					const chart = echarts.init(dom, null, {
						renderer: 'canvas',
						useDirtyRect: false,
						devicePixelRatio: window.devicePixelRatio
					});
					
					chart.setOption({
						...chartConfig,
						series: [{
							...chartConfig.series[0],
							name: name,
							itemStyle: {
								color: color,
								borderColor: '#fff',
								borderWidth: 1
							},
							areaStyle: {
								...chartConfig.series[0].areaStyle,
								color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{
									offset: 0,
									color: color.replace(')', ', 0.2)').replace('rgb', 'rgba')
								}, {
									offset: 1,
									color: color.replace(')', ', 0.05)').replace('rgb', 'rgba')
								}])
							}
						}]
					});
					
					return chart;
				};
				
				this.charts.heartRate = initChart('heartRateChart', '心率', '#1890FF');
				this.charts.breathingRate = initChart('breathingRateChart', '呼吸', '#91CB74');
				this.charts.temperature = initChart('temperatureChart', '体温', '#FAC858');
				this.charts.snore = initChart('snoreChart', '鼾声', '#EE6666');
				
				// 监听窗口大小变化
				window.addEventListener('resize', this.resizeCharts);
			},
			
			updateCharts() {
				Object.entries(this.charts).forEach(([type, chart]) => {
					if (!chart) return;
					
					chart.setOption({
						xAxis: {
							data: this.chartData[type].times
						},
						series: [{
							data: this.chartData[type].values
						}]
					});
				});
			},
			
			resizeCharts() {
				Object.values(this.charts).forEach(chart => {
					chart && chart.resize();
				});
			},
			
			handleRecordingComplete(recording) {
				console.log('录音完成:', recording);
				this.lastRecording = recording;
			},
			
			// 添加新数据点（不再更新范围）
			addNewData(type, value) {
				// 更新图表数据
				const now = new Date();
				const time = `${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`;
				
				this.chartData[type].times.push(time);
				this.chartData[type].values.push(value);
				
				// 保持最近24小时的数据
				if (this.chartData[type].times.length > this.totalHours) {
					this.chartData[type].times.shift();
					this.chartData[type].values.shift();
				}
				
				// 更新图表（使用固定的范围）
				this.charts[type].setOption({
					xAxis: {
						data: this.chartData[type].times
					},
					series: [{
						data: this.chartData[type].values
					}]
				});
			},
			formatTime(timestamp) {
				const date = new Date(timestamp);
				return `${String(date.getHours()).padStart(2, '0')}:${String(date.getMinutes()).padStart(2, '0')}`;
			}
		},
		onUnload() {
			// 清理定时器
			if (this.timer) {
				clearInterval(this.timer);
			}
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
	.sleep-container {
		min-height: 100vh;
		background-color: #f8f8f8;
		padding: 30rpx;
		padding-bottom: calc(30rpx + env(safe-area-inset-bottom));
		box-sizing: border-box;
		transition: background-color 0.3s ease;
	}
	
	.sleep-container.dark-theme {
		background-color: #1a1a1a;
	}
	
	.header-right {
		display: flex;
		align-items: center;
		gap: 20rpx;
	}
	
	.theme-switch {
		width: 60rpx;
		height: 60rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		background: rgba(255,255,255,0.1);
		border-radius: 30rpx;
		cursor: pointer;
		transition: background-color 0.3s ease;
	}
	
	.dark-theme .theme-switch {
		background: rgba(255,255,255,0.1);
	}
	
	.theme-icon {
		font-size: 32rpx;
	}
	
	.status-card {
		background: linear-gradient(135deg, #007AFF, #0056b3);
		border-radius: 24rpx;
		padding: 30rpx;
		color: #fff;
		margin-bottom: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0,122,255,0.2);
		transition: box-shadow 0.3s ease;
	}
	
	.dark-theme .status-card {
		box-shadow: 0 4rpx 20rpx rgba(0,122,255,0.3);
	}
	
	.status-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 30rpx;
	}
	
	.status-title {
		font-size: 32rpx;
		font-weight: 600;
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
		opacity: 0.8;
		display: block;
		margin-bottom: 10rpx;
	}
	
	.status-value {
		font-size: 36rpx;
		font-weight: 600;
	}
	
	.charts-container {
		display: flex;
		flex-direction: column;
		gap: 30rpx;
		margin-bottom: 30rpx;
	}
	
	.chart-item {
		background-color: #fff;
		border-radius: 24rpx;
		padding: 15rpx 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.08);
		transition: all 0.3s ease;
	}
	
	.dark-theme .chart-item {
		background-color: #2c2c2c;
		box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.2);
	}
	
	.chart-item:active {
		transform: scale(0.98);
	}
	
	.chart-title {
		display: block;
		font-size: 28rpx;
		color: #666;
		margin-bottom: 20rpx;
		font-weight: 500;
	}
	
	.dark-theme .chart-title {
		color: #999;
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
		gap: 30rpx;
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
</style> 