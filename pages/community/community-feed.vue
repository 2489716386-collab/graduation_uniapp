<template>
  <view class="feed-wrapper">
    <view class="search-box">
      <view class="search-inner">
        <text class="search-icon">🔍</text>
        <input class="search-input" type="text" v-model="searchQuery" placeholder="搜索宠物日常、经验、问答..." @confirm="onSearch" />
      </view>
    </view>

    <view class="post-list">
      <view class="post-card" v-for="(post, index) in myPostList" :key="index" @click.stop="goToDetail(post.id || post.postId)">
        <view class="post-header">
          <image class="avatar" :src="post.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
          <view class="user-info">
            <text class="nickname">{{ post.nickname }}</text>
            <text class="time">{{ post.createTime }}</text>
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
            <text :class="post.isLiked ? 'color-active' : 'color-normal'">
              {{ post.isLiked ? '已赞' : '点赞' }} {{ post.likesCount || post.likes || 0 }}
            </text>
          </view>
          <view class="action-btn">
            <text class="color-normal">评论 {{ post.commentsCount || post.comments || 0 }}</text>
          </view>
          <view class="action-btn" @click.stop="toggleFavorite(post)">
            <text :class="post.isFavorited ? 'color-active' : 'color-normal'">
              {{ post.isFavorited ? '已收藏' : '收藏' }}
            </text>
          </view>
        </view>
      </view>
      <view class="loading-tip">没有更多动态啦~</view>
    </view>
  </view>
</template>

<script>
export default {
  name: "CommunityFeed",
  data() {
    return {
      searchQuery: '',
      // 这里依然先放假数据供你预览，后期你可以用 fetchFeed 接口替换
      postList: [
        {
          id: 1, // 模拟后端返回的 postId
          nickname: "铲屎官大本营",
          avatar: "",
          createTime: "10分钟前",
          content: "今天带着我家金毛去草坪玩飞盘啦，天气真好！大家周末都带毛孩子去哪里玩呀？",
          tags: ["金毛", "日常打卡"],
          likesCount: 24,
          commentsCount: 5,
          isLiked: false,
          isFavorited: true
        }
      ]
    };
  },
  methods: {
    loadMore() {
      console.log("CommunityFeed 触底加载...");
    },
    refresh() {
      console.log("CommunityFeed 刷新数据...");
    },
    onSearch() {
      if (!this.searchQuery.trim()) return;
      uni.showToast({ title: '搜索: ' + this.searchQuery, icon: 'none' });
    },
	goToDetail(postId) {
	    if (!postId) return;
	    uni.navigateTo({
	        url: `/pages/community/post-detail?id=${postId}`
	    });
	},
    
    // ================== 点赞核心逻辑 ==================
    toggleLike(post) {
      const token = uni.getStorageSync('token');
      if (!token) {
        return uni.showToast({ title: '请先登录', icon: 'none' });
      }

      // 1. 乐观 UI 更新（先变色和数字，不卡顿）
      post.isLiked = !post.isLiked;
      // 兼容后端字段 likesCount 和前端假数据 likes
      if (post.likesCount !== undefined) {
        post.likesCount += post.isLiked ? 1 : -1;
      } else if (post.likes !== undefined) {
        post.likes += post.isLiked ? 1 : -1;
      }

      // 2. 发起后端请求
      const targetId = post.postId || post.id;
      uni.request({
        url: `http://localhost:8080/likes/toggle/${targetId}`,
        method: 'POST',
        header: { 'token': token },
        success: (res) => {
          if (res.data.code === 200) {
            uni.showToast({ title: res.data.msg || (post.isLiked ? '点赞成功' : '取消点赞成功'), icon: 'none' });
          } else {
            // 失败时状态回滚
            this.rollbackLike(post);
            uni.showToast({ title: res.data.msg || '操作失败', icon: 'none' });
          }
        },
        fail: () => {
          this.rollbackLike(post);
          uni.showToast({ title: '网络异常', icon: 'none' });
        }
      });
    },
    rollbackLike(post) {
      post.isLiked = !post.isLiked;
      if (post.likesCount !== undefined) post.likesCount += post.isLiked ? 1 : -1;
      else if (post.likes !== undefined) post.likes += post.isLiked ? 1 : -1;
    },

    // ================== 收藏核心逻辑 ==================
    toggleFavorite(post) {
      const token = uni.getStorageSync('token');
      if (!token) {
        return uni.showToast({ title: '请先登录', icon: 'none' });
      }

      // 1. 乐观 UI 更新
      post.isFavorited = !post.isFavorited;

      // 2. 发起后端请求
      const targetId = post.postId || post.id;
      uni.request({
        url: `http://localhost:8080/favorites/toggle/${targetId}`,
        method: 'POST',
        header: { 'token': token },
        success: (res) => {
          if (res.data.code === 200) {
            uni.showToast({ title: post.isFavorited ? '已收藏' : '已取消收藏', icon: 'none' });
          } else {
            post.isFavorited = !post.isFavorited; // 回滚
            uni.showToast({ title: res.data.msg || '操作失败', icon: 'none' });
          }
        },
        fail: () => {
          post.isFavorited = !post.isFavorited; // 回滚
          uni.showToast({ title: '网络异常', icon: 'none' });
        }
      });
    }
  }
}
</script>

<style scoped>
/* 保持原有样式不变 */
.feed-wrapper { width: 100%; }
.search-box { padding: 20rpx 30rpx; background-color: #FFFFFF; }
.search-inner { display: flex; align-items: center; background-color: #F5F5F5; border-radius: 40rpx; padding: 12rpx 30rpx; height: 64rpx; }
.search-icon { font-size: 28rpx; margin-right: 16rpx; }
.search-input { flex: 1; font-size: 28rpx; }
.post-list { padding: 20rpx; }
.post-card { background-color: #FFFFFF; border-radius: 16rpx; padding: 30rpx; margin-bottom: 20rpx; }
.post-header { display: flex; align-items: center; margin-bottom: 20rpx; }
.avatar { width: 80rpx; height: 80rpx; border-radius: 50%; margin-right: 20rpx; background-color: #EEE; }
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
.loading-tip { text-align: center; font-size: 24rpx; color: #999; padding: 30rpx 0; }
</style>