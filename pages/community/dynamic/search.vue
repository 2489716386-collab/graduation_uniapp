<template>
  <view class="search-page">
    <view class="search-bar">
      <input v-model="keyword" placeholder="搜索动态..." @confirm="onSearch" />
      <text @click="onSearch">搜索</text>
    </view>

    <view class="sort-tabs">
      <text :class="{active: sortType === 'hot'}" @click="switchSort('hot')">热度</text>
      <text :class="{active: sortType === 'time'}" @click="switchSort('time')">时间</text>
    </view>

    <view class="result-list">
      <view v-for="post in results" :key="post.postId" class="post-card">
        </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      keyword: '',
      sortType: 'hot', // 默认热度排序
      results: []
    }
  },
  methods: {
    onSearch() {
      if (!this.keyword.trim()) return;
      this.fetchResults();
    },
    switchSort(type) {
      this.sortType = type;
      this.fetchResults();
    },
    fetchResults() {
      uni.request({
        url: 'http://localhost:8080/community-posts/search',
        method: 'GET',
        data: {
          keyword: this.keyword,
          sort: this.sortType, // 'hot' 或 'time'
          token: uni.getStorageSync('token')
        },
        success: (res) => {
          if (res.data.code === 200) {
            this.results = res.data.data;
          }
        }
      });
    }
  }
}
</script>