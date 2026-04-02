<template>
	<view class="notice-container">
		<view v-if="isLoading" class="loading-state">
			<text>正在加载新鲜事...</text>
		</view>

		<view v-else-if="noticeList.length === 0" class="empty-state">
			<text class="empty-emoji">📭</text>
			<text class="empty-text">暂无互动消息，快去发条动态吧~</text>
		</view>

		<view class="notice-list" v-else>
			<view class="notice-card" v-for="(item, index) in noticeList" :key="item.notificationId" @click="goToDetail(item.postId)">
				<image class="avatar" :src="item.senderAvatar || '/static/default-avatar.png'" mode="aspectFill"></image>
				
				<view class="content-wrapper">
					<view class="header">
						<text class="nickname">{{ item.senderNickname || '热心网友' }}</text>
						<text class="time">{{ formatTime(item.createTime) }}</text>
					</view>
					
					<view class="action-wrap">
						<text class="action-text" :class="getActionColor(item.type)">
							{{ formatAction(item.type) }}
						</text>
					</view>
					
					<view class="comment-box" v-if="item.type === 'COMMENT' && item.content">
						<text class="comment-text">{{ item.content }}</text>
					</view>
					
					<view class="post-preview-card">
						<view class="preview-line"></view>
						<text class="preview-text" v-if="item.postContentTeaser">
							{{ item.postContentTeaser }}
						</text>
						<text class="preview-text deleted" v-else>
							该动态已被删除
						</text>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			noticeList: [],
			isLoading: true
		};
	},
	onLoad() {
		this.fetchNotices();
	},
	methods: {
		fetchNotices() {
			this.isLoading = true;
			uni.request({
				url: 'http://localhost:8080/interaction-notices/list?pageNum=1&pageSize=50',
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					this.isLoading = false;
					if (res.data.code === 200) {
						this.noticeList = res.data.data.records || res.data.data;
						this.clearUnreadCount(); // 拉取列表成功后，顺便告诉后端全部已读
					}
				},
				fail: () => {
					this.isLoading = false;
					uni.showToast({ title: '网络开小差了', icon: 'none' });
				}
			});
		},
		clearUnreadCount() {
			uni.request({
				url: 'http://localhost:8080/interaction-notices/read-all',
				method: 'PUT',
				header: { 'token': uni.getStorageSync('token') }
			});
		},
		formatAction(type) {
			const map = { 
				'LIKE': '赞了你的动态', 
				'FAVORITE': '收藏了你的动态', 
				'COMMENT': '评论了你' 
			};
			return map[type] || '与你进行了互动';
		},
		getActionColor(type) {
			// 为不同的互动类型匹配不同的 CSS 类名，改变颜色
			const map = { 
				'LIKE': 'color-like', 
				'FAVORITE': 'color-fav', 
				'COMMENT': 'color-comment' 
			};
			return map[type] || '';
		},
		formatTime(timeStr) {
			if (!timeStr) return '';
			// 简单的截取，把 2026-04-01T12:30:00 变成 04-01 12:30
			return timeStr.replace('T', ' ').substring(5, 16);
		},
		goToDetail(postId) {
			if (!postId) return uni.showToast({ title: '原动态已丢失', icon: 'none' });
			uni.navigateTo({ url: `/pages/community/post-detail?id=${postId}` });
		}
	}
}
</script>

<style scoped>
.notice-container {
	min-height: 100vh;
	background-color: #F8F9FA;
	padding: 20rpx;
}

.loading-state, .empty-state {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	padding-top: 200rpx;
}

.empty-emoji {
	font-size: 100rpx;
	margin-bottom: 30rpx;
}

.empty-text {
	font-size: 28rpx;
	color: #999;
}

.notice-list {
	padding-bottom: 60rpx;
}

.notice-card {
	display: flex;
	background-color: #FFFFFF;
	border-radius: 20rpx;
	padding: 30rpx;
	margin-bottom: 20rpx;
	box-shadow: 0 4rpx 16rpx rgba(0, 0, 0, 0.02);
	transition: all 0.2s;
}

.notice-card:active {
	background-color: #F5F5F5;
}

.avatar {
	width: 88rpx;
	height: 88rpx;
	border-radius: 50%;
	margin-right: 24rpx;
	background-color: #EEE;
	flex-shrink: 0;
	border: 2rpx solid #F0F0F0;
}

.content-wrapper {
	flex: 1;
	overflow: hidden;
}

.header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 8rpx;
}

.nickname {
	font-size: 30rpx;
	font-weight: bold;
	color: #333;
}

.time {
	font-size: 22rpx;
	color: #BBB;
}

.action-wrap {
	margin-bottom: 16rpx;
}

.action-text {
	font-size: 28rpx;
	font-weight: 500;
}

/* 互动类型颜色定制 */
.color-like { color: #FF4D4F; }      /* 点赞红 */
.color-fav { color: #FAAD14; }       /* 收藏金 */
.color-comment { color: #1890FF; }   /* 评论蓝 */

.comment-box {
	margin-bottom: 20rpx;
}

.comment-text {
	font-size: 30rpx;
	color: #333;
	line-height: 1.5;
}

/* 原贴引用气泡 */
.post-preview-card {
	background-color: #F5F7FA;
	border-radius: 12rpx;
	padding: 20rpx 24rpx;
	display: flex;
	align-items: flex-start;
}

.preview-line {
	width: 6rpx;
	height: 30rpx;
	background-color: #DCDFE6;
	border-radius: 6rpx;
	margin-right: 16rpx;
	margin-top: 6rpx;
}

.preview-text {
	font-size: 26rpx;
	color: #666;
	line-height: 1.5;
	flex: 1;
	display: -webkit-box;
	-webkit-box-orient: vertical;
	-webkit-line-clamp: 2; /* 最多显示两行 */
	overflow: hidden;
}

.deleted {
	color: #999;
	font-style: italic;
}
</style>