<template>
  <view class="container">
    <view class="info-card">
      <image class="main-avatar" :src="petInfo.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
      <view class="info-content">
        <view class="row"><text class="label">品种：</text><text>{{ petInfo.breed }}</text></view>
        <view class="row"><text class="label">年龄：</text><text>{{ petInfo.age }}岁</text></view>
        <view class="row"><text class="label">体重：</text><text>{{ petInfo.weight }}kg</text></view>
        <view class="row"><text class="label">当前状态：</text><text class="status-val">{{ petInfo.healthStatus }}</text></view>
      </view>
    </view>

    <view class="action-group">
      <button class="btn btn-history" @click="goToHistory">历史记录</button>
      <button class="btn btn-update" @click="goToGenerate">更新计划</button>
    </view>

    <view class="weekly-section">
      <view class="section-title">本周养护详细计划</view>
      
      <view class="day-block" v-for="(day, index) in weeklyPlan" :key="index">
        <view class="day-header">
          <text class="day-name">{{ day.name }}</text>
          <text class="day-date">{{ day.date }}</text>
        </view>
        <view class="day-content">
          <view class="task-line" v-for="(task, tIndex) in day.tasks" :key="tIndex">
            <text class="task-cate">[{{ task.category }}]</text>
            <text class="task-val">{{ task.content }}</text>
          </view>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      petInfo: { avatar: '', breed: '金毛', age: 3, weight: 25.5, healthStatus: '肝指标偏高，需静养' },
      // 模拟完整的周计划数据
      weeklyPlan: [
        { 
          name: '周一', date: '04-06', 
          tasks: [
            { category: '饮食', content: '早晚各100g保肝处方粮' },
            { category: '医疗', content: '早饭后喂食护肝片1粒' },
            { category: '运动', content: '仅限室内慢走' }
          ]
        },
        { 
          name: '周二', date: '04-07', 
          tasks: [
            { category: '饮食', content: '早晚各100g保肝处方粮' },
            { category: '清洁', content: '全身梳毛，检查皮肤状况' }
          ]
        },
        // ... 此处省略周三至周日
      ]
    };
  },
  methods: {
    goToHistory() { uni.navigateTo({ url: `/pages/plan/history` }); },
    goToGenerate() { uni.navigateTo({ url: `/pages/plan/generate` }); }
  }
}
</script>

<style lang="scss">
.container { padding: 15px; background: #f8f8f8; min-height: 100vh; }

.info-card { 
  background: #fff; padding: 20px; border-radius: 12px; margin-bottom: 15px; display: flex; align-items: center;
  .main-avatar { width: 80px; height: 80px; border-radius: 12px; margin-right: 20px; background: #eee; }
  .info-content { flex: 1; .row { margin-bottom: 5px; font-size: 13px; .label { color: #999; } .status-val { color: #f39c12; font-weight: bold; } } }
}

.action-group { display: flex; justify-content: space-between; margin-bottom: 20px;
  .btn { width: 48%; font-size: 14px; border-radius: 20px; height: 36px; line-height: 36px; }
  .btn-history { background: #fff; color: #333; border: 1px solid #ddd; }
  .btn-update { background: #007AFF; color: #fff; }
}

.weekly-section {
  background: #fff; padding: 15px; border-radius: 12px;
  .section-title { font-size: 16px; font-weight: bold; margin-bottom: 20px; padding-left: 10px; border-left: 4px solid #007AFF; }
  
  .day-block {
    margin-bottom: 20px;
    .day-header { display: flex; align-items: baseline; margin-bottom: 10px;
      .day-name { font-size: 15px; font-weight: bold; color: #333; margin-right: 10px; }
      .day-date { font-size: 12px; color: #999; }
    }
    .day-content {
      background: #fcfcfc; border-radius: 8px; padding: 10px;
      .task-line { font-size: 13px; margin-bottom: 6px; line-height: 1.6;
        .task-cate { color: #007AFF; margin-right: 6px; font-weight: bold; }
        .task-val { color: #555; }
      }
    }
  }
}
</style>