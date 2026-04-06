<template>
  <view class="container">
    
    <view v-if="step === 1" class="step-form">
      <view class="title">核对宠物信息</view>
      <view class="form-group">
        <view class="label">品种 (可修改)</view>
        <input class="input" v-model="formData.breed" placeholder="例如: 金毛" />
      </view>
      <view class="form-group">
        <view class="label">年龄 (岁)</view>
        <input class="input" type="number" v-model="formData.age" />
      </view>
      <view class="form-group">
        <view class="label">体重 (kg)</view>
        <input class="input" type="digit" v-model="formData.weight" />
      </view>
      <view class="form-group">
        <view class="label">健康状况描述</view>
        <textarea class="textarea" v-model="formData.healthStatus" placeholder="请描述宠物目前的健康状况，例如：肠胃不好、有泪痕等"></textarea>
      </view>
      
      <view class="ocr-upload" @click="uploadReport">
        <text class="icon">📄</text>
        <text>上传体检报告自动提取异常指标 (推荐)</text>
      </view>

      <button class="submit-btn" @click="startGenerate">提交生成计划</button>
    </view>

    <view v-if="step === 2" class="step-loading">
      <view class="loading-spinner"></view>
      <view class="loading-text">🐾 AI 正在化身专业兽医...</view>
      <view class="loading-subtext">正在为您定制专属养护计划，预计需要 5-8 秒</view>
    </view>

    <view v-if="step === 3" class="step-preview">
      <view class="title">计划预览</view>
      
      <scroll-view scroll-y class="preview-box">
        <view class="focus-alert">💡 本周重点：{{ generatedPlan.weekly_focus }}</view>
        <view v-for="(task, index) in generatedPlan.daily_tasks" :key="index" class="preview-task">
          <text class="tag">{{ task.category }}</text>
          <text class="desc">{{ task.content }}</text>
        </view>
      </scroll-view>

      <view class="feedback-area">
        <text class="label">不满意？告诉 AI 您的修改意见：</text>
        <view class="feedback-input-box">
          <input class="input" v-model="feedbackText" placeholder="例如：我家狗对鸡肉过敏" />
          <button class="btn-regenerate" @click="regeneratePlan">重新生成</button>
        </view>
      </view>

      <button class="submit-btn" @click="confirmImport">确定导入本周计划</button>
    </view>

  </view>
</template>

<script>
export default {
  data() {
    return {
      step: 1, // 控制当前显示的步骤：1表单，2加载，3预览
      petId: null,
      formData: {
        breed: '', age: '', weight: '', healthStatus: ''
      },
      feedbackText: '', // 用户的重生成要求
      generatedPlan: {} // 存放 AI 吐出的 JSON 结果
    };
  },
  onLoad(options) {
    this.petId = options.petId;
    // TODO: 调用后端接口，获取该宠物当前资料，填充到 formData 中
    this.formData = { breed: '金毛', age: 3, weight: 25, healthStatus: '' };
  },
  methods: {
    // 触发 OCR
    uploadReport() {
      uni.chooseImage({
        count: 1,
        success: (res) => {
          const tempFilePath = res.tempFilePaths[0];
          uni.showLoading({ title: '报告解析中...' });
          // TODO: 将图片上传到 uniCloud 或后端调用百度/阿里 OCR 接口
          setTimeout(() => {
            uni.hideLoading();
            // 模拟 OCR 提取成功，自动填入健康状况
            this.formData.healthStatus += " [体检异常指标]：白细胞偏高，肝功能轻微异常。";
            uni.showToast({ title: '提取成功', icon: 'success' });
          }, 1500);
        }
      });
    },

    // 提交生成
    startGenerate() {
      if (!this.formData.breed) return uni.showToast({ title: '请填写品种', icon: 'none' });
      
      this.step = 2; // 切换到 Loading 页面
      
      // TODO: 调用后端 AI 生成接口 (Spring Boot 调用大模型)
      // 模拟接口请求等待 3 秒
      setTimeout(() => {
        // 模拟 AI 返回的 JSON 数据
        this.generatedPlan = {
          weekly_focus: "关注肝脏健康，本周需清淡饮食并增加饮水。",
          daily_tasks: [
            { category: "饮食", content: "早晚各喂食定制处方粮，禁止喂食含鸡肉零食。" },
            { category: "运动", content: "散步时间控制在20分钟内，避免剧烈奔跑。" },
            { category: "医疗", content: "喂食保肝药物一次。" }
          ]
        };
        this.step = 3; // 切换到预览页面
      }, 3000);
    },

    // 带反馈的重新生成
    regeneratePlan() {
      if (!this.feedbackText) return uni.showToast({ title: '请输入修改意见', icon: 'none' });
      
      this.step = 2; // 再次切换回 Loading 页面
      
      // TODO: 携带 this.feedbackText 再次调用 AI 接口
      setTimeout(() => {
        // 模拟根据反馈修改后的数据
        this.generatedPlan.daily_tasks[0].content = "已将零食替换为鸭肉冻干。早晚各喂食处方粮。";
        this.feedbackText = ''; // 清空输入框
        this.step = 3; // 回到预览
      }, 3000);
    },

    // 最终确认导入
    confirmImport() {
      uni.showLoading({ title: '保存中...' });
      // TODO: 调用后端接口，将 formData（更新宠物信息）和 generatedPlan（保存为周计划和每日任务）存入数据库
      setTimeout(() => {
        uni.hideLoading();
        uni.showToast({ title: '计划已更新！', icon: 'success' });
        // 保存成功，返回详情页
        setTimeout(() => {
          uni.navigateBack();
        }, 1500);
      }, 1000);
    }
  }
}
</script>

<style lang="scss">
.container { padding: 20px; background: #fff; min-height: 100vh; }
.title { font-size: 20px; font-weight: bold; margin-bottom: 20px; color: #333; }

/* 表单样式 */
.form-group { margin-bottom: 15px; 
  .label { font-size: 14px; color: #555; margin-bottom: 8px; }
  .input { border: 1px solid #ddd; padding: 10px; border-radius: 8px; background: #f9f9f9; }
  .textarea { width: 100%; border: 1px solid #ddd; padding: 10px; border-radius: 8px; background: #f9f9f9; height: 80px; box-sizing: border-box; }
}
.ocr-upload { background: #eef6ff; color: #007AFF; padding: 15px; border-radius: 8px; text-align: center; margin-bottom: 25px; border: 1px dashed #007AFF; }
.submit-btn { background: #007AFF; color: white; border-radius: 25px; font-size: 16px; margin-top: 20px; }

/* Loading 骨架屏样式 */
.step-loading { display: flex; flex-direction: column; align-items: center; justify-content: center; height: 60vh; 
  .loading-spinner { width: 40px; height: 40px; border: 4px solid #f3f3f3; border-top: 4px solid #007AFF; border-radius: 50%; animation: spin 1s linear infinite; margin-bottom: 20px; }
  .loading-text { font-size: 18px; font-weight: bold; color: #333; margin-bottom: 10px; }
  .loading-subtext { font-size: 13px; color: #999; }
}
@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

/* 预览页样式 */
.preview-box { background: #f8f8f8; padding: 15px; border-radius: 10px; max-height: 350px; margin-bottom: 20px; border: 1px solid #eee; }
.focus-alert { background: #fff3cd; color: #856404; padding: 10px; border-radius: 5px; margin-bottom: 15px; font-size: 14px; }
.preview-task { display: flex; margin-bottom: 10px; background: #fff; padding: 10px; border-radius: 6px; 
  .tag { background: #e0eaff; color: #007AFF; padding: 2px 6px; border-radius: 4px; font-size: 12px; margin-right: 10px; height: fit-content; white-space: nowrap; }
  .desc { font-size: 14px; color: #333; line-height: 1.5; }
}
.feedback-area { margin-bottom: 20px; 
  .label { font-size: 14px; color: #666; margin-bottom: 10px; display: block; }
  .feedback-input-box { display: flex; align-items: center; 
    .input { flex: 1; border: 1px solid #ddd; padding: 8px 12px; border-radius: 20px; font-size: 14px; margin-right: 10px; }
    .btn-regenerate { font-size: 13px; background: #ff9500; color: white; border-radius: 20px; padding: 0 15px; height: 36px; line-height: 36px; margin: 0; }
  }
}
</style>