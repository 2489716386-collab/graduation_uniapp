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
				confirmedKeyword: '', // 用于锁定当前真正执行搜索的关键词
				localHistory: [], // 本地缓存的历史记录
				sortType: 'hot',
				results: [],
				hasSearched: false
			}
		},
		computed: {
			// 动态生成防串号的本地缓存 Key
			historyStorageKey() {
				// 1. 尝试从用户信息中获取唯一ID
				const userInfo = uni.getStorageSync('userInfo');
				let uid = userInfo ? (userInfo.id || userInfo.userId) : uni.getStorageSync('userId');
				
				// 2. 兜底方案：如果本地没存 userId，就拿当前登录的 token 的后10位作为临时隔离标识
				if (!uid) {
					const token = uni.getStorageSync('token') || 'guest';
					uid = token !== 'guest' ? token.slice(-10) : 'guest';
				}
				
				return `post_search_history_${uid}`;
			}
		},
		onLoad(options) {
			// 支持从其他页面携带关键词跳转进来，例如：/pages/community/dynamic/search?kw=狗狗
			if (options.kw) {
				this.keyword = options.kw;
				this.doSearch();
			}
		},
		onShow() {
			// 页面进入时，加载属于当前账号的历史记录
			this.localHistory = uni.getStorageSync(this.historyStorageKey) || [];
		},
		methods: {
			doSearch() {
				const kw = this.keyword.trim();
				if (!kw) {
					uni.showToast({ title: '请输入搜索内容', icon: 'none' });
					return;
				}

				// 关键点：锁定关键词，防止用户搜索后清空输入框，再点击“最新发布”时导致参数为空
				this.confirmedKeyword = kw; 

				// 1. 更新本地历史逻辑
				let history = this.localHistory;
				history = history.filter(i => i !== kw); // 去重
				history.unshift(kw); // 插到最前面
				if (history.length > 10) history.pop(); // 只留10条

				this.localHistory = history;
				// 存入当前账号专属的 Key 中
				uni.setStorageSync(this.historyStorageKey, history);

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
							// 只清空当前账号的缓存记录，不影响手机上的其他账号
							uni.removeStorageSync(this.historyStorageKey);
						}
					}
				});
			},
			switchSort(type) {
				if (this.sortType === type) return;
				this.sortType = type;
				// 切换排序时，必须基于已确认的关键词去请求
				if (this.confirmedKeyword) {
					this.fetchResults();
				}
			},
			fetchResults() {
				uni.showLoading({
					title: '搜索中...'
				});
				uni.request({
					url: 'http://localhost:8080/community-posts/search',
					method: 'GET',
					data: {
						// 核心修改：这里一定要用 confirmedKeyword，不能用 this.keyword.trim()
						keyword: this.confirmedKeyword, 
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