<template>
  <MyCard class="skin-challenge" :statusClass="statusClass">
    <template #icon>
      <img src="/icons/1733492491706152.png" alt="换皮闯关" />
    </template>
    <template #title>
      <h3>换皮闯关</h3>
      <p>挑战关卡赢取奖励</p>
    </template>
    <template #badge>
      <!-- Badge content moved to default slot -->
    </template>
    <template #default>
      <div v-if="towerActivities.length === 0" class="no-activity">
        暂无闯关活动
      </div>

      <div v-for="(activity, idx) in towerActivities" :key="activity.actId" class="tower-section">
        <div class="header-info">
          <span class="challenge-count">活动 {{ idx + 1 }} (ID: {{ activity.actId }})</span>
          <span class="challenge-count">今日挑战 {{ activity.dailyFightNum }}/10</span>
          <span class="daily-target" v-if="activity.isValid">活动中</span>
          <span class="daily-target" v-else>活动已结束</span>
        </div>

        <div v-if="!activity.isValid" class="expired-mask">
          当前活动已结束
        </div>
        <div class="boss-grid" :class="{ 'disabled': !activity.isValid }">
          <div
            v-for="type in 6"
            :key="type"
            class="boss-card"
            :class="{
              'active': isTowerOpen(activity, type),
              'cleared': isTowerCleared(activity, type),
              'locked': !isTowerOpen(activity, type)
            }"
          >
            <div class="boss-title">BOSS {{ type }}</div>
            <div class="boss-level">第 {{ getTowerLevel(activity, type) }} 层</div>

            <div class="boss-status">
              <span v-if="isTowerCleared(activity, type)" class="status-text cleared">已通关</span>
              <span v-else-if="!isTowerOpen(activity, type)" class="status-text locked">未开放</span>
              <span v-else class="status-text active">进行中</span>
            </div>

            <button
              class="challenge-btn"
              :disabled="!canChallenge(activity, type) || isFighting"
              @click="challengeSingle(activity, type)"
            >
              挑战
            </button>
          </div>
        </div>
      </div>

      <div class="action-row">
        <button
          class="action-button secondary"
          :disabled="isFighting"
          @click="refreshInfo"
        >
          {{ isFighting ? "刷新中..." : "刷新进度" }}
        </button>
      </div>
    </template>
  </MyCard>
</template>

<script setup>
import { computed, ref, watch } from "vue";
import { useTokenStore } from "@/stores/tokenStore";
import { useMessage } from "naive-ui";
import MyCard from "../Common/MyCard.vue";

const tokenStore = useTokenStore();
const message = useMessage();

const isFighting = ref(false);

// 多个闯关活动
const towerActivities = ref([]);
// 保留当前正在挑战的活动引用
const currentActivity = ref(null);

// 用 startTime 时间戳判断活动是否有效（7天窗口）
const checkActivityValid = (startTime) => {
  if (!startTime) return false;
  const endDate = startTime + 7 * 24 * 60 * 60 * 1000;
  const now = Date.now();
  return now >= startTime && now < endDate;
};

const statusClass = computed(() => {
  return towerActivities.value.length > 0 ? "active" : "completed";
});

// 用服务器返回的 fighting 字段判断 BOSS 是否可挑战
const isTowerOpen = (activity, type) => {
  return activity.towerDetail?.[String(type)]?.fighting === true;
};

const isTowerCleared = (activity, type) => {
  const map = activity.levelRewardMap || {};
  const key1 = `${type}008`;
  const key2 = Number(key1);
  return !!(map[key1] || map[key2]);
};

const getTowerLevel = (activity, type) => {
  const map = activity.levelRewardMap || {};
  for (let i = 8; i >= 1; i--) {
    const key1 = `${type}00${i}`;
    const key2 = Number(key1);
    if (map[key1] || map[key2]) {
        if (i === 8) return 8;
        return i + 1;
    }
  }
  return 1;
};

const canChallenge = (activity, type) => {
  return activity.isValid && isTowerOpen(activity, type) && !isTowerCleared(activity, type);
};

const getInfo = async () => {
  if (!tokenStore.selectedToken) return;
  const tokenId = tokenStore.selectedToken.id;
  if (tokenStore.getWebSocketStatus(tokenId) !== "connected") return;

  try {
    const actIdList = tokenStore.getMultiTowerActIds();
    if (actIdList.length === 0) {
      towerActivities.value = [];
      return;
    }

    const activities = [];
    for (const { actId, startTime } of actIdList) {
      try {
        const res = await tokenStore.sendMessageWithPromise(tokenId, "towers_getinfo", { actId }, 5000);
        if (res) {
          const data = res.towerData || res;
          activities.push({
            actId: data.actId || actId,
            startTime,
            towerDetail: data.towerData || {},
            levelRewardMap: data.levelRewardMap || {},
            dailyFightNum: data.todayUseTickCnt || 0,
            isValid: checkActivityValid(startTime),
          });
        }
      } catch (e) {
        console.warn(`获取闯关活动 ${actId} 信息失败:`, e.message);
      }
    }

    towerActivities.value = activities;
    currentActivity.value = activities.find(a => a.isValid) || activities[0] || null;

    console.log('SkinChallenge Info:', activities);
  } catch (e) {
    console.error(e);
  }
};

const refreshInfo = async () => {
  isFighting.value = true;
  await getInfo();
  message.success("进度已刷新");
  isFighting.value = false;
};

const challengeSingle = async (activity, type) => {
  if (isFighting.value) return;

  isFighting.value = true;
  currentActivity.value = activity;
  const tokenId = tokenStore.selectedToken.id;
  const actId = activity.actId;

  try {
     message.info(`开始挑战 BOSS ${type} (活动 ${actId})`);

     let needStart = true;
     let loop = true;
     let failCount = 0;

     while (loop) {
        if (needStart) {
            await tokenStore.sendMessageWithPromise(tokenId, "towers_start", { actId, towerType: type }, 5000);
        }

        const fightRes = await tokenStore.sendMessageWithPromise(tokenId, "towers_fight", { actId, towerType: type }, 5000);
        const battleData = fightRes?.battleData;
        const curHP = battleData?.result?.accept?.ext?.curHP;

        if (curHP === 0) {
            const currentLevel = getTowerLevel(activity, type);
            message.success(`BOSS ${type} 第 ${currentLevel} 层挑战成功`);

            needStart = false;
            failCount = 0;

            await getInfo();
            // 重新获取当前活动数据
            const updated = towerActivities.value.find(a => a.actId === actId);
            if (updated && isTowerCleared(updated, type)) {
                loop = false;
                message.success(`BOSS ${type} 已全部通关`);
            } else {
                await new Promise(r => setTimeout(r, 1000));
            }
        } else {
            const currentLevel = getTowerLevel(activity, type);
            message.warning(`BOSS ${type} 第 ${currentLevel} 层挑战失败`);
            needStart = true;
            failCount++;

            if (failCount >= 3) {
                message.error(`BOSS ${type} 第 ${currentLevel} 层连续失败 3 次，停止挑战`);
                loop = false;
            } else {
                await new Promise(r => setTimeout(r, 1000));
            }
        }
     }
  } catch (e) {
     message.error(`挑战出错: ${e.message}`);
  } finally {
     isFighting.value = false;
     await getInfo();
  }
};

watch(
  () => tokenStore.selectedToken,
  (newVal) => {
    if (newVal) {
       setTimeout(getInfo, 1000);
    }
  },
  { immediate: true }
);

watch(
  () => tokenStore.getWebSocketStatus(tokenStore.selectedToken?.id),
  (status) => {
    if (status === "connected") {
      getInfo();
    }
  }
);
</script>

<style scoped lang="scss">
.no-activity {
  text-align: center;
  padding: var(--spacing-lg);
  color: var(--text-secondary);
}

.tower-section {
  margin-bottom: var(--spacing-lg);
  padding-bottom: var(--spacing-md);
  border-bottom: 1px dashed var(--border-color);

  &:last-child {
    margin-bottom: 0;
    padding-bottom: 0;
    border-bottom: none;
  }
}

.header-info {
  display: flex;
  gap: var(--spacing-md);
  font-size: var(--font-size-sm);
  align-items: center;
  margin-bottom: var(--spacing-md);
  padding-bottom: var(--spacing-sm);
  border-bottom: 1px solid var(--border-color);
}

.challenge-count {
  font-weight: bold;
  color: var(--primary-color);
}

.daily-target {
  color: var(--text-secondary);
}

.boss-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: var(--spacing-sm);
  margin-bottom: var(--spacing-md);
}

.boss-card {
  background: var(--bg-secondary);
  border-radius: var(--border-radius-medium);
  padding: var(--spacing-sm);
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  border: 1px solid transparent;
  transition: all var(--transition-fast);
  
  &.active {
    background: #fff;
    border-color: var(--primary-color);
    box-shadow: var(--shadow-sm);
  }
  
  &.cleared {
    background: rgba(34, 197, 94, 0.05);
    border-color: var(--success-color);
  }
  
  &.locked {
    opacity: 0.7;
    background: var(--bg-tertiary);
  }
}

.expired-mask {
  text-align: center;
  color: var(--error-color);
  font-weight: bold;
  padding: var(--spacing-sm);
  background: rgba(239, 68, 68, 0.1);
  border-radius: var(--border-radius-medium);
  margin-bottom: var(--spacing-md);
}

.boss-grid.disabled {
  opacity: 0.5;
  pointer-events: none;
}

.boss-title {
  font-weight: bold;
  color: var(--primary-color);
  font-size: var(--font-size-md);
  margin-bottom: 4px;
}

.boss-level {
  font-size: var(--font-size-lg);
  font-weight: bold;
  margin-bottom: 8px;
}

.boss-status {
  margin-bottom: 8px;
}

.status-text {
  font-size: var(--font-size-sm);
  font-weight: bold;
  
  &.cleared {
    color: var(--success-color);
  }
  
  &.locked {
    color: var(--text-tertiary);
  }
  
  &.active {
    display: none; // Hide "进行中" text if button is there, or show it?
  }
}

.challenge-btn {
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: var(--border-radius-small);
  padding: 4px 12px;
  font-size: var(--font-size-xs);
  cursor: pointer;
  transition: background var(--transition-fast);
  
  &:disabled {
    background: var(--bg-tertiary);
    color: var(--text-tertiary);
    cursor: not-allowed;
  }
  
  &:not(:disabled):hover {
    background: var(--primary-color-hover);
  }
}

.action-row {
  margin-top: auto;
  display: flex;
  justify-content: flex-start;
}

.action-button {
  padding: var(--spacing-xs) var(--spacing-md);
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  border: none;
  border-radius: var(--border-radius-medium);
  cursor: pointer;
  background: var(--bg-tertiary);
  color: var(--text-secondary);

  &:hover {
    background: var(--bg-secondary);
  }
}

@media (max-width: 640px) {
  .boss-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}
</style>
