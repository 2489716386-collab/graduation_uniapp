<template>
  <view class="container">
    <view v-if="historyList.length === 0" class="empty-state">
      <text class="icon">📭</text>
      <text class="text">暂无历史养护计划记录</text>
    </view>

    <view class="history-list" v-else>
      <view class="history-card" v-for="(plan, index) in historyList" :key="plan.id">
        
        <view class="card-header">
          <view class="date-range">
            <text class="icon">📅</text>
            <text>{{ plan.startDate }} ~ {{ plan.endDate }}</text>
          </view>
          <text class="status-tag">已归档</text>
        </view>

        <view class="card-body">
          <view class="snapshot-row">
            <text class="label">当时记录：</text>
            <text class="value">{{ plan.snapshotAge }}岁 · {{ plan.snapshotWeight }}kg</text>
          </view>
          <view class="snapshot-row">
            <text class="label">健康状况：</text>
            <text class="value truncate">{{ plan.snapshotHealthStatus || '无异常' }}</text>
          </view>
          <view class="focus-box">
            <text class="focus-title">💡 当周重点：</text>
            <text class="focus-content">{{ plan.weeklyFocus }}</text>
          </view>
        </view>

        <view class="card-footer" @click="viewHistoryDetail(plan.id)">
          <text class="footer-text">查看该周详细任务</text>
          <text class="arrow">></text>
        </view>
        
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      petId: null,
      historyList: []
    };
  },
  onLoad(options) {
	  const id = options.petid || options.petId;
	      if (id && id !== 'undefined' && id !== 'null') {
	          this.petId = id;
	          this.fetchHistoryData();
	      }
  },
  methods: {
    fetchHistoryData() {
      uni.showLoading({ title: '加载中...' });
      // TODO: 调用后端接口 GET /api/care-plans/history?petId=xxx
      // 这里用 setTimeout 模拟接口返回的数据
      setTimeout(() => {
        uni.hideLoading();
        this.historyList = [
          {
            id: 102,
            startDate: '2026-03-23',
            endDate: '2026-03-29',
            snapshotAge: 3,
            snapshotWeight: 24.5,
            snapshotHealthStatus: '轻微拉肚子，已喂食益生菌',
            weeklyFocus: '肠胃恢复期，少食多餐，暂停户外剧烈运动。'
          },
          {
            id: 101,
            startDate: '2026-03-16',
            endDate: '2026-03-22',
            snapshotAge: 3,
            snapshotWeight: 25.0,
            snapshotHealthStatus: '健康',
            weeklyFocus: '春季换毛期，加强每日梳毛频率，补充鱼油。'
          }
        ];
      }, 800);
    },
    
    viewHistoryDetail(planId) {
      // 可选：跳转到一个只读的详情页，展示那一周具体的 daily_tasks
      // uni.navigateTo({ url: `/pages/plan/history-detail?planId=${planId}` });
      uni.showToast({ title: '准备查看计划ID: ' + planId, icon: 'none' });
    }
  }
}
</script>

<style lang="scss">
.container { padding: 15px; background: #f5f7fa; min-height: 100vh; }

/* 空状态样式 */
.empty-state { display: flex; flex-direction: column; align-items: center; justify-content: center; padding-top: 100px; 
  .icon { font-size: 50px; margin-bottom: 15px; }
  .text { color: #999; font-size: 15px; }
}

/* 卡片样式 */
.history-card { background: #fff; border-radius: 12px; padding: 15px; margin-bottom: 15px; box-shadow: 0 2px 10px rgba(0,0,0,0.03); }

.card-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #f0f0f0; padding-bottom: 12px; margin-bottom: 12px; 
  .date-range { font-size: 15px; font-weight: bold; color: #333; display: flex; align-items: center; 
    .icon { margin-right: 6px; font-size: 14px; }
  }
  .status-tag { font-size: 12px; color: #8c8c8c; background: #f5f5f5; padding: 3px 8px; border-radius: 4px; }
}

.card-body { margin-bottom: 15px; 
  .snapshot-row { margin-bottom: 6px; font-size: 13px; display: flex; 
    .label { color: #888; width: 70px; flex-shrink: 0; }
    .value { color: #333; flex: 1; }
    .truncate { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  }
  .focus-box { margin-top: 12px; background: #fff8e6; padding: 10px; border-radius: 6px; 
    .focus-title { font-size: 13px; font-weight: bold; color: #d48806; margin-bottom: 4px; display: block; }
    .focus-content { font-size: 13px; color: #666; line-height: 1.5; }
  }
}

.card-footer { display: flex; justify-content: space-between; align-items: center; border-top: 1px dashed #eee; padding-top: 12px; 
  .footer-text { font-size: 13px; color: #007AFF; }
  .arrow { color: #ccc; font-size: 16px; }
}
</style>