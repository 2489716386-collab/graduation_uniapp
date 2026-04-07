<template>
  <view class="desk-background">
    <view class="book-container">
      
      <view class="book-spine"></view>

      <view class="book-header">
        <text class="book-title">🐾 我的心情手账</text>
        <text class="book-subtitle">记录主人的陪伴时光</text>
      </view>

      <view class="camera-btn-area">
        <button class="snap-btn" @click="handleUploadMood" :loading="isGenerating">
          <text v-if="!isGenerating">📷 记录今日心情</text>
          <text v-else>AI 正在努力写日记中...</text>
        </button>
      </view>

      <scroll-view scroll-y class="diary-list">
        <view v-if="historyList.length > 0">
          <view class="diary-entry" v-for="log in historyList" :key="log.moodId">
            <view class="date-ribbon">
              <text>{{ formatDate(log.recordDate) }}</text>
            </view>
            
            <view class="polaroid">
              <image class="mood-img" :src="log.imageUrl" mode="aspectFill" @click="previewImg(log.imageUrl)"></image>
              <view class="mood-tag" :class="getMoodColor(log.moodType)">
                心情: {{ translateMood(log.moodType) }} (AI确信度: {{ (log.confidence * 100).toFixed(0) }}%)
              </view>
            </view>

            <view class="diary-content">
              <text class="text-content">“{{ log.diaryContent || '今天也是充满希望的一天呀～' }}”</text>
            </view>
          </view>
        </view>

        <view class="empty-page" v-else>
          <image class="empty-book-img" src="/static/default-avatar.png" mode="aspectFill"></image>
          <text class="empty-text">日记本还是空白的呢\n快拍张照让我记录今天的心情吧！</text>
        </view>
      </scroll-view>

    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      petId: null,
      historyList: [],
      isGenerating: false // 控制生成时的 Loading 状态
    };
  },
  onLoad(options) {
    if (options.petId) {
      this.petId = options.petId;
    }
  },
  onShow() {
    if (this.petId) {
      this.fetchHistory();
    }
  },
  methods: {
    // 1. 获取历史日记列表
    fetchHistory() {
      const token = uni.getStorageSync('token');
      uni.request({
        url: `http://127.0.0.1:8080/mood-records/history/${this.petId}`,
        method: 'GET',
        header: { 'token': token },
        success: (res) => {
          if (res.data.code === 200) {
            this.historyList = res.data.data;
          }
        }
      });
    },

    // 2. 核心功能：拍照 -> 上传云端 -> 调用 Java 生成接口
    handleUploadMood() {
      uni.chooseImage({
        count: 1,
        sourceType: ['camera', 'album'],
        success: async (chooseRes) => {
          const tempFilePath = chooseRes.tempFilePaths[0];
          this.isGenerating = true;

          try {
            // 第一步：上传到 uniCloud 获取云端 URL
            uni.showLoading({ title: '图片上传中...' });
            let uploadRes = await uniCloud.uploadFile({
              filePath: tempFilePath,
              cloudPath: 'mood_' + Date.now() + '.jpg'
            });
            const imageUrl = uploadRes.fileID;
            uni.hideLoading();

            // 第二步：将云端 URL 发给 Java 后端，让后端调度 Python 和 DeepSeek
            uni.showLoading({ title: 'AI检测并写日记中...', mask: true });
            
            const userInfo = uni.getStorageSync('userInfo') || { userId: 1 }; // 这里用你真实的获取用户ID逻辑
            const token = uni.getStorageSync('token');

            uni.request({
              url: 'http://127.0.0.1:8080/mood-records/generate',
              method: 'POST',
              header: { 
                'token': token,
                'content-type': 'application/x-www-form-urlencoded' 
              },
              data: {
                userId: userInfo.userId || 1, 
                petId: this.petId,
                imageUrl: imageUrl
              },
              success: (res) => {
                uni.hideLoading();
                this.isGenerating = false;

                if (res.data.code === 200) {
                  uni.showToast({ title: '日记生成成功！', icon: 'success' });
                  // 生成成功后重新拉取列表，新日记就会出现在页面最上方
                  this.fetchHistory();
                } else {
                  // 这里会捕捉到 Python 发现“非宠物照片”等异常提示
                  uni.showModal({ title: '提示', content: res.data.msg || '生成失败', showCancel: false });
                }
              },
              fail: (err) => {
                uni.hideLoading();
                this.isGenerating = false;
                uni.showToast({ title: '网络异常', icon: 'none' });
              }
            });

          } catch (e) {
            uni.hideLoading();
            this.isGenerating = false;
            uni.showToast({ title: '上传图片失败', icon: 'none' });
            console.error(e);
          }
        }
      });
    },

    // 图片预览
    previewImg(url) {
      uni.previewImage({ urls: [url] });
    },

    // 格式化日期：比如把 2026-04-07 转成 4月7日
    formatDate(dateStr) {
      if (!dateStr) return '';
      const date = new Date(dateStr);
      return `${date.getMonth() + 1}月${date.getDate()}日`;
    },

    // 翻译英文心情到中文萌趣词
    translateMood(mood) {
      const dict = {
        'happy': '开心到飞起 (*^▽^*)',
        'sad': '有点小忧郁 (T_T)',
        'angry': '超凶的！(｀・ω・´)',
        'normal': '情绪稳定 (￣▽￣)'
      };
      // 注意这里要把字母转成小写匹配，防止 Python 返回大写
      return dict[mood?.toLowerCase()] || mood || '神秘情绪';
    },

    // 根据不同心情返回不同的 CSS 类名（改变标签颜色）
    getMoodColor(mood) {
      const m = mood?.toLowerCase();
      if (m === 'happy') return 'tag-happy';
      if (m === 'sad') return 'tag-sad';
      if (m === 'angry') return 'tag-angry';
      return 'tag-normal';
    }
  }
}
</script>

<style scoped>
/* 整个背景是一张木质桌面或温馨背景 */
.desk-background {
  background-color: #D2B48C; /* 暖色调模拟原木桌面 */
  min-height: 100vh;
  padding: 40rpx;
  display: flex;
  justify-content: center;
}

/* 核心魔法：书本 3D 效果 */
.book-container {
  background-color: #FFFDF8; /* 旧纸张颜色 */
  width: 100%;
  border-radius: 10rpx 40rpx 40rpx 10rpx; /* 右侧圆角模拟书页边缘 */
  box-shadow: 
    20rpx 20rpx 30rpx rgba(0,0,0,0.15), /* 书本外投影 */
    inset 15rpx 0 20rpx rgba(0,0,0,0.05); /* 左侧装订线内阴影 */
  position: relative;
  padding: 40rpx 30rpx 40rpx 60rpx; /* 左侧留出装订线空间 */
  display: flex;
  flex-direction: column;
}

/* 模拟左侧的书脊线 */
.book-spine {
  position: absolute;
  left: 30rpx;
  top: 0;
  bottom: 0;
  width: 4rpx;
  background: linear-gradient(to right, rgba(0,0,0,0.05), transparent);
  border-left: 2rpx dashed #d1c1a3;
}

/* 头部样式 */
.book-header {
  text-align: center;
  margin-bottom: 40rpx;
  border-bottom: 2rpx solid #EEDBB5;
  padding-bottom: 20rpx;
}
.book-title {
  font-size: 38rpx;
  font-weight: bold;
  color: #5C4A3D;
  display: block;
}
.book-subtitle {
  font-size: 24rpx;
  color: #A89B8C;
}

/* 拍照按钮 */
.camera-btn-area {
  margin-bottom: 30rpx;
}
.snap-btn {
  background-color: #FFB3A7; /* 可爱的粉色 */
  color: #fff;
  border-radius: 40rpx;
  font-size: 30rpx;
  font-weight: bold;
  box-shadow: 0 8rpx 16rpx rgba(255, 179, 167, 0.4);
}
.snap-btn::after {
  border: none;
}

/* 日记列表页区 */
.diary-list {
  flex: 1;
  height: 60vh; /* 限制高度，让内部可以滚动 */
}

/* 单条日记：拍立得风格 */
.diary-entry {
  margin-bottom: 50rpx;
  position: relative;
}
.date-ribbon {
  background-color: #A1D4C6;
  color: #fff;
  display: inline-block;
  padding: 6rpx 20rpx;
  border-radius: 10rpx 10rpx 10rpx 0;
  font-size: 24rpx;
  font-weight: bold;
  position: relative;
  left: -20rpx;
  margin-bottom: 10rpx;
  box-shadow: 2rpx 4rpx 8rpx rgba(0,0,0,0.1);
}
.date-ribbon::after {
  content: '';
  position: absolute;
  bottom: -10rpx;
  left: 0;
  border-top: 10rpx solid #7BB8A8;
  border-left: 10rpx solid transparent;
}

/* 拍立得照片容器 */
.polaroid {
  background: #fff;
  padding: 16rpx 16rpx 24rpx 16rpx;
  border-radius: 4rpx;
  box-shadow: 0 4rpx 16rpx rgba(0,0,0,0.1);
  transform: rotate(-2deg); /* 微微倾斜显得随意 */
  margin-bottom: 20rpx;
}
.mood-img {
  width: 100%;
  height: 350rpx;
  background-color: #f5f5f5;
  border-radius: 4rpx;
}

/* 情绪标签 */
.mood-tag {
  font-size: 24rpx;
  margin-top: 16rpx;
  text-align: center;
  font-weight: bold;
}
.tag-happy { color: #FF7B89; }
.tag-sad { color: #5D9CEC; }
.tag-angry { color: #E85757; }
.tag-normal { color: #48C9B0; }

/* 日记文本 */
.diary-content {
  background: repeating-linear-gradient(
    to bottom,
    transparent,
    transparent 48rpx,
    #EEDBB5 48rpx,
    #EEDBB5 50rpx
  );
  line-height: 50rpx;
  padding-top: 6rpx;
  margin-top: 10rpx;
  transform: rotate(1deg);
}
.text-content {
  font-size: 30rpx;
  color: #4A4A4A;
  font-family: "楷体", "STKaiti", serif; /* 尝试使用手写体/楷体 */
}

/* 空白状态 */
.empty-page {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 100rpx;
}
.empty-book-img {
  width: 200rpx;
  height: 200rpx;
  opacity: 0.3;
}
.empty-text {
  color: #A89B8C;
  font-size: 28rpx;
  margin-top: 20rpx;
  text-align: center;
  line-height: 1.6;
}
</style>