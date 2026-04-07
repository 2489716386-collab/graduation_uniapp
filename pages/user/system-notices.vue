<template>
	<view class="container">
		<view class="notice-list" v-if="noticeList.length > 0">
			<view class="notice-item" v-for="item in noticeList" :key="item.noticeId">
				<view class="notice-header">
					<text class="notice-title">{{ item.title }}</text>
					<text class="notice-time">{{ formatTime(item.createTime) }}</text>
				</view>
				<view class="notice-content">
					{{ item.content }}
				</view>
			</view>
		</view>

		<view class="empty-state" v-else>
			<image src="/static/images/empty-notice.png" mode="aspectFit" class="empty-img"></image>
			<text>暂无系统通知</text>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				noticeList: []
			}
		},
		onShow() {
			this.fetchNotices();
		},
		methods: {
			// 1. 获取通知列表（包含全局和个人）
			fetchNotices() {
				const token = uni.getStorageSync('token');
				uni.request({
					url: 'http://localhost:8080/notifications/user/list', // 对应你后端的接口
					method: 'GET',
					header: { 'token': token },
					success: (res) => {
						if (res.data.code === 200) {
							this.noticeList = res.data.data;
							
							// 【核心逻辑】列表加载成功后，如果列表不为空，则通知后端更新“最后阅读ID”
							if (this.noticeList.length > 0) {
								this.markAllAsRead();
							}
						}
					}
				});
			},

			// 2. 调用你刚刚写的“标记已读”接口
			markAllAsRead() {
				const token = uni.getStorageSync('token');
				uni.request({
					url: 'http://localhost:8080/notifications/user/mark-read', // 对应你提供的后端接口
					method: 'POST',
					header: { 'token': token },
					success: (res) => {
						if (res.data.code === 200) {
							console.log('已成功更新高水位线，小红点已消除');
						}
					}
				});
			},

			// 简单的日期格式化
			formatTime(timeStr) {
				if (!timeStr) return '';
				const date = new Date(timeStr);
				return `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}-${date.getDate().toString().padStart(2, '0')}`;
			}
		}
	}
</script>

<style scoped lang="scss">
	.container {
		background-color: #F7F9FC;
		min-height: 100vh;
		padding: 30rpx;
	}

	.notice-item {
		background-color: #FFFFFF;
		border-radius: 20rpx;
		padding: 30rpx;
		margin-bottom: 30rpx;
		box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.03);
	}

	.notice-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 16rpx;
	}

	.notice-title {
		font-size: 32rpx;
		font-weight: bold;
		color: #1A1C20;
	}

	.notice-time {
		font-size: 24rpx;
		color: #9AA0AF;
	}

	.notice-content {
		font-size: 28rpx;
		color: #4E5561;
		line-height: 1.6;
		word-break: break-all;
	}

	.empty-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding-top: 200rpx;
		color: #9AA0AF;
		
		.empty-img {
			width: 240rpx;
			height: 240rpx;
			margin-bottom: 20rpx;
		}
	}
</style>