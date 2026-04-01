<template>
  <view class="container">
    <view class="custom-header">
      <view class="status-bar-placeholder"></view>
      <view class="nav-tabs">
        <text class="tab-item" :class="{ 'active': currentMainTab === 'community' }" @click="switchMainTab('community')">动态</text>
        <text class="tab-item" :class="{ 'active': currentMainTab === 'personal' }" @click="switchMainTab('personal')">个人</text>
      </view>
    </view>

    <view class="main-content">
      <community-feed v-show="currentMainTab === 'community'" ref="feedRef" />
      <personal-center v-show="currentMainTab === 'personal'" ref="personalRef" />
    </view>
  </view>
</template>

<script>
import CommunityFeed from './community-feed.vue';
import PersonalCenter from './personal-center.vue';

export default {
  components: {
    CommunityFeed,
    PersonalCenter
  },
  data() {
    return {
      currentMainTab: 'community' // 默认显示社区
    };
  },
  onShow() {
    // 页面显示时，通知当前激活的子组件刷新数据
    this.$nextTick(() => {
      if (this.currentMainTab === 'community' && this.$refs.feedRef) {
        this.$refs.feedRef.refresh();
      } else if (this.currentMainTab === 'personal' && this.$refs.personalRef) {
        this.$refs.personalRef.refresh();
      }
    });
  },
  // 页面原生触底事件
  onReachBottom() {
    // 判断当前在哪个 tab，就把触底事件转发给对应的组件
    if (this.currentMainTab === 'community') {
      this.$refs.feedRef.loadMore();
    } else {
      this.$refs.personalRef.loadMore();
    }
  },
  methods: {
    switchMainTab(tab) {
      this.currentMainTab = tab;
      uni.pageScrollTo({ scrollTop: 0, duration: 0 }); // 切换时回到顶部
      
      // 切换 tab 时，也顺便刷新一下该 tab 的数据
      this.$nextTick(() => {
        if (tab === 'community') {
          this.$refs.feedRef.refresh();
        } else {
          // 如果用户没登录，personal-center 组件内部会自动处理拦截
          this.$refs.personalRef.refresh();
        }
      });
    }
  }
}
</script>

<style scoped>
page { background-color: #F8F8F8; }
.container { width: 100%; }

/* 固定顶部导航样式 */
.custom-header { position: fixed; top: 0; left: 0; width: 100%; background-color: #FFFFFF; z-index: 999; box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05); }
.status-bar-placeholder { height: var(--status-bar-height); width: 100%; }
.nav-tabs { height: 88rpx; display: flex; justify-content: center; align-items: center; gap: 80rpx; }
.tab-item { font-size: 32rpx; color: #999; font-weight: 500; position: relative; height: 88rpx; line-height: 88rpx; }
.tab-item.active { color: #333; font-weight: bold; font-size: 34rpx; }
.tab-item.active::after { content: ''; position: absolute; bottom: 10rpx; left: 50%; transform: translateX(-50%); width: 40rpx; height: 6rpx; background-color: #42b983; border-radius: 4rpx; }

/* 内容区包裹 */
.main-content { padding-top: calc(var(--status-bar-height) + 88rpx); width: 100%; }
</style>