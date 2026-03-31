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
							// 【修复 1】防御性赋值，防止后端返回 null 导致后续报错
							this.noticeList = res.data.data || [];
							
							if (this.noticeList.length > 0) {
								uni.setStorageSync('lastReadNoticeId', this.noticeList[0].noticeId);
							}
						}
					}
				});
			},
			
			// 【修复 2】终极时间格式化方法：兼容数组和字符串两种后端格式
			formatDate(dateData) {
				if (!dateData) return '';
				
				// 情况 A：Spring Boot 将时间返回成了数组 [2026, 3, 30, 10, 30]
				if (Array.isArray(dateData)) {
					// 提取月、日、时、分，并保证两位数显示 (比如 3 变成 03)
					const month = String(dateData[1] || 1).padStart(2, '0');
					const day = String(dateData[2] || 1).padStart(2, '0');
					const hour = String(dateData[3] || 0).padStart(2, '0');
					const minute = String(dateData[4] || 0).padStart(2, '0');
					return `${month}-${day} ${hour}:${minute}`;
				}
				
				// 情况 B：如果后端返回的是 ISO 字符串 "2026-03-30T10:30:00"
				if (typeof dateData === 'string') {
					if (dateData.includes('T')) {
						return dateData.substring(5, 16).replace('T', ' ');
					}
					return dateData;
				}
				
				return String(dateData);
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