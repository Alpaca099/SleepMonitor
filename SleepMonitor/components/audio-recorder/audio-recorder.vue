<template>
	<view class="recorder-container">
		<view class="recorder-card" :class="{ 'dark-theme': isDarkTheme }">
			<view class="recorder-header">
				<text class="recorder-title">睡眠录音</text>
				<text class="recorder-subtitle">记录睡眠时的声音</text>
			</view>
			
			<view class="recorder-controls">
				<view class="recorder-status">
					<view class="status-indicator" :class="{ 'recording': isRecording }">
						<view class="status-dot"></view>
					</view>
					<text class="status-text">{{ isRecording ? '正在录音' : '未开始录音' }}</text>
				</view>
				
				<view class="recorder-button" 
					:class="{ 'recording': isRecording }" 
					@click="toggleRecording"
				>
					<view class="button-inner">
						<view class="button-icon" :class="{ 'recording': isRecording }">
							<view class="icon-circle"></view>
						</view>
					</view>
				</view>
				
				<view class="recorder-info" v-if="isRecording">
					<text class="time-display">{{ formatTime(recordingDuration) }}</text>
					<text class="size-display">{{ formatFileSize(currentSize) }}</text>
				</view>
			</view>
			
			<view class="recorder-tips">
				<text class="tip-text">• 最长录音时间：5分钟</text>
				<text class="tip-text">• 录音将自动保存</text>
				<text class="tip-text">• 请确保麦克风权限已开启</text>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		name: 'AudioRecorder',
		props: {
			isDarkTheme: {
				type: Boolean,
				default: false
			}
		},
		data() {
			return {
				isRecording: false,
				recordingDuration: 0,
				durationTimer: null,
				lastRecording: null,
				maxDuration: 5 * 60 * 1000, // 最大录音时长5分钟
				mediaRecorder: null, // H5环境下的MediaRecorder实例
				recorderManager: null, // 录音管理器实例
				currentSize: 0, // 当前录音文件大小
				options: {
					duration: 300000, // 最长录音时间，单位ms
					sampleRate: 44100, // 采样率
					numberOfChannels: 1, // 录音通道数
					encodeBitRate: 192000, // 编码码率
					format: 'mp3', // 音频格式
					frameSize: 50 // 指定帧大小
				}
			}
		},
		created() {
			// 初始化录音管理器
			this.initRecorderManager();
			
			if (!this.recorderManager) {
				console.warn('当前环境不支持录音功能');
				return;
			}
			
			// 监听录音结束事件
			this.recorderManager.onStop((res) => {
				console.log('录音结束:', res);
				this.isRecording = false;
				this.recordingDuration = 0;
				this.currentSize = 0;
				clearInterval(this.durationTimer);
				
				// 保存录音信息
				this.lastRecording = {
					tempFilePath: res.tempFilePath,
					duration: res.duration || this.recordingDuration,
					size: res.fileSize || this.currentSize,
					timestamp: new Date().getTime()
				};
				
				// 保存录音文件
				this.saveRecordingFile(res.tempFilePath);
				
				// 触发录音结束事件
				this.$emit('recording-complete', this.lastRecording);
			});
			
			// 监听录音错误事件
			this.recorderManager.onError((res) => {
				console.error('录音错误:', res);
				uni.showToast({
					title: '录音出错: ' + (res.errMsg || res.message || '未知错误'),
					icon: 'none'
				});
				this.isRecording = false;
				this.recordingDuration = 0;
				this.currentSize = 0;
				clearInterval(this.durationTimer);
			});

			// 监听录音帧数据
			this.recorderManager.onFrameRecorded((res) => {
				const { frameBuffer, isLastFrame } = res;
				if (frameBuffer && frameBuffer.length > 0) {
					// 计算文件大小（估算）
					this.currentSize = Math.floor(this.recordingDuration / 1000 * this.options.encodeBitRate / 8);
				}
			});
		},
		methods: {
			initRecorderManager() {
				// #ifdef APP-PLUS
				this.recorderManager = uni.getRecorderManager();
				// #endif
				
				// #ifdef H5
				// H5环境下使用Web Audio API
				if (window.navigator.mediaDevices && window.navigator.mediaDevices.getUserMedia) {
					let recorderInstance = null;
					this.recorderManager = {
						start: (options) => {
							return new Promise((resolve, reject) => {
								window.navigator.mediaDevices.getUserMedia({ audio: true })
									.then(stream => {
										recorderInstance = new MediaRecorder(stream);
										const audioChunks = [];
										
										recorderInstance.ondataavailable = (event) => {
											audioChunks.push(event.data);
										};
										
										recorderInstance.onstop = () => {
											const audioBlob = new Blob(audioChunks, { type: 'audio/mp3' });
											const audioUrl = URL.createObjectURL(audioBlob);
											
											// 触发录音结束事件
											if (this.recorderManager.stopCallback) {
												this.recorderManager.stopCallback({
													tempFilePath: audioUrl,
													duration: this.recordingDuration,
													fileSize: audioBlob.size
												});
											}
											
											// 停止所有音轨
											stream.getTracks().forEach(track => track.stop());
										};
										
										recorderInstance.start();
										this.mediaRecorder = recorderInstance;
										resolve();
									})
									.catch(error => {
										console.error('获取麦克风权限失败:', error);
										if (this.recorderManager.errorCallback) {
											this.recorderManager.errorCallback(error);
										}
										reject(error);
									});
							});
						},
						stop: () => {
							if (recorderInstance && recorderInstance.state !== 'inactive') {
								recorderInstance.stop();
							}
						},
						onStop: (callback) => {
							this.recorderManager.stopCallback = callback;
						},
						onError: (callback) => {
							this.recorderManager.errorCallback = callback;
						},
						stopCallback: null,
						errorCallback: null
					};
				}
				// #endif
			},
			
			async toggleRecording() {
				if (!this.recorderManager) {
					uni.showToast({
						title: '当前环境不支持录音功能',
						icon: 'none'
					});
					return;
				}
				
				if (this.isRecording) {
					// 停止录音
					this.recorderManager.stop();
				} else {
					// 检查权限并开始录音
					try {
						const auth = await this.checkPermission();
						if (auth) {
							this.startRecording();
						}
					} catch (error) {
						console.error('权限检查失败:', error);
						uni.showToast({
							title: '无法获取录音权限',
							icon: 'none'
						});
					}
				}
			},
			
			async checkPermission() {
				return new Promise((resolve, reject) => {
					// #ifdef APP-PLUS
					const permission = 'android.permission.RECORD_AUDIO';
					plus.android.requestPermissions(
						[permission],
						function(resultObj) {
							if (resultObj.granted.length === 1) {
								resolve(true);
							} else {
								uni.showModal({
									title: '提示',
									content: '需要录音权限才能使用此功能',
									confirmText: '去设置',
									success: (res) => {
										if (res.confirm) {
											// 打开应用设置页面
											plus.runtime.openURL('app-settings:');
										}
										resolve(false);
									}
								});
							}
						},
						function(error) {
							console.error('申请权限错误:', error);
							reject(error);
						}
					);
					// #endif
					
					// #ifdef H5
					if (window.navigator.mediaDevices && window.navigator.mediaDevices.getUserMedia) {
						window.navigator.mediaDevices.getUserMedia({ audio: true })
							.then(stream => {
								// 立即停止流，我们只需要测试权限
								stream.getTracks().forEach(track => track.stop());
								resolve(true);
							})
							.catch(error => {
								console.error('获取麦克风权限失败:', error);
								uni.showModal({
									title: '提示',
									content: '需要麦克风权限才能使用此功能',
									confirmText: '确定',
									showCancel: false
								});
								resolve(false);
							});
					} else {
						uni.showToast({
							title: '当前浏览器不支持录音功能',
							icon: 'none'
						});
						resolve(false);
					}
					// #endif
				});
			},
			
			startRecording() {
				// #ifdef APP-PLUS
				this.recorderManager.start(this.options);
				// #endif
				
				// #ifdef H5
				this.recorderManager.start(this.options).catch(error => {
					console.error('开始录音失败:', error);
					uni.showToast({
						title: '开始录音失败',
						icon: 'none'
					});
					return;
				});
				// #endif
				
				this.isRecording = true;
				this.startTime = Date.now();
				this.recordingDuration = 0;
				this.currentSize = 0;
				
				console.log('开始录音，时间戳:', this.startTime);
				
				// 开始计时
				this.durationTimer = setInterval(() => {
					const now = Date.now();
					const duration = now - this.startTime;
					console.log('当前时间:', now, '开始时间:', this.startTime, '时长:', duration);
					
					this.recordingDuration = duration;
					
					// 估算文件大小
					this.currentSize = Math.floor(duration / 1000 * this.options.encodeBitRate / 8);
					
					// 检查是否达到最大录音时长
					if (duration >= this.maxDuration) {
						this.recorderManager.stop();
					}
				}, 100);
			},
			
			async saveRecordingFile(tempFilePath) {
				try {
					// 生成文件名
					const fileName = `sleep_recording_${new Date().getTime()}.mp3`;
					
					// #ifdef APP-PLUS
					// 在APP环境下保存到应用私有目录
					const savePath = plus.io.convertLocalFileSystemURL(`_doc/${fileName}`);
					await new Promise((resolve, reject) => {
						plus.io.resolveLocalFileSystemURL(tempFilePath, (entry) => {
							entry.copyTo(
								plus.io.convertLocalFileSystemURL('_doc'),
								fileName,
								() => {
									console.log('录音文件保存成功:', savePath);
									this.lastRecording.savedPath = savePath;
									resolve();
								},
								(error) => {
									console.error('保存录音文件失败:', error);
									reject(error);
								}
							);
						});
					});
					// #endif
					
					// #ifdef H5
					// H5环境下直接使用Blob URL
					this.lastRecording.savedPath = tempFilePath;
					// 可以在这里添加下载功能
					const link = document.createElement('a');
					link.href = tempFilePath;
					link.download = fileName;
					link.click();
					// #endif
					
					uni.showToast({
						title: '录音已保存',
						icon: 'success'
					});
				} catch (error) {
					console.error('保存录音文件失败:', error);
					uni.showToast({
						title: '保存录音失败',
						icon: 'none'
					});
				}
			},
			
			formatTime(ms) {
				//console.log('格式化时间，输入值:', ms, '类型:', typeof ms);
				
				// 确保ms是有效的数字
				if (typeof ms !== 'number' || isNaN(ms)) {
					//console.log('无效的时间值，返回00:00');
					return '00:00';
				}
				
				// 确保ms是整数
				ms = Math.floor(ms);
				
				const totalSeconds = Math.floor(ms / 1000);
				const minutes = Math.floor(totalSeconds / 60);
				const seconds = totalSeconds % 60;
				
				//console.log('计算得到 - 总秒数:', totalSeconds, '分钟:', minutes, '秒:', seconds);
				
				// 确保分钟和秒数都是有效的数字
				if (isNaN(minutes) || isNaN(seconds)) {
					//console.log('计算后的分钟或秒数无效，返回00:00');
					return '00:00';
				}
				
				const result = `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
				console.log('格式化结果:', result);
				return result;
			},

			
			formatFileSize(bytes) {
				if (bytes < 1024) {
					return bytes + 'B';
				} else if (bytes < 1024 * 1024) {
					return (bytes / 1024).toFixed(2) + 'KB';
				} else {
					return (bytes / (1024 * 1024)).toFixed(2) + 'MB';
				}
			},
			
			formatDate(timestamp) {
				const date = new Date(timestamp);
				return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')} ${String(date.getHours()).padStart(2, '0')}:${String(date.getMinutes()).padStart(2, '0')}`;
			}
		},
		beforeDestroy() {
			// 组件销毁前清理定时器
			if (this.durationTimer) {
				clearInterval(this.durationTimer);
			}
		}
	}
</script>

<style>
	.recorder-container {
		padding: 0;
	}
	
	.recorder-card {
		background: #ffffff;
		border-radius: 24rpx;
		padding: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.08);
		transition: all 0.3s ease;
	}
	
	.recorder-card.dark-theme {
		background: #2c2c2c;
		box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.2);
	}
	
	.recorder-header {
		margin-bottom: 30rpx;
	}
	
	.recorder-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #333;
		display: block;
		margin-bottom: 8rpx;
		transition: color 0.3s ease;
	}
	
	.dark-theme .recorder-title {
		color: #fff;
	}
	
	.recorder-subtitle {
		font-size: 24rpx;
		color: #666;
		transition: color 0.3s ease;
	}
	
	.dark-theme .recorder-subtitle {
		color: #999;
	}
	
	.recorder-controls {
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin-bottom: 30rpx;
		padding: 20rpx 0;
		border-top: 2rpx solid #f5f5f5;
		border-bottom: 2rpx solid #f5f5f5;
		transition: border-color 0.3s ease;
	}
	
	.dark-theme .recorder-controls {
		border-color: #333;
	}
	
	.recorder-status {
		display: flex;
		align-items: center;
		flex: 1;
	}
	
	.status-indicator {
		width: 24rpx;
		height: 24rpx;
		border-radius: 12rpx;
		background: #f5f5f5;
		margin-right: 16rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		transition: all 0.3s ease;
	}
	
	.status-indicator.recording {
		background: #ff4d4f;
		animation: pulse 1.5s infinite;
	}
	
	.status-dot {
		width: 12rpx;
		height: 12rpx;
		border-radius: 6rpx;
		background: #fff;
		opacity: 0;
		transition: opacity 0.3s ease;
	}
	
	.status-indicator.recording .status-dot {
		opacity: 1;
	}
	
	.status-text {
		font-size: 28rpx;
		color: #666;
		transition: color 0.3s ease;
	}
	
	.dark-theme .status-text {
		color: #999;
	}
	
	.recorder-button {
		width: 120rpx;
		height: 120rpx;
		border-radius: 60rpx;
		background: #f5f5f5;
		display: flex;
		align-items: center;
		justify-content: center;
		transition: all 0.3s ease;
		margin: 0 40rpx;
	}
	
	.dark-theme .recorder-button {
		background: #333;
	}
	
	.button-inner {
		width: 100rpx;
		height: 100rpx;
		border-radius: 50rpx;
		background: #fff;
		display: flex;
		align-items: center;
		justify-content: center;
		transition: all 0.3s ease;
	}
	
	.dark-theme .button-inner {
		background: #2c2c2c;
	}
	
	.button-icon {
		width: 40rpx;
		height: 40rpx;
		border-radius: 20rpx;
		background: #ff4d4f;
		transition: all 0.3s ease;
		display: flex;
		align-items: center;
		justify-content: center;
	}
	
	.button-icon.recording {
		width: 32rpx;
		height: 32rpx;
		border-radius: 6rpx;
	}
	
	.icon-circle {
		width: 24rpx;
		height: 24rpx;
		border-radius: 12rpx;
		background: #fff;
		transition: all 0.3s ease;
	}
	
	.button-icon.recording .icon-circle {
		width: 16rpx;
		height: 16rpx;
		border-radius: 2rpx;
	}
	
	.recorder-info {
		flex: 1;
		text-align: right;
	}
	
	.time-display {
		font-size: 32rpx;
		font-weight: 600;
		color: #333;
		display: block;
		margin-bottom: 8rpx;
		transition: color 0.3s ease;
	}
	
	.dark-theme .time-display {
		color: #fff;
	}
	
	.size-display {
		font-size: 24rpx;
		color: #666;
		transition: color 0.3s ease;
	}
	
	.dark-theme .size-display {
		color: #999;
	}
	
	.recorder-tips {
		margin-top: 20rpx;
	}
	
	.tip-text {
		font-size: 24rpx;
		color: #999;
		display: block;
		margin-bottom: 8rpx;
		transition: color 0.3s ease;
	}
	
	.dark-theme .tip-text {
		color: #666;
	}
	
	@keyframes pulse {
		0% {
			transform: scale(1);
			opacity: 1;
		}
		50% {
			transform: scale(1.2);
			opacity: 0.8;
		}
		100% {
			transform: scale(1);
			opacity: 1;
		}
	}
</style> 