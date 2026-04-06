<template>
  <view class="container">
    <view class="header">请选择要查看计划的宠物</view>
    
    <view class="pet-list">
      <view class="pet-group" v-for="(pet, pIndex) in petList" :key="pet.id">
        <view class="pet-card" @click="goToDetail(pet.id)">
          <image class="avatar" :src="pet.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
          <view class="info">
            <text class="name">{{ pet.name }}</text>
            <text class="desc">{{ pet.breed }} · {{ pet.age }}岁</text>
          </view>
          <text class="arrow">❯</text>
        </view>

        <view class="todo-section">
          <view class="todo-header">
            <text class="todo-title">今日待办</text>
            <text class="todo-count">{{ getCompletedCount(pet.tasks) }}/{{ pet.tasks.length }}</text>
          </view>
          <view class="task-list">
            <view class="task-item" v-for="(task, tIndex) in pet.tasks" :key="tIndex">
              <view class="task-left">
                <text class="task-dot" :class="{ 'dot-done': task.completed }"></text>
                <text class="task-text" :class="{ 'text-done': task.completed }">{{ task.content }}</text>
              </view>
              <button 
                class="check-btn" 
                :class="{ 'btn-done': task.completed }"
                @click.stop="toggleTask(pIndex, tIndex)"
              >
                {{ task.completed ? '已打卡' : '打卡' }}
              </button>
            </view>
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
      petList: [
        { 
          id: 1, name: '旺财', breed: '金毛', age: 3, avatar: '',
          tasks: [
            { content: '早饭：100g 处方粮', completed: true },
            { content: '午后散步 20 分钟', completed: false },
            { content: '晚间梳毛护理', completed: false }
          ]
        },
        { 
          id: 2, name: '咪咪', breed: '布偶猫', age: 1, avatar: '',
          tasks: [
            { content: '清理猫砂盆', completed: true },
            { content: '喂食化毛膏', completed: true }
          ]
        }
      ]
    };
  },
  methods: {
    goToDetail(petId) {
      uni.navigateTo({ url: `/pages/plan/detail?petId=${petId}` });
    },
    toggleTask(pIndex, tIndex) {
      this.petList[pIndex].tasks[tIndex].completed = !this.petList[pIndex].tasks[tIndex].completed;
      if (this.petList[pIndex].tasks[tIndex].completed) {
        uni.showToast({ title: '打卡成功！', icon: 'none' });
      }
    },
    getCompletedCount(tasks) {
      return tasks.filter(t => t.completed).length;
    }
  }
}
</script>

<style lang="scss">
.container { padding: 20px; background-color: #f8f8f8; min-height: 100vh; box-sizing: border-box; }
.header { font-size: 14px; color: #666; margin-bottom: 15px; padding-left: 4px; }

.pet-group { background: #fff; border-radius: 12px; margin-bottom: 20px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.04); }

.pet-card {
  display: flex; align-items: center; padding: 15px; border-bottom: 1px solid #f0f0f0;
  .avatar { width: 45px; height: 45px; border-radius: 50%; margin-right: 12px; background: #eee; }
  .info { flex: 1; .name { font-size: 16px; font-weight: bold; color: #333; } .desc { font-size: 12px; color: #888; } }
  .arrow { color: #c0c0c0; font-size: 14px; }
}

.todo-section {
  padding: 12px 15px;
  .todo-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;
    .todo-title { font-size: 13px; font-weight: bold; color: #555; }
    .todo-count { font-size: 12px; color: #999; }
  }
  .task-item {
    display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;
    .task-left { display: flex; align-items: center; flex: 1;
      .task-dot { width: 6px; height: 6px; border-radius: 50%; background: #007AFF; margin-right: 10px; }
      .dot-done { background: #ccc; }
      .task-text { font-size: 14px; color: #444; }
      .text-done { color: #aaa; text-decoration: line-through; }
    }
    .check-btn { 
      font-size: 11px; padding: 0 12px; height: 24px; line-height: 24px; border-radius: 12px; 
      background: #007AFF; color: #fff; margin: 0;
      &::after { border: none; }
    }
    .btn-done { background: #f0f0f0; color: #999; }
  }
}
</style>