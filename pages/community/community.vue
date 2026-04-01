<template>
  <view class="container">
    
    <view class="custom-header">
      <view class="status-bar-placeholder"></view>
      <view class="nav-tabs">
        <text class="tab-item" :class="{ 'active': currentMainTab === 'community' }" @click="switchMainTab('community')">社区</text>
        <text class="tab-item" :class="{ 'active': currentMainTab === 'personal' }" @click="switchMainTab('personal')">个人</text>
      </view>
    </view>

    <view class="main-content">
      
      <view v-show="currentMainTab === 'community'" class="module-wrapper">
        <view class="search-box">
          <view class="search-inner">
            <text class="search-icon">🔍</text>
            <input class="search-input" type="text" v-model="searchQuery" placeholder="搜索宠物日常、经验、问答..." confirm-type="search" @confirm="onSearch" />
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

      <view v-show="currentMainTab === 'personal'" class="module-wrapper">
        
        <view class="profile-header">
          <image class="cover-img" src="https://images.unsplash.com/photo-1548767797-d8c844163c4c?auto=format&fit=crop&w=800&q=80" mode="aspectFill"></image>
          
          <view class="profile-user-info">
            <text class="profile-nickname">{{ userInfo.nickname }}</text>
            <image class="profile-avatar" :src="userInfo.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
          </view>
        </view>

        <view class="profile-bottom">
          <text class="signature">{{ userInfo.signature }}</text>
          <view class="notice-btn" @click="goToNotices">
            消息通知 <text class="badge" v-if="unreadCount > 0">{{ unreadCount }}</text>
          </view>
        </view>

        <view class="personal-tabs">
          <view class="p-tab" :class="{ 'active': personalTab === 'posts' }" @click="personalTab = 'posts'">动态</view>
          <view class="p-tab" :class="{ 'active': personalTab === 'likes' }" @click="personalTab = 'likes'">喜欢</view>
          <view class="p-tab" :class="{ 'active': personalTab === 'favorites' }" @click="personalTab = 'favorites'">收藏</view>
        </view>

        <view class="personal-list">
          <view v-if="personalTab === 'posts'" class="empty-state">你还没有发布过动态哦~</view>
          <view v-if="personalTab === 'likes'" class="empty-state">还没有喜欢的动态~</view>
          <view v-if="personalTab === 'favorites'" class="empty-state">还没有收藏的动态~</view>
        </view>

      </view>

    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      currentMainTab: 'community', 
      personalTab: 'posts', 
      searchQuery: '',
      unreadCount: 5, 
      
      userInfo: {
        nickname: "一只大芒果",
        avatar: "", 
        signature: "愿所有的毛孩子都能被世界温柔以待。"
      },

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
        },
        {
          id: 2,
          nickname: "喵星人研究所",
          avatar: "",
          createTime: "1小时前",
          content: "布偶猫最近肠胃不太好，换了无谷猫粮之后好像好转了。求推荐好用的宠物益生菌！",
          tags: ["宠物健康", "求助"],
          likes: 128,
          comments: 32,
          isLiked: true,
          isFavorited: false
        }
      ]
    };
  },
  // 替代 scroll-view 的原生触底加载钩子
  onReachBottom() {
    if (this.currentMainTab === 'community') {
      console.log("页面触底，加载下一页...");
      // TODO: 调用加载更多接口
    }
  },
  methods: {
    switchMainTab(tab) {
      this.currentMainTab = tab;
      // 切换标签时回到顶部
      uni.pageScrollTo({ scrollTop: 0, duration: 0 });
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
    },
    goToNotices() {
      this.unreadCount = 0;
      uni.showToast({ title: '进入消息列表', icon: 'none' });
    }
  }
}
</script>

<style scoped>
/* 全局页面背景色 */
page {
  background-color: #F8F8F8;
}

.container {
  width: 100%;
}

/* ================== 固定顶部导航 (绝对不乱的核心) ================== */
.custom-header {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  background-color: #FFFFFF;
  z-index: 999;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
}
.status-bar-placeholder {
  /* 自动获取手机状态栏高度，H5端会自动设为0 */
  height: var(--status-bar-height); 
  width: 100%;
}
.nav-tabs {
  height: 88rpx; /* 微信标准标题栏高度 */
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 80rpx;
}
.tab-item {
  font-size: 32rpx;
  color: #999;
  font-weight: 500;
  position: relative;
  height: 88rpx;
  line-height: 88rpx;
}
.tab-item.active {
  color: #333;
  font-weight: bold;
  font-size: 34rpx;
}
.tab-item.active::after {
  content: '';
  position: absolute;
  bottom: 10rpx;
  left: 50%;
  transform: translateX(-50%);
  width: 40rpx;
  height: 6rpx;
  background-color: #42b983;
  border-radius: 4rpx;
}

/* ================== 内容区包裹 ================== */
.main-content {
  /* 预留顶部 fixed 的高度：状态栏 + 导航栏88rpx */
  padding-top: calc(var(--status-bar-height) + 88rpx);
  width: 100%;
}

/* ================== 社区：搜索栏 ================== */
.search-box {
  padding: 20rpx 30rpx;
  background-color: #FFFFFF;
}
.search-inner {
  display: flex;
  align-items: center;
  background-color: #F5F5F5;
  border-radius: 40rpx;
  padding: 12rpx 30rpx;
  height: 64rpx;
}
.search-icon {
  font-size: 28rpx;
  margin-right: 16rpx;
}
.search-input {
  flex: 1;
  font-size: 28rpx;
}

/* ================== 社区：帖子卡片 ================== */
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
.action-btn { font-size: 28rpx; }
.color-normal { color: #666; }
.color-active { color: #42b983; font-weight: bold; }

.loading-tip {
  text-align: center;
  font-size: 24rpx;
  color: #999;
  padding: 30rpx 0;
}

/* ================== 个人模块：完美朋友圈头部 ================== */
.profile-header {
  position: relative;
  background-color: #FFFFFF;
  /* 留出下方空间给溢出的头像 */
  margin-bottom: 60rpx; 
}
.cover-img {
  width: 100%;
  height: 400rpx; 
  display: block;
}
.profile-user-info {
  position: absolute;
  right: 30rpx;
  /* 头像往下压，形成朋友圈的视觉效果 */
  bottom: -40rpx; 
  display: flex;
  align-items: flex-end; 
}
.profile-nickname {
  font-size: 36rpx;
  color: #FFFFFF;
  font-weight: bold;
  margin-right: 30rpx;
  /* 加点阴影防止背景图太白看不清字 */
  text-shadow: 2rpx 2rpx 4rpx rgba(0,0,0,0.6); 
  margin-bottom: 40rpx; /* 对齐头像的视觉高度 */
}
.profile-avatar {
  width: 130rpx;
  height: 130rpx;
  border-radius: 16rpx; /* 圆角矩形 */
  border: 4rpx solid #FFFFFF; /* 朋友圈专属白边 */
  box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.1);
  background-color: #EEE;
}

/* 个性签名与消息区 */
.profile-bottom {
  background-color: #FFFFFF;
  padding: 0 30rpx 30rpx 30rpx;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}
.signature {
  font-size: 26rpx;
  color: #666;
  flex: 1;
  margin-right: 20rpx;
  line-height: 1.5;
}
.notice-btn {
  font-size: 24rpx;
  background-color: #F5F5F5;
  color: #333;
  padding: 12rpx 24rpx;
  border-radius: 30rpx;
  position: relative;
  flex-shrink: 0;
}
.badge {
  position: absolute;
  top: -8rpx;
  right: -8rpx;
  background-color: #FF4D4F;
  color: #FFF;
  font-size: 20rpx;
  padding: 2rpx 10rpx;
  border-radius: 20rpx;
}

/* ================== 个人模块：分类 Tab ================== */
.personal-tabs {
  display: flex;
  background-color: #FFFFFF;
  border-bottom: 2rpx solid #F5F5F5;
  margin-top: 20rpx;
}
.p-tab {
  flex: 1;
  text-align: center;
  font-size: 28rpx;
  color: #666;
  padding: 24rpx 0;
  position: relative;
}
.p-tab.active {
  color: #333;
  font-weight: bold;
}
.p-tab.active::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 40rpx;
  height: 6rpx;
  background-color: #42b983;
  border-radius: 4rpx;
}

.empty-state {
  text-align: center;
  padding: 100rpx 0;
  color: #999;
  font-size: 28rpx;
}
</style>