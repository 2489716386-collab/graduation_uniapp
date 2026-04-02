<template>
	<view class="personal-wrapper">
		<view class="profile-header">
			<view class="cover-wrapper" @click.stop="changeCoverImage">
				<image class="cover-img" :src="userInfo.coverImage" mode="aspectFill"></image>
				<view class="cover-tip" v-if="isLoggedIn">点击更换背景</view>
			</view>
			<view class="profile-user-info">
				<text class="profile-nickname">{{ userInfo.nickname }}</text>
				<image class="profile-avatar" :src="userInfo.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
			</view>
		</view>

		<view class="profile-bottom">
			<view class="notice-btn" @click.stop="goToNotices">
			    消息通知 
			    <text class="badge" v-if="unreadCount > 0">
			        {{ unreadCount > 99 ? '99+' : unreadCount }}
			    </text>
			</view>
			<text class="signature">{{ userInfo.signature }}</text>
		</view>

		<view class="personal-tabs">
			<view class="p-tab" :class="{ 'active': personalTab === 'posts' }" @click.stop="switchTab('posts')">动态</view>
			<view class="p-tab" :class="{ 'active': personalTab === 'likes' }" @click.stop="switchTab('likes')">喜欢</view>
			<view class="p-tab" :class="{ 'active': personalTab === 'favorites' }" @click.stop="switchTab('favorites')">收藏</view>
		</view>

		<view class="personal-list">
			
      <view v-show="personalTab === 'posts'">
				<view v-if="myPostList.length === 0" class="empty-state">你还没有发布过动态哦~</view>
				<view class="post-card" v-for="(post, index) in myPostList" :key="'p'+index" @click="goToDetail(post.postId)">
					<view class="post-header">
						<image class="avatar" :src="userInfo.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
						<view class="user-info">
							<text class="nickname">{{ userInfo.nickname }}</text>
							<text class="time">{{ post.createTime }}</text>
						</view>
					</view>
					<view class="post-body">
						<text class="content-text">{{ post.content }}</text>
					</view>
					<view class="post-footer">
						<view class="action-btn" @click.stop="toggleLikeForMyPost(post)">
							<text :class="post.isLiked ? 'color-active' : 'color-normal'">{{ post.isLiked ? '已赞' : '点赞' }} {{ post.likeCount || post.likes || 0 }}</text>
						</view>
						<view class="action-btn">
							<text class="color-normal">评论 {{ post.commentCount || post.comments || 0 }}</text>
						</view>
						<view class="action-btn" @click.stop="deleteMyPost(post.postId)">
							<text style="color: #FF4D4F;">删除</text>
						</view>
					</view>
				</view>
			</view>

      <view v-show="personalTab === 'likes'">
        <view v-if="myLikeList.length === 0" class="empty-state">还没有喜欢的动态~</view>
        <view class="post-card" v-for="(post, index) in myLikeList" :key="'l'+index" @click="goToDetail(post.postId)">
					<view class="post-header">
					    <image class="avatar" :src="post.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
					    <view class="user-info">
					        <text class="nickname">{{ post.nickname || '用户_' + post.userId }}</text>
					        <text class="time">{{ post.createTime }}</text>
					    </view>
					</view>
					<view class="post-body">
						<text class="content-text">{{ post.content }}</text>
					</view>
					<view class="post-footer">
						<view class="action-btn" @click.stop="toggleLikeInList(post, index, myLikeList)">
							<text class="color-active">已赞 {{ post.likeCount || post.likes || 0 }}</text>
						</view>
						<view class="action-btn">
							<text class="color-normal">评论 {{ post.commentCount || post.comments || 0 }}</text>
						</view>
					</view>
				</view>
      </view>

      <view v-show="personalTab === 'favorites'">
        <view v-if="myFavList.length === 0" class="empty-state">还没有收藏的动态~</view>
        <view class="post-card" v-for="(post, index) in myFavList" :key="'f'+index" @click="goToDetail(post.postId)">
					<view class="post-header">
					    <image class="avatar" :src="post.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
					    <view class="user-info">
					        <text class="nickname">{{ post.nickname || '用户_' + post.userId }}</text>
					        <text class="time">{{ post.createTime }}</text>
					    </view>
					</view>
					<view class="post-body">
						<text class="content-text">{{ post.content }}</text>
					</view>
					<view class="post-footer">
						<view class="action-btn">
							<text class="color-normal">点赞 {{ post.likeCount || post.likes || 0 }}</text>
						</view>
						<view class="action-btn">
							<text class="color-normal">评论 {{ post.commentCount || post.comments || 0 }}</text>
						</view>
						<view class="action-btn" @click.stop="toggleFavoriteInList(post, index, myFavList)">
							<text class="color-active">已收藏</text>
						</view>
					</view>
				</view>
      </view>

		</view>
	</view>
</template>

<script>
export default {
	name: "PersonalCenter",
	data() {
		return {
			personalTab: 'posts',
			unreadCount: 0,
			isLoggedIn: false,
			userInfo: {
				nickname: "未登录",
				avatar: "/static/default-avatar.png",
				signature: "登录后查看更多精彩内容~",
				coverImage: "https://mp-aa1e3790-0b44-4bfc-afa7-9b9be364f376.cdn.bspapp.com/cloudstorage/f07035fc-384f-4743-b642-32109a13f0ff.jpg"
			},
      myPostList: [],
			myPostPage: 1,
      myLikeList: [],
      myLikePage: 1,
      myFavList: [],
      myFavPage: 1
		};
	},
	methods: {
		refresh() {
			const token = uni.getStorageSync('token');
			if (token) {
				this.isLoggedIn = true;
				this.fetchUserInfo();
				this.fetchMyPosts(true);
        if (this.personalTab === 'likes') this.fetchMyLikes(true);
        if (this.personalTab === 'favorites') this.fetchMyFavorites(true);
			} else {
				this.isLoggedIn = false;
				this.myPostList = [];
        this.myLikeList = [];
        this.myFavList = [];
			}
		},
		loadMore() {
			if (!this.isLoggedIn) return;
      if (this.personalTab === 'posts') {
				this.myPostPage++;
				this.fetchMyPosts(false);
			} else if (this.personalTab === 'likes') {
        this.myLikePage++;
        this.fetchMyLikes(false);
      } else if (this.personalTab === 'favorites') {
        this.myFavPage++;
        this.fetchMyFavorites(false);
      }
		},
		goToDetail(postId) {
		    if (!postId) return;
		    uni.navigateTo({ url: `/pages/community/post-detail?id=${postId}` });
		},
    switchTab(tab) {
      this.personalTab = tab;
      if (!this.isLoggedIn) return;
      if (tab === 'likes' && this.myLikeList.length === 0) {
        this.fetchMyLikes(true);
      } else if (tab === 'favorites' && this.myFavList.length === 0) {
        this.fetchMyFavorites(true);
      }
    },
		handleAuthError(res) {
			if (res.data.code === 401) {
				uni.removeStorageSync('token');
				this.isLoggedIn = false;
				uni.showToast({ title: res.data.msg || '登录已过期', icon: 'none' });
				setTimeout(() => { uni.navigateTo({ url: '/pages/login/login' }); }, 1000);
				return true;
			}
			return false;
		},
		fetchUserInfo() {
			uni.request({
				url: 'http://localhost:8080/users/user/profile',
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (this.handleAuthError(res)) return;
					if (res.data.code === 200) {
						const data = res.data.data;
						this.userInfo = {
							nickname: data.nickname || '微信用户',
							avatar: data.avatarUrl || '/static/default-avatar.png',
							signature: data.bio || '这个人很懒，什么都没留下~',
							coverImage: data.coverImage || this.userInfo.coverImage
						};
					}
				}
			});
		},
    processPostData(records) {
      const posts = records || [];
      posts.forEach(post => {
        if (typeof post.tags === 'string' && post.tags.trim() !== '') {
          post.tags = post.tags.split(',');
        } else if (!post.tags) {
          post.tags = [];
        }
      });
      return posts;
    },
	fetchUnreadCount() {
	    uni.request({
	        url: 'http://localhost:8080/interaction-notifications/unread-count',
	        method: 'GET',
	        header: { 'token': uni.getStorageSync('token') },
	        success: (res) => {
	            if (res.data.code === 200) {
	                this.unreadCount = res.data.data;
	            }
	        }
	    });
	},
	// 在 refresh() 方法里调用它
	refresh() {
	    const token = uni.getStorageSync('token');
	    if (token) {
	        this.isLoggedIn = true;
	        this.fetchUserInfo();
	        this.fetchUnreadCount(); // 获取未读数
	        // ...
	    }
	},
		fetchMyPosts(isRefresh = false) {
			if (isRefresh) { this.myPostPage = 1; this.myPostList = []; }
			uni.request({
				url: `http://localhost:8080/community-posts/user/my?pageNum=${this.myPostPage}&pageSize=10`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (this.handleAuthError(res)) return;
					if (res.data.code === 200) {
            const newPosts = this.processPostData(res.data.data.records);
						this.myPostList = this.myPostList.concat(newPosts);
					}
				}
			});
		},
    fetchMyLikes(isRefresh = false) {
      if (isRefresh) { this.myLikePage = 1; this.myLikeList = []; }
			uni.request({
				url: `http://localhost:8080/likes/my?pageNum=${this.myLikePage}&pageSize=10`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (this.handleAuthError(res)) return;
					if (res.data.code === 200) {
            const newPosts = this.processPostData(res.data.data.records);
						this.myLikeList = this.myLikeList.concat(newPosts);
					}
				}
			});
    },
    fetchMyFavorites(isRefresh = false) {
      if (isRefresh) { this.myFavPage = 1; this.myFavList = []; }
			uni.request({
				url: `http://localhost:8080/favorites/my?pageNum=${this.myFavPage}&pageSize=10`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (this.handleAuthError(res)) return;
					if (res.data.code === 200) {
            const newPosts = this.processPostData(res.data.data.records);
						this.myFavList = this.myFavList.concat(newPosts);
					}
				}
			});
    },
		// 新增：专属给“我的动态”列表使用的点赞逻辑
		toggleLikeForMyPost(post) {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });
			uni.request({
				url: `http://localhost:8080/likes/toggle/${post.postId || post.id}`,
				method: 'POST',
				header: { 'token': token },
				success: (res) => {
					if (res.data.code === 200) {
						post.isLiked = res.data.data; // 后端返回 true/false
						// 动态修改点赞数量
						if (post.isLiked) {
							post.likeCount = (post.likeCount || 0) + 1;
						} else {
							post.likeCount = Math.max(0, (post.likeCount || 0) - 1);
						}
					}
				}
			});
		},
    toggleLikeInList(post, index, listArray) {
      uni.request({
        url: `http://localhost:8080/likes/toggle/${post.postId}`,
        method: 'POST',
        header: { 'token': uni.getStorageSync('token') },
        success: (res) => {
          if (res.data.code === 200) {
            if (res.data.data === false) {
              uni.showToast({ title: '已取消点赞', icon: 'none' });
              listArray.splice(index, 1);
            }
          }
        }
      });
    },
    toggleFavoriteInList(post, index, listArray) {
      uni.request({
        url: `http://localhost:8080/favorites/toggle/${post.postId}`,
        method: 'POST',
        header: { 'token': uni.getStorageSync('token') },
        success: (res) => {
          if (res.data.code === 200) {
            if (res.data.data === false) {
              uni.showToast({ title: '已取消收藏', icon: 'none' });
              listArray.splice(index, 1);
            }
          }
        }
      });
    },
		deleteMyPost(postId) {
			uni.showModal({
				title: '提示',
				content: '确定要删除这条动态吗？',
				success: (res) => {
					if (res.confirm) {
						uni.request({
							url: `http://localhost:8080/community-posts/user/delete/${postId}`,
							method: 'DELETE',
							header: { 'token': uni.getStorageSync('token') },
							success: (delRes) => {
								if (this.handleAuthError(delRes)) return;
								if (delRes.data.code === 200) {
									uni.showToast({ title: '删除成功', icon: 'success' });
									this.myPostList = this.myPostList.filter(item => item.postId !== postId);
								}
							}
						});
					}
				}
			});
		},
		changeCoverImage() {
			if (!this.isLoggedIn) return uni.showToast({ title: '请先登录', icon: 'none' });
			uni.chooseImage({
				count: 1,
				sizeType: ['compressed'],
				sourceType: ['album', 'camera'],
				success: (res) => {
					this.uploadCoverToUniCloud(res.tempFilePaths[0]);
				}
			});
		},
		uploadCoverToUniCloud(filePath) {
			uni.showLoading({ title: '上传中...', mask: true });
			const fileExt = filePath.split('.').pop() || 'jpg';
			const fileName = `cover_${Date.now()}.${fileExt}`;
			uniCloud.uploadFile({
				filePath: filePath,
				cloudPath: fileName,
				success: (res) => { this.updateCoverToBackend(res.fileID); },
				fail: () => { uni.hideLoading(); uni.showToast({ title: '上传失败', icon: 'none' }); }
			});
		},
		updateCoverToBackend(url) {
			uni.request({
				url: 'http://localhost:8080/users/user/update',
				method: 'PUT',
				header: { 'token': uni.getStorageSync('token') },
				data: { coverImage: url },
				success: (res) => {
					uni.hideLoading();
					if (this.handleAuthError(res)) return;
					if (res.data.code === 200) {
						this.userInfo.coverImage = url;
						uni.showToast({ title: '背景已更新', icon: 'success' });
					}
				}
			});
		},
		goToNotices() {
					if (!this.isLoggedIn) {
		                return uni.navigateTo({ url: '/pages/login/login' });
		            }
		            // 1. 跳转到真实的通知列表页面
					uni.navigateTo({ url: '/pages/community/notices' });
		            
		            // 2. 【关键体验】点击的同时，前端立刻把未读红点清零，给用户“秒读”的快感
		            this.unreadCount = 0;
				}
	}
}
</script>

<style scoped>
.personal-wrapper { width: 100%; }

/* 优化：减小底部间距，让下方内容更靠近 */
.profile-header { position: relative; background-color: #FFFFFF; margin-bottom: 40rpx; }
.cover-wrapper { position: relative; width: 100%; height: 400rpx; background-color: #f0f0f0; }
.cover-img { width: 100%; height: 100%; display: block; }
.cover-tip { position: absolute; top: 30rpx; right: 30rpx; background-color: rgba(0, 0, 0, 0.4); color: #FFF; font-size: 22rpx; padding: 6rpx 16rpx; border-radius: 20rpx; border: 1px solid rgba(255, 255, 255, 0.5); }
.profile-user-info { position: absolute; right: 30rpx; bottom: -40rpx; display: flex; align-items: flex-end; }
.profile-nickname { font-size: 36rpx; color: #FFFFFF; font-weight: bold; margin-right: 30rpx; text-shadow: 2rpx 2rpx 4rpx rgba(0, 0, 0, 0.6); margin-bottom: 40rpx; }
.profile-avatar { width: 130rpx; height: 130rpx; border-radius: 16rpx; border: 4rpx solid #FFFFFF; box-shadow: 0 4rpx 10rpx rgba(0, 0, 0, 0.1); background-color: #EEE; flex-shrink: 0; }

/* 优化：垂直居中对齐，缩减底部内边距 */
.profile-bottom { background-color: #FFFFFF; padding: 0 30rpx 10rpx 30rpx; display: flex; justify-content: space-between; align-items: center; }

/* 优化：修改左右边距，并将文字靠右，与头像形成视觉呼应 */
.signature { font-size: 26rpx; color: #666; flex: 1; margin-left: 20rpx; text-align: right; line-height: 1.5; }

.notice-btn { font-size: 24rpx; background-color: #F5F5F5; color: #333; padding: 12rpx 24rpx; border-radius: 30rpx; position: relative; flex-shrink: 0; }
.badge { position: absolute; top: -8rpx; right: -8rpx; background-color: #FF4D4F; color: #FFF; font-size: 20rpx; padding: 2rpx 10rpx; border-radius: 20rpx; }

/* 优化：去掉了 margin-top 让 Tab 栏紧贴着上方信息 */
.personal-tabs { display: flex; background-color: #FFFFFF; border-bottom: 2rpx solid #F5F5F5; margin-top: 0; }
.p-tab { flex: 1; text-align: center; font-size: 28rpx; color: #666; padding: 24rpx 0; position: relative; }
.p-tab.active { color: #333; font-weight: bold; }
.p-tab.active::after { content: ''; position: absolute; bottom: 0; left: 50%; transform: translateX(-50%); width: 40rpx; height: 6rpx; background-color: #42b983; border-radius: 4rpx; }

.personal-list { padding: 20rpx; }
.empty-state { text-align: center; padding: 100rpx 0; color: #999; font-size: 28rpx; }
.post-card { background-color: #FFFFFF; border-radius: 16rpx; padding: 30rpx; margin-bottom: 20rpx; }
.post-header { display: flex; align-items: center; margin-bottom: 20rpx; }
.avatar { width: 80rpx; height: 80rpx; border-radius: 50%; margin-right: 20rpx; background-color: #EEE; flex-shrink: 0; }
.user-info { display: flex; flex-direction: column; }
.nickname { font-size: 30rpx; font-weight: bold; color: #333; }
.time { font-size: 24rpx; color: #999; margin-top: 6rpx; }
.content-text { font-size: 30rpx; color: #333; line-height: 1.6; }
.tags-wrap { margin-top: 16rpx; display: flex; flex-wrap: wrap; gap: 16rpx; }
.tag { font-size: 24rpx; color: #42b983; background-color: rgba(66, 185, 131, 0.1); padding: 6rpx 20rpx; border-radius: 20rpx; }
.post-footer { display: flex; justify-content: space-between; margin-top: 24rpx; padding-top: 24rpx; border-top: 2rpx solid #F5F5F5; }
.action-btn { font-size: 28rpx; }
.color-normal { color: #666; }
.color-active { color: #42b983; font-weight: bold; }
</style>