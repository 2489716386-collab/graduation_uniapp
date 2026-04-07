<template>
  <view class="playground-container">
    <view class="title-area">
      <text class="main-title">My pets</text>
      <text class="sub-title">  完成今日计划，让陪伴更有意义</text>
    </view>

    <view class="pets-area" v-if="petList.length > 0">
      <view 
        v-for="(pet, index) in petList" 
        :key="pet.petid"
        class="pet-wrapper"
        :class="index % 2 === 0 ? 'align-left' : 'align-right'"
        :style="{ animationDelay: (index * 0.5) + 's' }"
      >
        <view class="interaction-node" :class="index % 2 === 0 ? 'node-left' : 'node-right'">
          
          <view class="pet-main-body" @click="goToBook(pet.petid)">
            <view class="cute-bubble">
              <text class="bubble-text">{{ pet.bubbleText || '正在思考喵生...' }}</text>
              <view class="bubble-tail" :class="index % 2 === 0 ? 'tail-left' : 'tail-right'"></view>
            </view>

            <view class="avatar-box">
              <image class="pet-avatar" :src="pet.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
              <text class="pet-name-tag">{{ pet.name }}</text>
            </view>
          </view>

          <view class="plan-box" v-if="pet.tasks && pet.tasks.length > 0">
            <view class="plan-header" @click="goToPlan">
              <text>📌 今日待办</text>
              <text class="header-arrow">❯</text>
            </view>
            
            <transition-group name="list-fade" tag="view" class="plan-list">
              <view 
                class="plan-item" 
                v-for="(task, tIndex) in pet.tasks.slice(0, 3)" 
                :key="task.id"
              >
                <text class="plan-name">{{ task.taskContent }}</text>
                <view class="check-btn" @click.stop="handleCheckIn(pet.petid, task.id, tIndex)">✓</view>
              </view>
            </transition-group>
            
            <view class="plan-more" v-if="pet.tasks.length > 3" @click="goToPlan">
              还有 {{ pet.tasks.length - 3 }} 项待办...
            </view>
          </view>

        </view>
      </view>
    </view>

    <view class="empty-state" v-else>
      <text class="empty-text">这里还没有小伙伴，快去添加吧~</text>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      petList: [],
      allBreeds: []
    };
  },
  async onShow() {
    const token = uni.getStorageSync('token');
    if (!token) {
      this.petList = [];
      return;
    }
    await this.loadBreeds(); 
    this.fetchPetsWithTasks();
  },
  methods: {
    loadBreeds() {
      return new Promise((resolve) => {
        uni.request({
          url: 'http://127.0.0.1:8080/pet-breeds/list',
          method: 'GET',
          header: { 'token': uni.getStorageSync('token') },
          success: (res) => {
            if (res.data.code === 200) this.allBreeds = res.data.data;
            resolve(); 
          },
          fail: () => resolve()
        });
      });
    },

    fetchPetsWithTasks() {
      const token = uni.getStorageSync('token');
      uni.request({
        url: 'http://127.0.0.1:8080/pets/user/list',
        method: 'GET',
        header: { 'token': token },
        success: (res) => {
          if (res.data.code === 200) {
            const rawPets = res.data.data;
            this.petList = rawPets.map(pet => {
              const currentPetId = pet.petid || pet.petId; 
              const breedObj = this.allBreeds.find(b => b.breedId === pet.breedId);
              return { 
                ...pet, 
                petid: currentPetId, 
                breedName: breedObj ? breedObj.breedName : '未知品种',
                tasks: [],
                bubbleText: '' 
              };
            });
            
            // 拿到宠物后即刻获取待办，并获取【历史日记】来生成气泡
            this.petList.forEach(pet => {
              this.fetchPetTasks(pet.petid);
              this.fetchHistoryAndGenerateBubble(pet); // 👈 新增的回忆杀方法
            });
          }
        }
      });
    },

    fetchPetTasks(petid) {
      if(!petid) return;
      uni.request({
        url: `http://127.0.0.1:8080/care-plans-day/today-plan/${petid}`,
        method: 'GET',
        header: { 'token': uni.getStorageSync('token') },
        success: (res) => {
          if (res.data.code === 200) {
            const petIndex = this.petList.findIndex(p => p.petid === petid);
            if (petIndex !== -1) {
              const unfinished = res.data.data.filter(t => t.isCompleted === 0);
              this.$set(this.petList[petIndex], 'tasks', unfinished);
            }
          }
        }
      });
    },

    handleCheckIn(petid, taskId, taskIndex) {
      uni.showLoading({ title: '打卡中...' });
      uni.request({
        url: `http://127.0.0.1:8080/care-plans-day/check-in/${taskId}`,
        method: 'POST',
        header: { 'token': uni.getStorageSync('token') },
        success: (res) => {
          uni.hideLoading();
          if (res.data.code === 200) {
            uni.showToast({ title: '打卡成功！', icon: 'success' });
            const pet = this.petList.find(p => p.petid === petid);
            if (pet) {
              pet.tasks.splice(taskIndex, 1);
            }
          }
        }
      });
    },

    // ================== 新增：拉取历史记录并生成气泡 ==================
    fetchHistoryAndGenerateBubble(pet) {
      uni.request({
        url: `http://127.0.0.1:8080/mood-records/history/${pet.petid}`,
        method: 'GET',
        header: { 'token': uni.getStorageSync('token') },
        success: (res) => {
          let history = [];
          if (res.data.code === 200 && res.data.data) {
            history = res.data.data;
          }
          this.generateBubbleText(pet, history);
        },
        fail: () => {
          // 如果接口挂了，依然保证能正常生成默认的聊天气泡
          this.generateBubbleText(pet, []);
        }
      });
    },

    // ================== 核心：带有“记忆”的气泡算法 ==================
    generateBubbleText(pet, history) {
      const today = new Date();
      const dateStr = `${today.getFullYear()}-${today.getMonth() + 1}-${today.getDate()}`;
      const currentHour = today.getHours();
      
      const logCacheKey = `diary_checked_${dateStr}_pet_${pet.petid}`;
      const hasCheckedLog = uni.getStorageSync(logCacheKey);

      let finalWords = "";

      // 【优先级 1：最高级】今天还没看日记
      if (!hasCheckedLog) {
        const diaryWords = [
          "主人，我的手账本更新啦，快来查收！",
          "今天还没看我的专属日记哦~",
          "猜猜我今天记录了什么小心情？戳我看看！"
        ];
        finalWords = diaryWords[pet.petid % diaryWords.length];
      } 
      // 【优先级 2：中级】根据时间提醒待办
      else if (currentHour >= 6 && currentHour < 10) {
        const morningWords = ["早上好呀！肚子饿饿，饭饭~", "新的一天也要活力满满！", "憋了一晚上了，快带我出去溜达！"];
        finalWords = morningWords[pet.petid % morningWords.length];
      } 
      else if (currentHour >= 11 && currentHour < 14) {
        const noonWords = ["午休时间到，和我一起打个盹吧zZZ", "中午吃什么好吃的？给我也来一口！"];
        finalWords = noonWords[pet.petid % noonWords.length];
      } 
      else if (currentHour >= 17 && currentHour < 20) {
        const eveningWords = ["天黑啦，晚饭时间到了！", "今天过得开心吗？快来摸摸我！", "吃饱喝足，想去散个步~"];
        finalWords = eveningWords[pet.petid % eveningWords.length];
      }
      else if (currentHour >= 22 || currentHour < 6) {
        const nightWords = ["嘘...本宝宝要睡觉了，晚安~", "好困哦，明天再陪你玩吧。"];
        finalWords = nightWords[pet.petid % nightWords.length];
      }
      // 【优先级 3：低级】发牢骚 或 触发“回忆杀”
      else {
        // 50%的概率，并且宠物确实有历史日记时，触发回忆
        if (history.length > 0 && Math.random() > 0.5) {
          // 随机抽取一条历史日记
          const randomLog = history[Math.floor(Math.random() * history.length)];
          if (randomLog && randomLog.diaryContent) {
            let content = randomLog.diaryContent;
            // 截断 AI 日记，防止气泡太大
            if (content.length > 25) content = content.substring(0, 25) + "...";
            finalWords = `我还记得那天：${content}`;
          }
        } 
        
        // 如果没有触发回忆，就走正常的随机撒娇文案
        if (!finalWords) {
          const affectionWords = [
            "好想你呀，要亲亲抱抱举高高！",
            "你是不是在外面有别的宠物了？哼！",
            "最爱主人啦，今天也要开开心心哦~",
            "不要总是看手机，看看可爱的我嘛！",
            "今天天气真不错，主人的心情好吗？"
          ];
          const randomIndex = (pet.petid + Math.floor(Math.random() * 10)) % affectionWords.length;
          finalWords = affectionWords[randomIndex];
        }
      }

      // 响应式更新到 Vue 视图
      this.$set(pet, 'bubbleText', finalWords);
    },

    goToBook(petid) {
      if(!petid) return;
      const today = new Date();
      const dateStr = `${today.getFullYear()}-${today.getMonth() + 1}-${today.getDate()}`;
      uni.setStorageSync(`diary_checked_${dateStr}_pet_${petid}`, true);
      uni.navigateTo({ url: `/pages/index/mood-log?petId=${petid}` });
    },

    goToPlan() {
      uni.switchTab({ url: '/pages/plan/plan' });
    }
  }
}
</script>

<style scoped lang="scss">
/* 样式依然原封不动，继续保持你的背景图和特效 */
.playground-container {
  background-image: url('https://mp-aa1e3790-0b44-4bfc-afa7-9b9be364f376.cdn.bspapp.com/cloudstorage/b6940cdd-529c-4f73-956e-bed852a4988b.webp');
  background-size: cover;
  background-position: center;
  background-attachment: fixed; 
  min-height: 100vh;
  padding: 40rpx;
  overflow-x: hidden;
}

.title-area {
  text-align: center;
  margin-top: 40rpx;
  margin-bottom: 60rpx;
  background: rgba(255, 255, 255, 0.4); 
  padding: 20rpx;
  border-radius: 40rpx;
}
.main-title { 
  font-size: 56rpx; 
  font-weight: 900; 
  color: #333; 
  font-family: 'Arial Rounded MT Bold', sans-serif;
  text-shadow: 2rpx 2rpx 4rpx rgba(255,255,255,0.8);
}
.sub-title { font-size: 24rpx; color: #666; margin-top: 8rpx; }

.pets-area { display: flex; flex-direction: column; gap: 80rpx; padding-bottom: 80rpx; }

.pet-wrapper {
  display: flex;
  width: 100%;
  animation: floating 4s ease-in-out infinite;
}

.align-left { justify-content: flex-start; }
.align-right { justify-content: flex-end; }

.interaction-node { display: flex; align-items: center; gap: 30rpx; }
.node-left { flex-direction: row; }
.node-right { flex-direction: row-reverse; } 

.plan-box {
  background: rgba(255, 255, 255, 0.9); 
  border-radius: 24rpx;
  padding: 20rpx;
  width: 340rpx; 
  border: 4rpx solid #FFD3B6;
  box-shadow: 0 10rpx 30rpx rgba(0,0,0,0.1);
}

.plan-header {
  font-size: 26rpx;
  color: #D2527F;
  font-weight: bold;
  display: flex;
  justify-content: space-between;
  margin-bottom: 16rpx;
  padding-bottom: 10rpx;
  border-bottom: 2rpx dashed #FFD3B6;
}

.plan-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12rpx 0;
}

.plan-name {
  font-size: 24rpx;
  color: #555;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
  margin-right: 15rpx;
}

.check-btn {
  width: 44rpx;
  height: 44rpx;
  border-radius: 50%;
  background-color: #A1D4C6;
  color: #fff;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 24rpx;
  box-shadow: 0 4rpx 8rpx rgba(161, 212, 198, 0.3);
}

.cute-bubble {
  background-color: #fff;
  padding: 15rpx 25rpx;
  border-radius: 30rpx;
  box-shadow: 0 8rpx 15rpx rgba(0,0,0,0.05);
  margin-bottom: 15rpx;
  border: 4rpx solid #FFD3B6;
  max-width: 300rpx; /* 限制一下最大宽度，防止字多撑满 */
}
.bubble-text { 
  font-size: 24rpx; 
  color: #666; 
  font-weight: bold; 
  line-height: 1.4; /* 稍微增加行高，多行字更好看 */
}

.avatar-box { display: flex; flex-direction: column; align-items: center; }
.pet-avatar {
  width: 160rpx; height: 160rpx;
  border-radius: 50%;
  border: 8rpx solid #fff;
  box-shadow: 0 10rpx 20rpx rgba(0,0,0,0.1);
}
.pet-name-tag {
  background-color: #4FA2FE;
  color: #fff; font-size: 22rpx;
  padding: 4rpx 18rpx;
  border-radius: 20rpx;
  margin-top: -15rpx;
  border: 4rpx solid #fff;
}

@keyframes floating {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20rpx); }
}

.empty-state { text-align: center; margin-top: 200rpx; color: #999; }
</style>