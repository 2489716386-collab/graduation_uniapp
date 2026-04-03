<template>
	<view class="search-page">
		<view class="search-header">
			<view class="search-bar">
				<text class="icon">🔍</text>
				<input class="input" v-model="keyword" placeholder="输入你想找的宠物内容..." @confirm="doSearch" focus />
				<text class="clear" v-if="keyword" @click="keyword = ''">✖</text>
			</view>
			<text class="search-btn" @click="doSearch">搜索</text>
		</view>
		<view class="history-container" v-if="!hasSearched && localHistory.length > 0">
			<view class="history-header">
				<text class="title">历史搜索</text>
				<text class="delete-icon" @click="clearLocalHistory">🗑️ 清空</text>
			</view>
			<view class="history-tags">
				<text class="tag" v-for="(item, index) in localHistory" :key="index" @click="quickSearch(item)">
					{{ item }}
				</text>
			</view>
		</view>

		<view class="sort-tabs" v-if="hasSearched">
			<view class="tab" :class="{active: sortType === 'hot'}" @click="switchSort('hot')">🔥 综合热度</view>
			<view class="tab" :class="{active: sortType === 'time'}" @click="switchSort('time')">🕒 最新发布</view>
		</view>

		<scroll-view scroll-y class="result-list" v-if="hasSearched">
			<view class="post-card" v-for="(post, index) in results" :key="index"
				@click="goToDetail(post.id || post.postId)">
				<view class="post-header">
					<image class="avatar" :src="post.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
					<view class="user-info">
						<text class="nickname">{{ post.nickname }}</text>
						<text class="time">{{ post.createTime }}</text>
					</view>
				</view>
				<view class="post-body">
					<text class="content-text">{{ post.content }}</text>
				</view>
				<view class="post-footer">
					<text class="color-normal">点赞 {{ post.likeCount || 0 }}</text>
					<text class="color-normal">评论 {{ post.commentCount || 0 }}</text>
				</view>
			</view>

			<view class="empty-tip" v-if="results.length === 0">
				<text>没有找到相关动态哦~</text>
			</view>
		</scroll-view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				keyword: '',
				localHistory: [], // 本地缓存的历史记录
				sortType: 'hot',
				results: [],
				hasSearched: false
			}
		},
		onShow() {
			// 页面进入时加载本地历史
			this.localHistory = uni.getStorageSync('post_search_history') || [];
		},
		methods: {
			doSearch() {
				const kw = this.keyword.trim();
				if (!kw) return;

				// 1. 更新本地历史逻辑
				let history = this.localHistory;
				history = history.filter(i => i !== kw); // 去重
				history.unshift(kw); // 插到最前面
				if (history.length > 10) history.pop(); // 只留10条

				this.localHistory = history;
				uni.setStorageSync('post_search_history', history);

				// 2. 执行后端搜索请求
				this.hasSearched = true;
				this.fetchResults();
			},
			quickSearch(word) {
				this.keyword = word;
				this.doSearch();
			},
			clearLocalHistory() {
				uni.showModal({
					title: '提示',
					content: '确定清空本地搜索历史吗？',
					success: (res) => {
						if (res.confirm) {
							this.localHistory = [];
							uni.removeStorageSync('post_search_history');
							// 注意：这里没有请求后端删除接口，数据库记录依然保留供推荐算法使用
						}
					}
				});
			},
			switchSort(type) {
				if (this.sortType === type) return;
				this.sortType = type;
				this.fetchResults();
			},
			fetchResults() {
				uni.showLoading({
					title: '搜索中...'
				});
				uni.request({
					url: 'http://localhost:8080/community-posts/search',
					method: 'GET',
					data: {
						keyword: this.keyword.trim(),
						sort: this.sortType
					},
					header: {
						'token': uni.getStorageSync('token')
					},
					success: (res) => {
						uni.hideLoading();
						if (res.data.code === 200) {
							this.results = res.data.data;
						} else {
							uni.showToast({
								title: res.data.msg,
								icon: 'none'
							});
						}
					},
					fail: () => {
						uni.hideLoading();
						uni.showToast({
							title: '网络异常',
							icon: 'none'
						});
					}
				});
			},
			goToDetail(postId) {
				if (!postId) return;
				uni.navigateTo({
					url: `/pages/community/post-detail?id=${postId}`
				});
			}
		}
	}
</script>

<style scoped>
	/* 搜索页特有样式 */
	.search-page {
		min-height: 100vh;
		background-color: #F5F5F5;
		display: flex;
		flex-direction: column;
	}

	.search-header {
		display: flex;
		align-items: center;
		padding: 20rpx 30rpx;
		background-color: #FFF;
	}

	.search-bar {
		flex: 1;
		display: flex;
		align-items: center;
		background-color: #F8F8F8;
		border-radius: 36rpx;
		padding: 0 24rpx;
		height: 72rpx;
	}

	.icon {
		font-size: 28rpx;
		color: #999;
		margin-right: 12rpx;
	}

	.input {
		flex: 1;
		font-size: 28rpx;
	}

	.clear {
		color: #CCC;
		padding: 10rpx;
	}

	.search-btn {
		font-size: 30rpx;
		color: #42b983;
		margin-left: 24rpx;
		font-weight: bold;
	}

	.sort-tabs {
		display: flex;
		background-color: #FFF;
		border-top: 1px solid #EEE;
		padding: 20rpx 0;
	}

	.tab {
		flex: 1;
		text-align: center;
		font-size: 28rpx;
		color: #666;
	}

	.tab.active {
		color: #42b983;
		font-weight: bold;
	}

	.result-list {
		flex: 1;
		padding: 20rpx;
		box-sizing: border-box;
		height: 0;
	}

	.empty-tip {
		text-align: center;
		color: #999;
		margin-top: 100rpx;
		font-size: 28rpx;
	}

	/* 复用帖子卡片基础样式 */
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

	.post-footer {
		display: flex;
		justify-content: space-around;
		margin-top: 24rpx;
		padding-top: 24rpx;
		border-top: 2rpx solid #F5F5F5;
	}

	.color-normal {
		color: #666;
		font-size: 26rpx;
	}
	.history-container {
			padding: 30rpx;
			background-color: #FFFFFF;
			margin-bottom: 20rpx;
		}
	
		.history-header {
			display: flex;
			justify-content: space-between;
			align-items: center;
			margin-bottom: 24rpx;
		}
	
		.history-header .title {
			font-size: 30rpx;
			font-weight: bold;
			color: #333;
		}
	
		.delete-icon {
			font-size: 26rpx;
			color: #999;
		}
	
		.history-tags {
			display: flex;
			flex-wrap: wrap;
			gap: 20rpx; /* 标签之间的间距 */
		}
	
		.history-tags .tag {
			background-color: #F5F5F5;
			padding: 12rpx 32rpx;
			border-radius: 30rpx; /* 圆角标签 */
			font-size: 26rpx;
			color: #666;
		}
</style>