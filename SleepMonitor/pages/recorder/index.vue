<template>
	<view class="recorder-page" :class="{ 'dark-theme': isDarkTheme }">
		<view class="page-header">
			<text class="page-title">录音</text>
			<view class="theme-switch" @tap="toggleTheme">
				<text class="theme-icon">{{ isDarkTheme ? '🌞' : '🌙' }}</text>
			</view>
		</view>
		
		<view class="page-content">
			<!-- 录音组件 -->
			<view class="recorder-section">
				<audio-recorder 
					@recording-complete="handleRecordingComplete" 
					@recording-error="handleRecordingError"
					:is-dark-theme="isDarkTheme"
				/>
			</view>
			
			<!-- 录音历史 -->
			<view class="history-section">
				<view class="section-header">
					<text class="section-title">录音历史</text>
					<text class="section-subtitle">最近{{ maxDisplayRecords }}次录音</text>
				</view>
				
				<view class="recording-list" v-if="recordings.length > 0">
					<view 
						class="recording-item"
						v-for="(recording, index) in recordings.slice(0, maxDisplayRecords)"
						:key="recording.timestamp"
					>
						<view class="recording-info">
							<view class="recording-header">
								<text class="recording-date">{{ formatRecordingInfo(recording).date }}</text>
							</view>
							<view class="recording-details">
								<view class="detail-item">
									<text class="detail-label">录音时长</text>
									<text class="detail-value">{{ formatRecordingInfo(recording).duration }}</text>
								</view>
								<view class="detail-item">
									<text class="detail-label">文件大小</text>
									<text class="detail-value">{{ formatRecordingInfo(recording).size }}</text>
								</view>
								<view class="detail-item">
									<text class="detail-label">鼾声统计</text>
									<text class="detail-value">{{ formatRecordingInfo(recording).snoreInfo }}</text>
								</view>
							</view>
						</view>
						<view class="recording-actions">
							<button 
								class="action-btn play-btn"
								@tap="playRecording(recording)"
							>播放</button>
							<button 
								class="action-btn delete-btn"
								@tap="deleteRecording(index)"
							>删除</button>
						</view>
					</view>
				</view>
				<view class="empty-state" v-else>
					<text class="empty-text">暂无录音记录</text>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	import AudioRecorder from '@/components/audio-recorder/audio-recorder.vue';
	
	export default {
		components: {
			AudioRecorder
		},
		data() {
			return {
				isDarkTheme: false,
				recordings: [
					{
						timestamp: Date.now() - 3600000 * 2, // 2小时前
						duration: 7200, // 2小时
						size: 1024 * 1024 * 2, // 2MB
						snoreData: {
							count: 45,
							details: [
								{ time: 1200, intensity: 65, duration: 3 },
								{ time: 1800, intensity: 70, duration: 2 },
								{ time: 2400, intensity: 68, duration: 4 }
							],
							averageIntensity: 67.5
						}
					},
					{
						timestamp: Date.now() - 3600000 * 24, // 1天前
						duration: 3600, // 1小时
						size: 1024 * 1024, // 1MB
						snoreData: {
							count: 25,
							details: [
								{ time: 600, intensity: 62, duration: 2 },
								{ time: 1200, intensity: 68, duration: 3 },
								{ time: 1800, intensity: 65, duration: 2 }
							],
							averageIntensity: 65.0
						}
					},
					{
						timestamp: Date.now() - 3600000 * 48, // 2天前
						duration: 5400, // 1.5小时
						size: 1024 * 1024 * 1.5, // 1.5MB
						snoreData: {
							count: 35,
							details: [
								{ time: 900, intensity: 70, duration: 3 },
								{ time: 1800, intensity: 65, duration: 2 },
								{ time: 2700, intensity: 72, duration: 4 }
							],
							averageIntensity: 69.0
						}
					}
				],
				audioContext: null,
				maxDisplayRecords: 5,
				mockData: {
					snoreFrequency: {
						min: 2,
						max: 8
					},
					snoreIntensity: {
						min: 45,
						max: 75
					},
					snoreDuration: {
						min: 1,
						max: 5
					}
				}
			}
		},
		onLoad() {
			// 初始化音频上下文
			this.audioContext = uni.createInnerAudioContext();
			// 加载历史录音
			this.loadRecordings();
		},
		methods: {
			// 加载录音记录
			loadRecordings() {
				const recordings = uni.getStorageSync('recordings') || [];
				this.recordings = recordings;
			},
			
			// 保存录音记录
			saveRecordings() {
				uni.setStorageSync('recordings', this.recordings);
			},
			
			// 处理录音完成事件
			handleRecordingComplete(recording) {
				// 生成模拟的鼾声数据
				const snoreData = this.generateMockSnoreData(recording.duration);
				
				// 添加录音记录
				this.recordings.unshift({
					...recording,
					timestamp: Date.now(),
					snoreData
				});
				
				// 保存到本地存储
				this.saveRecordings();
			},
			
			// 处理录音错误
			handleRecordingError(error) {
				console.error('录音错误：', error);
				uni.showToast({
					title: '录音失败：' + error.message,
					icon: 'error'
				});
			},
			
			// 生成模拟的鼾声数据
			generateMockSnoreData(duration) {
				const minutes = duration / 60;
				const snoreCount = Math.floor(minutes * (Math.random() * (this.mockData.snoreFrequency.max - this.mockData.snoreFrequency.min) + this.mockData.snoreFrequency.min));
				
				const snoreEvents = [];
				let totalIntensity = 0;
				
				for (let i = 0; i < snoreCount; i++) {
					const intensity = Math.random() * (this.mockData.snoreIntensity.max - this.mockData.snoreIntensity.min) + this.mockData.snoreIntensity.min;
					const snoreDuration = Math.random() * (this.mockData.snoreDuration.max - this.mockData.snoreDuration.min) + this.mockData.snoreDuration.min;
					const time = Math.random() * duration;
					
					snoreEvents.push({
						time,
						intensity,
						duration: snoreDuration
					});
					
					totalIntensity += intensity;
				}
				
				return {
					count: snoreCount,
					details: snoreEvents,
					averageIntensity: snoreCount > 0 ? totalIntensity / snoreCount : 0
				};
			},
			
			// 格式化时间
			formatTime(seconds) {
				const hours = Math.floor(seconds / 3600);
				const minutes = Math.floor((seconds % 3600) / 60);
				const remainingSeconds = Math.floor(seconds % 60);
				
				if (hours > 0) {
					return `${hours}小时${minutes}分${remainingSeconds}秒`;
				} else if (minutes > 0) {
					return `${minutes}分${remainingSeconds}秒`;
				} else {
					return `${remainingSeconds}秒`;
				}
			},
			
			// 格式化日期
			formatDate(timestamp) {
				const date = new Date(timestamp);
				const month = date.getMonth() + 1;
				const day = date.getDate();
				const hours = date.getHours().toString().padStart(2, '0');
				const minutes = date.getMinutes().toString().padStart(2, '0');
				return `${month}月${day}日 ${hours}:${minutes}`;
			},
			
			// 格式化录音信息
			formatRecordingInfo(recording) {
				const date = this.formatDate(recording.timestamp);
				const duration = this.formatTime(recording.duration);
				const size = (recording.size / 1024).toFixed(1);
				const snoreCount = recording.snoreData?.count || 0;
				const avgIntensity = recording.snoreData?.averageIntensity.toFixed(1) || 0;
				
				return {
					date,
					duration,
					size: `${size} KB`,
					snoreInfo: `${snoreCount}次 (平均${avgIntensity}分贝)`
				};
			},
			
			// 播放录音
			playRecording(recording) {
				if (this.audioContext) {
					this.audioContext.stop();
					this.audioContext.src = recording.tempFilePath;
					this.audioContext.play();
				}
			},
			
			// 删除录音
			deleteRecording(index) {
				uni.showModal({
					title: '确认删除',
					content: '确定要删除这条录音记录吗？',
					success: (res) => {
						if (res.confirm) {
							const recording = this.recordings[index];
							// 删除临时文件
							uni.removeSavedFile({
								filePath: recording.tempFilePath,
								complete: () => {
									// 从列表中移除
									this.recordings.splice(index, 1);
									// 更新存储
									this.saveRecordings();
								}
							});
						}
					}
				});
			},
			
			// 切换主题
			toggleTheme() {
				this.isDarkTheme = !this.isDarkTheme;
				uni.setStorageSync('theme', this.isDarkTheme ? 'dark' : 'light');
			}
		},
		onUnload() {
			// 停止播放
			if (this.audioContext) {
				this.audioContext.stop();
				this.audioContext.destroy();
			}
		}
	}
</script>

<style>
	.recorder-page {
		min-height: 100vh;
		background-color: #f5f5f5;
		padding: 20rpx;
		box-sizing: border-box;
	}
	
	.dark-theme {
		background-color: #1a1a1a;
	}
	
	.page-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 20rpx 0;
		margin-bottom: 30rpx;
	}
	
	.page-title {
		font-size: 36rpx;
		font-weight: bold;
		color: #333;
	}
	
	.dark-theme .page-title {
		color: #fff;
	}
	
	.theme-switch {
		width: 80rpx;
		height: 80rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		background-color: #fff;
		border-radius: 50%;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
	}
	
	.dark-theme .theme-switch {
		background-color: #2c2c2c;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.2);
	}
	
	.theme-icon {
		font-size: 40rpx;
	}
	
	.page-content {
		display: flex;
		flex-direction: column;
		gap: 30rpx;
	}
	
	.recorder-section {
		background-color: #fff;
		border-radius: 20rpx;
		padding: 30rpx;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
	}
	
	.dark-theme .recorder-section {
		background-color: #2c2c2c;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
	}
	
	.history-section {
		background-color: #fff;
		border-radius: 20rpx;
		padding: 30rpx;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
	}
	
	.dark-theme .history-section {
		background-color: #2c2c2c;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.1);
	}
	
	.section-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 30rpx;
	}
	
	.section-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #333;
	}
	
	.dark-theme .section-title {
		color: #fff;
	}
	
	.section-subtitle {
		font-size: 24rpx;
		color: #999;
	}
	
	.dark-theme .section-subtitle {
		color: #666;
	}
	
	.recording-list {
		display: flex;
		flex-direction: column;
		gap: 20rpx;
	}
	
	.recording-item {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		padding: 24rpx;
		background-color: #f8f8f8;
		border-radius: 16rpx;
	}
	
	.dark-theme .recording-item {
		background-color: #333;
	}
	
	.recording-info {
		flex: 1;
		margin-right: 20rpx;
	}
	
	.recording-header {
		margin-bottom: 16rpx;
	}
	
	.recording-date {
		font-size: 28rpx;
		color: #666;
		font-weight: 500;
	}
	
	.dark-theme .recording-date {
		color: #999;
	}
	
	.recording-details {
		display: flex;
		flex-direction: column;
		gap: 12rpx;
	}
	
	.detail-item {
		display: flex;
		align-items: center;
		gap: 16rpx;
	}
	
	.detail-label {
		font-size: 26rpx;
		color: #999;
		min-width: 120rpx;
	}
	
	.dark-theme .detail-label {
		color: #666;
	}
	
	.detail-value {
		font-size: 26rpx;
		color: #333;
		font-weight: 500;
	}
	
	.dark-theme .detail-value {
		color: #fff;
	}
	
	.recording-actions {
		display: flex;
		flex-direction: column;
		gap: 12rpx;
	}
	
	.action-btn {
		padding: 12rpx 24rpx;
		font-size: 26rpx;
		border-radius: 8rpx;
		border: none;
		background-color: #f0f0f0;
		color: #666;
		line-height: 1.4;
	}
	
	.dark-theme .action-btn {
		background-color: #444;
		color: #999;
	}
	
	.play-btn {
		background-color: #91cc75;
		color: #fff;
	}
	
	.dark-theme .play-btn {
		background-color: #7ab55c;
	}
	
	.delete-btn {
		background-color: #ee6666;
		color: #fff;
	}
	
	.dark-theme .delete-btn {
		background-color: #d45555;
	}
	
	.empty-state {
		padding: 60rpx 0;
		text-align: center;
	}
	
	.empty-text {
		font-size: 28rpx;
		color: #999;
	}
	
	.dark-theme .empty-text {
		color: #666;
	}
</style> 