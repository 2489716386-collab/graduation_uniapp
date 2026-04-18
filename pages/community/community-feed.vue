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
						<text class="nickname">{{ post.nickname || '神秘宠友'}}</text>
						<text class="time">{{ post.createTime }}</text>
					</view>
					<view class="recommend-badge" v-if="post.recommendScore && post.recommendScore > 0">
						<text>🔥 匹配度 {{ post.recommendScore }}</text>
					</view>
					<view class="more-icon" @click.stop="handlePostOptions(post.id)">
					  <uni-icons type="more-filled" size="20"></uni-icons>
					</view>
				</view>

				<view class="post-body">
				    <text class="content-text">{{ post.content }}</text>
				    
				    <view class="image-grid" v-if="post.imageList && post.imageList.length > 0">
				        <image 
				            class="grid-img" 
				            v-for="(img, idx) in post.imageList" 
				            :key="idx" 
				            :src="img" 
				            mode="aspectFill"
				            @click.stop="previewImage(img, post.imageList)">
				        </image>
				    </view>
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
			        url: 'http://localhost:8080/community-posts/recommend',
			        method: 'GET',
			        header: {
			            'token': token
			        },
			        success: (res) => {
			            if (res.data.code === 200) {
			                let records = res.data.data;
			                // 👇 新增的数据解析逻辑 👇
			                records.forEach(item => {
			                    if (item.mediaUrls) {
			                        try {
			                            // 将后端的 JSON 字符串解析为数组
			                            item.imageList = JSON.parse(item.mediaUrls);
			                        } catch (e) {
			                            item.imageList = [];
			                        }
			                    } else {
			                        item.imageList = [];
			                    }
			                });
			                // 赋值给绑定的数组
			                this.postList = records;
			            }
			        }
			    });
			},
			
			// 新增一个方法，用于点击图片放大预览 👇
			previewImage(currentUrl, urlList) {
			    uni.previewImage({
			        current: currentUrl, // 当前显示图片的http链接
			        urls: urlList // 需要预览的图片http链接列表
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
			handlePostOptions(postId) {
			    uni.showActionSheet({
			      itemList: ['举报该动态'],
			      itemColor: '#ff0000', // 设为红色以示警告
			      success: (res) => {
			        if (res.tapIndex === 0) {
			          this.submitReport(postId, 'POST'); // 注意这里对应你后端的 TargetType.POST
			        }
			      }
			    });
			  },
			
			  // 提交举报的统一方法
			  submitReport(targetId, targetType) {
			    uni.showModal({
			      title: '提示',
			      content: '确认举报该内容吗？',
			      success: (res) => {
			        if (res.confirm) {
			          // 调用你后端的 ReportsController 接口
			          uni.request({
			            url: '你的后端基础路径/reports', // 替换为你实际的后端新增举报接口
			            method: 'POST',
			            header: {
			              'Authorization': uni.getStorageSync('token') // 携带用户的 token
			            },
			            data: {
			              targetId: targetId,
			              targetType: targetType,
			              reason: '涉嫌违规内容' // 后期你可以做一个弹窗让用户选原因，这里先写死测试
			            },
			            success: (response) => {
			              if(response.data.code === 200 || response.data.code === 1) { // 根据你的 Result.java 成功码调整
			                uni.showToast({ title: '举报成功，感谢您的反馈', icon: 'none' });
			              } else {
			                uni.showToast({ title: response.data.msg || '举报失败', icon: 'none' });
			              }
			            }
			          });
			        }
			      }
			    });
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
	/* 朋友圈九宫格的图片样式  */
	.image-grid {
	    display: flex;
	    flex-wrap: wrap;
	    gap: 12rpx; /* 图片之间的间距 */
	    margin-top: 20rpx;
	}
	
	.grid-img {
	    width: 216rpx;
	    height: 216rpx;
	    border-radius: 12rpx;
	    background-color: #F8F8F8;
	}
	.header-right {
	  display: flex;
	  align-items: center;
	  margin-left: auto; /* 将整个右侧区域推到最右边 */
	  gap: 20rpx; /* 徽章和三个点之间的间距 */
	}
	.more-icon {
	  padding: 10rpx; /* 增加点击区域大小，方便用户手指点击 */
	  display: flex;
	  align-items: center;
	  justify-content: center;
	}
	.more-icon:active {
	  opacity: 0.7; /* 点击时的按压反馈 */
	}
	/* 👆 样式结束 👆 */
</style>