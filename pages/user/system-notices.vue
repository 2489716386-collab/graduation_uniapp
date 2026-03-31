<template>
	<view class="container">
		<view class="notice-list" v-if="noticeList.length > 0">
			<view class="notice-card" v-for="item in noticeList" :key="item.noticeId">
				<view class="card-header">
					<view class="title-wrapper">
						<text class="tag" :class="item.type === 1 ? 'system' : 'activity'">
							{{ item.type === 1 ? '系统' : '活动' }}
						</text>
						<text class="title">{{ item.title }}</text>
					</view>
					<text class="time">{{ formatDate(item.createdAt) }}</text>
				</view>
				<view class="card-body">
					<text class="content">{{ item.content }}</text>
				</view>
			</view>
		</view>

		<view class="empty-state" v-else>
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
			this.loadNotices();
		},
		methods: {
			loadNotices() {
				uni.request({
					url: 'http://localhost:8080/notifications/user/list',
					method: 'GET',
					header: { 'token': uni.getStorageSync('token') },
					success: (res) => {
						if (res.data.code === 200) {
							this.noticeList = res.data.data;
							
							// 【消灭小红点的核心魔法】
							// 当用户点进这个页面时，把列表中最新的那个通知 ID 存起来
							if (this.noticeList.length > 0) {
								uni.setStorageSync('lastReadNoticeId', this.noticeList[0].noticeId);
							}
						}
					}
				});
			},
			formatDate(dateString) {
				if (!dateString) return '';
				// 简单的截取，把 2026-03-30T10:00:00 截成 03-30 10:00
				return dateString.substring(5, 16).replace('T', ' '); 
			}
		}
	}
</script>

<style scoped lang="scss">
	.container { background-color: #F7F9FC; min-height: 100vh; padding: 30rpx; }
	.notice-card { background: #FFFFFF; border-radius: 20rpx; padding: 30rpx; margin-bottom: 30rpx; box-shadow: 0 4rpx 16rpx rgba(0,0,0,0.03); }
	.card-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20rpx; }
	.title-wrapper { display: flex; align-items: center; }
	.tag { font-size: 20rpx; padding: 4rpx 12rpx; border-radius: 8rpx; margin-right: 16rpx; font-weight: bold; }
	.tag.system { background-color: #FFEFEF; color: #E85757; }
	.tag.activity { background-color: #EAF4FF; color: #4FA2FE; }
	.title { font-size: 32rpx; font-weight: bold; color: #1A1C20; }
	.time { font-size: 24rpx; color: #9AA0AF; }
	.card-body .content { font-size: 28rpx; color: #787E8C; line-height: 1.6; display: block; }
	.empty-state { text-align: center; color: #9AA0AF; margin-top: 200rpx; font-size: 28rpx; }
</style>