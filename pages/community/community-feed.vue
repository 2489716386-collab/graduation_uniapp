<template>
	<view class="feed-wrapper">
		<view class="search-box">
			<view class="search-inner" @click="goToSearch">
				<text class="search-placeholder">搜索宠物日常、经验、问答...</text>
				<view class="search-confirm-btn">
					<text class="search-icon">🔍</text>
					<text class="search-text">搜索</text>
				</view>
			</view>
		</view>

		<view class="post-list">
			<view class="post-card" v-for="(post, index) in postList" :key="index"
				@click="goToDetail(post.id || post.postId)">
				<view class="post-header">
					<image class="avatar" :src="post.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
					<view class="user-info">
						<text class="nickname">{{ post.nickname }}</text>
						<text class="time">{{ post.createTime }}</text>
					</view>
					<view class="recommend-badge" v-if="post.recommendScore && post.recommendScore > 0">
						<text>🔥 匹配度 {{ post.recommendScore }}</text>
					</view>
				</view>

				<view class="post-body">
					<text class="content-text">{{ post.content }}</text>
					<view class="tags-wrap" v-if="post.tags && post.tags.length">
						<text class="tag" v-for="(tag, tidx) in post.tags" :key="tidx">#{{ tag }}</text>
					</view>
				</view>

				<view class="post-footer">
					<view class="action-btn" @click.stop="toggleLike(post)">
						<text :class="post.isLiked ? 'color-active' : 'color-normal'">{{ post.isLiked ? '已赞' : '点赞' }}
							{{ post.likeCount || post.likes || 0 }}</text>
					</view>
					<view class="action-btn">
						<text class="color-normal">评论 {{ post.commentCount || post.comments || 0 }}</text>
					</view>
					<view class="action-btn" @click.stop="toggleFavorite(post)">
						<text
							:class="post.isFavorited ? 'color-active' : 'color-normal'">{{ post.isFavorited ? '已收藏' : '收藏' }}</text>
					</view>
				</view>
			</view>
			<view class="loading-tip">没有更多动态啦~</view>
		</view>

		<view class="add-post-btn" @click="goToAddPost">
			<text class="add-icon">+</text>
		</view>
	</view>
</template>

<script>
	export default {
		name: "CommunityFeed",
		data() {
			return {
				postList: []
			};
		},
		methods: {
			loadMore() {
				console.log("CommunityFeed 触底加载...");
			},
			refresh() {
				console.log("CommunityFeed 刷新数据...");
			},

			// 获取个性化推荐列表
			fetchRecommendedPosts() {
				const token = uni.getStorageSync('token');
				uni.request({
					// 假设你在 CommunityPostsController 里写的推荐接口是这个路径
					url: 'http://localhost:8080/community-posts/recommend',
					method: 'GET',
					header: {
						'token': token
					}, // 必须传 token，后端才能解析 userId 获取宠物和搜索历史
					success: (res) => {
						if (res.data.code === 200) {
							this.postList = res.data.data;
						}
					}
				});
			},

			// 下拉刷新时调用
			refresh() {
				this.fetchRecommendedPosts();
				uni.stopPullDownRefresh();
			},

			// 跳转到搜索结果界面（通常先进入搜索页进行输入）
			goToSearch() {
				uni.navigateTo({
					url: '/pages/community/dynamic/search'
				});
			},

			goToAddPost() {
				const token = uni.getStorageSync('token');
				if (!token) return uni.showToast({
					title: '请先登录',
					icon: 'none'
				});
				uni.navigateTo({
					url: '/pages/community/dynamic/add-post'
				});
			},

			goToDetail(postId) {
				if (!postId) return;
				uni.navigateTo({
					url: `/pages/community/post-detail?id=${postId}`
				});
			},

			toggleLike(post) {
				const token = uni.getStorageSync('token');
				if (!token) return uni.showToast({
					title: '请先登录',
					icon: 'none'
				});
				post.isLiked = !post.isLiked;
				if (post.likeCount !== undefined) post.likeCount += post.isLiked ? 1 : -1;
				const targetId = post.postId || post.id;
				uni.request({
					url: `http://localhost:8080/likes/toggle/${targetId}`,
					method: 'POST',
					header: {
						'token': token
					},
					success: (res) => {
						if (res.data.code !== 200) this.rollbackLike(post);
					},
					fail: () => this.rollbackLike(post)
				});
			},

			rollbackLike(post) {
				post.isLiked = !post.isLiked;
				if (post.likeCount !== undefined) post.likeCount += post.isLiked ? 1 : -1;
			},

			toggleFavorite(post) {
				const token = uni.getStorageSync('token');
				if (!token) return uni.showToast({
					title: '请先登录',
					icon: 'none'
				});
				post.isFavorited = !post.isFavorited;
				uni.request({
					url: `http://localhost:8080/favorites/toggle/${post.postId || post.id}`,
					method: 'POST',
					header: {
						'token': token
					},
					success: (res) => {
						if (res.data.code !== 200) post.isFavorited = !post.isFavorited;
					},
					fail: () => post.isFavorited = !post.isFavorited
				});
			}
		}
	}
</script>

<style scoped>
	.recommend-badge {
	  margin-left: auto; /* 推到最右边 */
	  background: linear-gradient(135deg, #ff9a9e 0%, #fecfef 99%, #fecfef 100%);
	  padding: 4rpx 16rpx;
	  border-radius: 20rpx;
	}
	.recommend-badge text {
	  font-size: 22rpx;
	  color: #d32f2f;
	  font-weight: bold;
	}
	.feed-wrapper {
		width: 100%;
		position: relative;
	}

	/* 搜索栏样式改造 */
	.search-box {
		padding: 20rpx 30rpx;
		background-color: #FFFFFF;
	}

	.search-inner {
		display: flex;
		align-items: center;
		justify-content: space-between;
		/* 确保内容两端分布 */
		background-color: #F5F5F5;
		border-radius: 40rpx;
		padding: 0 0 0 30rpx;
		/* 左侧留出文字间距 */
		height: 72rpx;
		overflow: hidden;
	}

	.search-placeholder {
		font-size: 26rpx;
		color: #999;
	}

	.search-confirm-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		background-color: #42b983;
		/* 使用主题绿 */
		height: 100%;
		padding: 0 30rpx;
		transition: opacity 0.2s;
	}

	.search-confirm-btn:active {
		opacity: 0.8;
	}

	.search-icon {
		font-size: 28rpx;
		margin-right: 8rpx;
	}

	.search-text {
		font-size: 26rpx;
		color: #FFFFFF;
		font-weight: bold;
	}

	.post-list {
		padding: 20rpx;
	}

	.post-card {
		background-color: #FFFFFF;
		border-radius: 16rpx;
		padding: 30rpx;
		margin-bottom: 20rpx;
	}

	.post-header {
		display: flex;
		align-items: center;
		margin-bottom: 20rpx;
	}

	.avatar {
		width: 80rpx;
		height: 80rpx;
		border-radius: 50%;
		margin-right: 20rpx;
		background-color: #EEE;
		flex-shrink: 0;
	}

	.user-info {
		display: flex;
		flex-direction: column;
	}

	.nickname {
		font-size: 30rpx;
		font-weight: bold;
		color: #333;
	}

	.time {
		font-size: 24rpx;
		color: #999;
		margin-top: 6rpx;
	}

	.content-text {
		font-size: 30rpx;
		color: #333;
		line-height: 1.6;
	}

	.tags-wrap {
		margin-top: 16rpx;
		display: flex;
		flex-wrap: wrap;
		gap: 16rpx;
	}

	.tag {
		font-size: 24rpx;
		color: #42b983;
		background-color: rgba(66, 185, 131, 0.1);
		padding: 6rpx 20rpx;
		border-radius: 20rpx;
	}

	.post-footer {
		display: flex;
		justify-content: space-between;
		margin-top: 24rpx;
		padding-top: 24rpx;
		border-top: 2rpx solid #F5F5F5;
	}

	.action-btn {
		font-size: 28rpx;
	}

	.color-normal {
		color: #666;
	}

	.color-active {
		color: #42b983;
		font-weight: bold;
	}

	.loading-tip {
		text-align: center;
		font-size: 24rpx;
		color: #999;
		padding: 30rpx 0;
	}

	/* 悬浮发布按钮 */
	.add-post-btn {
		position: fixed;
		right: 40rpx;
		bottom: 180rpx;
		width: 100rpx;
		height: 100rpx;
		background-color: #42b983;
		border-radius: 50%;
		display: flex;
		justify-content: center;
		align-items: center;
		box-shadow: 0 4rpx 16rpx rgba(66, 185, 131, 0.4);
		z-index: 99;
	}

	.add-icon {
		color: #FFFFFF;
		font-size: 60rpx;
		font-weight: 300;
		line-height: 1;
	}
</style>