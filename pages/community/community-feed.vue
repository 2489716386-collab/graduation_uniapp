<template>
  <view class="feed-wrapper">
    <view class="search-box">
      <view class="search-inner">
        <text class="search-icon">🔍</text>
        <input class="search-input" type="text" v-model="searchQuery" placeholder="搜索宠物日常、经验、问答..." @confirm="onSearch" />
      </view>
    </view>

    <view class="post-list">
      <view class="post-card" v-for="(post, index) in postList" :key="index">
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
          <view class="action-btn" @click="toggleLike(post)">
            <text :class="post.isLiked ? 'color-active' : 'color-normal'">{{ post.isLiked ? '已赞' : '点赞' }} {{ post.likes }}</text>
          </view>
          <view class="action-btn">
            <text class="color-normal">评论 {{ post.comments }}</text>
          </view>
          <view class="action-btn" @click="toggleFavorite(post)">
            <text :class="post.isFavorited ? 'color-active' : 'color-normal'">{{ post.isFavorited ? '已收藏' : '收藏' }}</text>
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
      // 模拟数据，后期可替换为后端请求
      postList: [
        {
          id: 1,
          nickname: "铲屎官大本营",
          avatar: "",
          createTime: "10分钟前",
          content: "今天带着我家金毛去草坪玩飞盘啦，天气真好！大家周末都带毛孩子去哪里玩呀？",
          tags: ["金毛", "日常打卡"],
          likes: 24,
          comments: 5,
          isLiked: false,
          isFavorited: true
        }
      ]
    };
  },
  methods: {
    // 供父组件调用的触底加载方法
    loadMore() {
      console.log("CommunityFeed 触底加载...");
      // TODO: 请求下一页数据
    },
    // 供父组件调用的刷新方法
    refresh() {
      console.log("CommunityFeed 刷新数据...");
      // TODO: 重新请求第一页数据
    },
    onSearch() {
      if (!this.searchQuery.trim()) return;
      uni.showToast({ title: '搜索: ' + this.searchQuery, icon: 'none' });
    },
    toggleLike(post) {
      post.isLiked = !post.isLiked;
      post.likes += post.isLiked ? 1 : -1;
    },
    toggleFavorite(post) {
      post.isFavorited = !post.isFavorited;
      uni.showToast({ title: post.isFavorited ? '已收藏' : '取消收藏', icon: 'none' });
    }
  }
}
</script>

<style scoped>
/* 这里的样式只对当前组件生效 */
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