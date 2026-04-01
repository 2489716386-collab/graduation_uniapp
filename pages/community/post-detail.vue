<template>
	<view class="detail-container">
		<view v-if="isLoading" class="loading-state">
			<text>加载中...</text>
		</view>

		<view v-else-if="!post" class="error-state">
			<text>糟糕，动态不见了~</text>
		</view>

		<view v-else class="post-content">
			<view class="author-header">
				<image class="avatar" src="/static/default-avatar.png" mode="aspectFill"></image>
				<view class="info">
					<text class="nickname">用户_{{ post.userId }}</text>
					<text class="time">{{ post.createTime }}</text>
				</view>
			</view>

			<view class="body-text">
				<text>{{ post.content }}</text>
			</view>

			<view class="tags-wrap" v-if="tagsArray.length > 0">
				<text class="tag" v-for="(tag, index) in tagsArray" :key="index">#{{ tag }}</text>
			</view>

			<view class="divider"></view>

			<view class="comment-section">
				<view class="comment-title">全部评论 ({{ post.commentsCount || 0 }})</view>
				<view class="empty-comment">
					<text>还没有人评论，快来抢沙发吧~</text>
				</view>
			</view>
		</view>

		<view class="bottom-bar" v-if="post">
			<view class="comment-input-btn">
				<text>说点什么...</text>
			</view>
			<view class="action-icons">
				<view class="icon-item" @click="toggleLike">
					<text :class="post.isLiked ? 'color-active' : 'color-normal'">{{ post.isLiked ? '已赞' : '点赞' }} {{ post.likesCount || 0 }}</text>
				</view>
				<view class="icon-item" @click="toggleFavorite">
					<text :class="post.isFavorited ? 'color-active' : 'color-normal'">{{ post.isFavorited ? '已收藏' : '收藏' }}</text>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			postId: null,
			post: null,
			isLoading: true,
			tagsArray: []
		};
	},
	// onLoad 接收 URL 传过来的参数
	onLoad(options) {
		if (options.id) {
			this.postId = options.id;
			this.fetchPostDetail();
		} else {
			this.isLoading = false;
		}
	},
	methods: {
		fetchPostDetail() {
			uni.request({
				// 调用你后端的根据 ID 查询接口
				url: `http://localhost:8080/community-posts/${this.postId}`,
				method: 'GET',
				header: {
					'token': uni.getStorageSync('token') // 如果未登录也可以看，后端记得放行这个接口
				},
				success: (res) => {
					this.isLoading = false;
					if (res.data.code === 200) {
						this.post = res.data.data;
						// 处理标签字符串
						if (this.post.tags) {
							this.tagsArray = this.post.tags.split(',');
						}
					} else {
						uni.showToast({ title: res.data.msg || '获取失败', icon: 'none' });
					}
				},
				fail: () => {
					this.isLoading = false;
					uni.showToast({ title: '网络异常', icon: 'none' });
				}
			});
		},
		// 详情页内的点赞
		toggleLike() {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });

			uni.request({
				url: `http://localhost:8080/likes/toggle/${this.postId}`,
				method: 'POST',
				header: { 'token': token },
				success: (res) => {
					if (res.data.code === 200) {
						// 乐观更新：根据后端返回的真实状态修改
						this.post.isLiked = res.data.data;
						this.post.likesCount += this.post.isLiked ? 1 : -1;
					}
				}
			});
		},
		// 详情页内的收藏
		toggleFavorite() {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });

			uni.request({
				url: `http://localhost:8080/favorites/toggle/${this.postId}`,
				method: 'POST',
				header: { 'token': token },
				success: (res) => {
					if (res.data.code === 200) {
						this.post.isFavorited = res.data.data;
						uni.showToast({ title: this.post.isFavorited ? '已收藏' : '已取消收藏', icon: 'none' });
					}
				}
			});
		}
	}
}
</script>

<style scoped>
.detail-container {
	min-height: 100vh;
	background-color: #FFFFFF;
	padding-bottom: 120rpx; /* 留出底部悬浮栏的高度 */
}

.loading-state, .error-state {
	text-align: center;
	padding-top: 200rpx;
	color: #999;
}

.post-content {
	padding: 30rpx;
}

/* 作者信息 */
.author-header {
	display: flex;
	align-items: center;
	margin-bottom: 30rpx;
}
.avatar {
	width: 80rpx;
	height: 80rpx;
	border-radius: 50%;
	margin-right: 20rpx;
	background-color: #f0f0f0;
}
.info {
	display: flex;
	flex-direction: column;
}
.nickname {
	font-size: 32rpx;
	font-weight: bold;
	color: #333;
}
.time {
	font-size: 24rpx;
	color: #999;
	margin-top: 6rpx;
}

/* 正文与标签 */
.body-text {
	font-size: 32rpx;
	color: #333;
	line-height: 1.8;
	margin-bottom: 20rpx;
}
.tags-wrap {
	display: flex;
	flex-wrap: wrap;
	gap: 16rpx;
	margin-bottom: 30rpx;
}
.tag {
	font-size: 24rpx;
	color: #42b983;
	background-color: rgba(66, 185, 131, 0.1);
	padding: 6rpx 24rpx;
	border-radius: 30rpx;
}

.divider {
	height: 16rpx;
	background-color: #F8F8F8;
	margin: 0 -30rpx 30rpx -30rpx;
}

/* 评论区占位 */
.comment-title {
	font-size: 30rpx;
	font-weight: bold;
	color: #333;
	margin-bottom: 30rpx;
}
.empty-comment {
	text-align: center;
	padding: 60rpx 0;
	color: #999;
	font-size: 28rpx;
}

/* 底部悬浮栏 */
.bottom-bar {
	position: fixed;
	bottom: 0;
	left: 0;
	width: 100%;
	height: 100rpx;
	background-color: #FFFFFF;
	border-top: 2rpx solid #EEEEEE;
	display: flex;
	align-items: center;
	padding: 0 30rpx;
	box-sizing: border-box;
	z-index: 99;
}
.comment-input-btn {
	flex: 1;
	background-color: #F5F5F5;
	height: 64rpx;
	border-radius: 32rpx;
	display: flex;
	align-items: center;
	padding: 0 30rpx;
	font-size: 26rpx;
	color: #999;
}
.action-icons {
	display: flex;
	align-items: center;
	margin-left: 30rpx;
	gap: 30rpx;
}
.icon-item {
	font-size: 26rpx;
}
.color-normal { color: #666; }
.color-active { color: #42b983; font-weight: bold; }
</style>