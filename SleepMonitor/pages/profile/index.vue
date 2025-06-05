<template>
	<view class="profile-container">
		<!-- 顶部用户信息卡片 -->
		<view class="user-card">
			<view class="user-info">
				<image class="avatar" src="/static/default-avatar.png" mode="aspectFill"></image>
				<view class="user-detail">
					<text class="username">未登录</text>
					<text class="user-desc">点击登录账号</text>
				</view>
			</view>
			<view class="user-stats">
				<view class="stat-item">
					<text class="stat-num">0</text>
					<text class="stat-label">睡眠记录</text>
				</view>
				<view class="stat-item">
					<text class="stat-num">0</text>
					<text class="stat-label">咨询次数</text>
				</view>
				<view class="stat-item">
					<text class="stat-num">0</text>
					<text class="stat-label">健康评分</text>
				</view>
			</view>
		</view>
		
		<!-- 功能列表 -->
		<view class="menu-list">
			<view class="menu-group">
				<view class="menu-item" @click="navigateTo('/pages/profile/records')">
					<text class="menu-icon">📊</text>
					<text class="menu-text">睡眠记录</text>
					<text class="menu-arrow">></text>
				</view>
				<view class="menu-item" @click="navigateTo('/pages/profile/analysis')">
					<text class="menu-icon">📈</text>
					<text class="menu-text">睡眠分析</text>
					<text class="menu-arrow">></text>
				</view>
				<view class="menu-item" @click="navigateTo('/pages/profile/reports')">
					<text class="menu-icon">📝</text>
					<text class="menu-text">健康报告</text>
					<text class="menu-arrow">></text>
				</view>
			</view>
			
			<view class="menu-group">
				<view class="menu-item" @click="navigateTo('/pages/profile/settings')">
					<text class="menu-icon">⚙️</text>
					<text class="menu-text">设置</text>
					<text class="menu-arrow">></text>
				</view>
				<view class="menu-item" @click="navigateTo('/pages/profile/about')">
					<text class="menu-icon">ℹ️</text>
					<text class="menu-text">关于我们</text>
					<text class="menu-arrow">></text>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				userInfo: null
			}
		},
		methods: {
			navigateTo(url) {
				uni.navigateTo({
					url: url
				});
			}
		}
	}
</script>

<style>
	.profile-container {
		min-height: 100vh;
		background-color: #f8f8f8;
		padding-bottom: env(safe-area-inset-bottom);
	}
	
	.user-card {
		background: linear-gradient(135deg, #007AFF, #0056b3);
		padding: 40rpx 30rpx;
		color: #fff;
		margin-bottom: 30rpx;
	}
	
	.user-info {
		display: flex;
		align-items: center;
		margin-bottom: 40rpx;
	}
	
	.avatar {
		width: 120rpx;
		height: 120rpx;
		border-radius: 60rpx;
		border: 4rpx solid rgba(255,255,255,0.3);
	}
	
	.user-detail {
		margin-left: 30rpx;
	}
	
	.username {
		font-size: 36rpx;
		font-weight: 600;
		margin-bottom: 10rpx;
		display: block;
	}
	
	.user-desc {
		font-size: 26rpx;
		opacity: 0.8;
	}
	
	.user-stats {
		display: flex;
		justify-content: space-around;
		padding: 20rpx 0;
		border-top: 2rpx solid rgba(255,255,255,0.1);
	}
	
	.stat-item {
		text-align: center;
	}
	
	.stat-num {
		font-size: 40rpx;
		font-weight: 600;
		display: block;
		margin-bottom: 8rpx;
	}
	
	.stat-label {
		font-size: 24rpx;
		opacity: 0.8;
	}
	
	.menu-list {
		padding: 0 30rpx;
	}
	
	.menu-group {
		background-color: #fff;
		border-radius: 20rpx;
		margin-bottom: 30rpx;
		overflow: hidden;
		box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
	}
	
	.menu-item {
		display: flex;
		align-items: center;
		padding: 30rpx;
		border-bottom: 2rpx solid #f5f5f5;
	}
	
	.menu-item:last-child {
		border-bottom: none;
	}
	
	.menu-icon {
		font-size: 40rpx;
		margin-right: 20rpx;
	}
	
	.menu-text {
		flex: 1;
		font-size: 30rpx;
		color: #333;
	}
	
	.menu-arrow {
		color: #999;
		font-size: 30rpx;
	}
	
	/* 深色模式适配 */
	@media (prefers-color-scheme: dark) {
		.profile-container {
			background-color: #1a1a1a;
		}
		
		.menu-group {
			background-color: #2c2c2c;
			box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.2);
		}
		
		.menu-item {
			border-bottom-color: #333;
		}
		
		.menu-text {
			color: #fff;
		}
		
		.menu-arrow {
			color: #666;
		}
	}
</style> 