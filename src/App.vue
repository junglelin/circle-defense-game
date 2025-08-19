<template>
  <div
    style="width: 100vw; height: 100vh; overflow: hidden; margin: 0; padding: 0"
  >
    <canvas
      ref="canvas"
      :width="canvasWidth"
      :height="canvasHeight"
      style="display: block; width: 100vw; height: 100vh"
    ></canvas>
    <!-- HP 顯示移到 canvas -->
  </div>
</template>

<script setup lang="ts">
// COIN 參數
type CoinDot = {
  x: number;
  y: number;
  r: number;
  value: number;
  collected: boolean;
  duration: number; // 剩餘毫秒
  lastUpdate: number; // 最後更新時間戳
};
const coinDots = ref<CoinDot[]>([]);

function spawnCoinDot() {
  // 隨機位置
  const x = Math.random() * canvasWidth.value;
  const y = Math.random() * canvasHeight.value;
  const r = Math.random() * 8 + 8; // 半徑8~16
  const value = Math.random() * 2.4 + 0.1; // 0.1~2.5
  const duration = (Math.floor(Math.random() * 6) + 3) * 1000; // 3~8秒
  coinDots.value.push({
    x,
    y,
    r,
    value,
    collected: false,
    duration,
    lastUpdate: Date.now(),
  });
}

import { ref, onMounted, onBeforeUnmount } from "vue";

let animate: any = null;
let animationId: number | null = null;

const canvas = ref<HTMLCanvasElement | null>(null);
const canvasWidth = ref(window.innerWidth);
const canvasHeight = ref(window.innerHeight);
const hp = ref(10);
const radius = ref(5); // 直徑10px，實際繪製時會根據 hp 動態調整
const GAME_OVER_TEXT = "Game Over";
let gameOver = false;

// 灰點資料結構
type GrayDot = {
  x: number;
  y: number;
  r: number;
  speed: number;
  atk: number;
  vx: number;
  vy: number;
};
const grayDots = ref<GrayDot[]>([]);

function spawnGrayDot(speedBoost = 0) {
  // 隨機從四邊生成
  const side = Math.floor(Math.random() * 4);
  let x = 0,
    y = 0;
  if (side === 0) {
    // 上
    x = Math.random() * canvasWidth.value;
    y = 0;
  } else if (side === 1) {
    // 下
    x = Math.random() * canvasWidth.value;
    y = canvasHeight.value;
  } else if (side === 2) {
    // 左
    x = 0;
    y = Math.random() * canvasHeight.value;
  } else {
    // 右
    x = canvasWidth.value;
    y = Math.random() * canvasHeight.value;
  }
  // 隨機方向（單位向量）
  let angle = Math.random() * Math.PI * 2;
  // 讓灰點不會直接往紅點
  // 但仍有機率碰到紅點
  const speed = Math.random() * 1.5 + 1 + speedBoost;
  const atk = Math.random() * 2.4 + 0.1; // 0.1~0.5
  grayDots.value.push({
    x,
    y,
    r: Math.random() * 4 + 4, // 半徑4~8
    speed,
    atk,
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
  });
}

// Time 計時功能
const gameTime = ref(0); // 單位：毫秒
let timeInterval: any = null;

const isPaused = ref(false);

// 紅點直接跟隨鼠標
const redDotPos = ref({ x: canvasWidth.value / 2, y: canvasHeight.value / 2 });

function handleMouseMove(e: MouseEvent) {
  if (!canvas.value || gameOver || isPaused.value) return;
  const rect = canvas.value.getBoundingClientRect();
  const x = (e.clientX - rect.left) * (canvas.value.width / rect.width);
  const y = (e.clientY - rect.top) * (canvas.value.height / rect.height);
  redDotPos.value.x = x;
  redDotPos.value.y = y;
  draw();
}

const config = {
  coin: {
    minNum: 1, // 每次生成最少顆數
    maxNum: 5, // 每次生成最多顆數
    minValue: 0.1, // 金幣最小值
    maxValue: 5.5, // 金幣最大值
    minRadius: 8, // 金幣最小半徑
    maxRadius: 16, // 金幣最大半徑
    minExpire: 3, // 金幣最短消失秒數
    maxExpire: 8, // 金幣最長消失秒數
    fadeout: 2, // 金幣淡出秒數
  },
  coinAuto: {
    minInterval: 3000, // 最短 3 秒
    maxInterval: 5000, // 最長 5 秒
    minAdd: 0.1, // 最少加 0.1
    maxAdd: 1.0, // 最多加 1
  },
  redDot: {
    minRadius: 10, // 紅點最小半徑
    hpToRadius: 0.5, // HP 轉半徑倍率
    healMin: 0.1, // 點擊紅點最小回血
    healMax: 2.5, // 點擊紅點最大回血
  },
  grayDot: {
    minAtk: 0.1, // 灰點最小攻擊力
    maxAtk: 0.5, // 灰點最大攻擊力
    minRadius: 4, // 灰點最小半徑
    maxRadius: 8, // 灰點最大半徑
    minSpeed: 0.1, // 灰點最小速度
    maxSpeed: 2.5, // 灰點最大速度
    speedBoost: 0.15, // 灰點每10秒加速
    extraNumMin: 1, // 灰點每10秒最少額外生成
    extraNumMax: 3, // 灰點每10秒最多額外生成
  },
  coinIntervalMin: 5000, // 最短 5 秒
  coinIntervalMax: 10000, // 最長 10 秒
  coinExpireCheck: 500, // 金幣消失檢查間隔(ms)
  grayDotInterval: 700, // 灰點生成間隔(ms)
  grayDotBoostInterval: 10000, // 灰點加速/大量生成間隔(ms)
};

// 金幣定時生成（改為隨機 5~10 秒）
function scheduleCoinSpawn() {
  if (gameOver || isPaused.value) {
    setTimeout(scheduleCoinSpawn, config.coinIntervalMin); // 暫停時延後
    return;
  }
  const coinNum =
    Math.floor(Math.random() * (config.coin.maxNum - config.coin.minNum + 1)) +
    config.coin.minNum;
  for (let i = 0; i < coinNum; i++) spawnCoinDot();
  const nextInterval =
    Math.floor(
      Math.random() * (config.coinIntervalMax - config.coinIntervalMin + 1)
    ) + config.coinIntervalMin;
  setTimeout(scheduleCoinSpawn, nextInterval);
}

onMounted(() => {
  // 金幣定時生成
  scheduleCoinSpawn();
  // 金幣自動消失（剩餘時間遞減，Pause時不遞減）
  setInterval(() => {
    if (isPaused.value) {
      // 暫停時只更新lastUpdate，不遞減duration
      const now = Date.now();
      for (const c of coinDots.value) {
        c.lastUpdate = now;
      }
      return;
    }
    const now = Date.now();
    for (const c of coinDots.value) {
      if (!c.collected) {
        c.duration -= now - c.lastUpdate;
        c.lastUpdate = now;
        if (c.duration <= 0) c.collected = true;
      }
    }
  }, config.coinExpireCheck);
  window.addEventListener("resize", handleResize);
  draw();
  redDotPos.value = { x: canvasWidth.value / 2, y: canvasHeight.value / 2 };
  canvas.value?.addEventListener("mousemove", handleMouseMove);
  // 灰點生成與動畫
  animate = function () {
    if (gameOver || isPaused.value) {
      animationId = null;
      return;
    }
    // 灰點移動
    const cx = canvasWidth.value / 2;
    const cy = canvasHeight.value / 2;
    for (let i = grayDots.value.length - 1; i >= 0; i--) {
      const dot = grayDots.value[i];
      dot.x += dot.vx;
      dot.y += dot.vy;
      // 判斷碰撞紅點
      const dist = Math.sqrt((dot.x - cx) ** 2 + (dot.y - cy) ** 2);
      if (dist <= radius.value + dot.r) {
        hp.value -= dot.atk;
        grayDots.value.splice(i, 1);
        // 檢查遊戲結束
        if (hp.value <= 0) {
          hp.value = 0;
          gameOver = true;
          draw();
          animationId = null;
          return;
        }
        continue;
      }
      // 超出畫布也移除
      if (
        dot.x < -20 ||
        dot.x > canvasWidth.value + 20 ||
        dot.y < -20 ||
        dot.y > canvasHeight.value + 20
      ) {
        grayDots.value.splice(i, 1);
      }
    }
    draw();
    if (!gameOver && !isPaused.value)
      animationId = requestAnimationFrame(animate);
    else animationId = null;
  };
  animate();
  // 灰点定时生成
  let extraGrayCount = 0;
  let speedBoost = 0;
  let lastIncrease = Date.now();
  const spawnInterval = setInterval(() => {
    if (gameOver || isPaused.value) return;
    // 每10秒累加生成數量與速度
    const now = Date.now();
    if (now - lastIncrease >= config.grayDotBoostInterval) {
      extraGrayCount += Math.floor(Math.random() * 6) + 5; // 5~10
      speedBoost += 1.5; // 每次再加速
      lastIncrease = now;
    }
    spawnGrayDot();
    // 額外生成
    if (extraGrayCount > 0) {
      for (let i = 0; i < extraGrayCount; i++) spawnGrayDot(speedBoost);
    }
  }, config.grayDotInterval);
  // 遊戲時間計時器
  timeInterval = setInterval(() => {
    if (!isPaused.value && !gameOver) {
      gameTime.value += 10;
    }
  }, 10);
  // 清理
  onBeforeUnmount(() => {
    if (animationId) cancelAnimationFrame(animationId);
    clearInterval(spawnInterval);
    if (timeInterval) clearInterval(timeInterval);
  });
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", handleResize);
  canvas.value?.removeEventListener("mousemove", handleMouseMove);
});

function draw() {
  const ctx = canvas.value?.getContext("2d");
  if (!ctx) return;
  ctx.clearRect(0, 0, canvasWidth.value, canvasHeight.value);
  // 紅點
  const dynamicRadius = Math.max(
    config.redDot.minRadius,
    hp.value * config.redDot.hpToRadius
  );
  radius.value = dynamicRadius;
  ctx.save();
  ctx.beginPath();
  ctx.arc(redDotPos.value.x, redDotPos.value.y, dynamicRadius, 0, Math.PI * 2);
  ctx.fillStyle = "#e22";
  ctx.shadowColor = "#e22";
  ctx.shadowBlur = 10;
  ctx.fill();
  ctx.restore();

  // 金幣
  for (const c of coinDots.value) {
    if (c.collected) continue;
    let alpha = 0.85;
    if (c.duration < 1000) {
      alpha = 0.85 * Math.max(0, c.duration / 1000);
    }
    ctx.save();
    ctx.beginPath();
    ctx.arc(c.x, c.y, c.r, 0, Math.PI * 2);
    ctx.fillStyle = "#ffd700";
    ctx.shadowColor = "#ffd700";
    ctx.shadowBlur = 10;
    ctx.globalAlpha = alpha;
    ctx.fill();
    ctx.restore();
    // 顯示coin值
    ctx.save();
    ctx.font = "14px Arial";
    ctx.fillStyle = "#b8860b";
    ctx.textAlign = "center";
    ctx.globalAlpha = alpha;
    ctx.fillText(c.value.toFixed(2), c.x, c.y + 5);
    ctx.restore();
    // 紅點碰到金幣直接加 hp
    const dist = Math.sqrt(
      (redDotPos.value.x - c.x) ** 2 + (redDotPos.value.y - c.y) ** 2
    );
    if (!gameOver && !isPaused.value && dist <= radius.value + c.r) {
      hp.value += c.value;
      c.collected = true;
    }
  }

  // 灰點
  for (const dot of grayDots.value) {
    ctx.beginPath();
    ctx.arc(dot.x, dot.y, dot.r, 0, Math.PI * 2);
    ctx.fillStyle = "rgba(128, 128, 128, 0.8)";
    ctx.fill();
    ctx.closePath();
    // 紅點碰撞判斷
    const dist = Math.sqrt(
      (redDotPos.value.x - dot.x) ** 2 + (redDotPos.value.y - dot.y) ** 2
    );
    if (!gameOver && !isPaused.value && dist <= radius.value + dot.r) {
      hp.value -= dot.atk;
      grayDots.value.splice(grayDots.value.indexOf(dot), 1);
      if (hp.value <= 0) {
        hp.value = 0;
        gameOver = true;
        draw();
        return;
      }
    }
  }

  // HP Bar
  const hpBarWidth = 200;
  const hpBarHeight = 20;
  const hpBarX = 10;
  const hpBarY = 10;
  ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
  ctx.fillRect(hpBarX, hpBarY, hpBarWidth, hpBarHeight);
  ctx.fillStyle = "rgba(255, 0, 0, 0.8)";
  ctx.fillRect(hpBarX, hpBarY, (hp.value / 10) * hpBarWidth, hpBarHeight);

  // 遊戲時間
  ctx.fillStyle = "black";
  ctx.font = "16px Arial";
  ctx.fillText(
    `Time: ${formatTime(gameTime.value)}`,
    canvasWidth.value - 180,
    30
  );

  // 遊戲結束畫面
  if (gameOver) {
    ctx.fillStyle = "rgba(0, 0, 0, 0.8)";
    ctx.fillRect(0, 0, canvasWidth.value, canvasHeight.value);
    ctx.fillStyle = "white";
    ctx.font = "32px Arial";
    ctx.fillText(
      GAME_OVER_TEXT,
      canvasWidth.value / 2 - ctx.measureText(GAME_OVER_TEXT).width / 2,
      canvasHeight.value / 2
    );
    ctx.font = "24px Arial";
    ctx.fillText(
      `Time: ${formatTime(gameTime.value)}`,
      canvasWidth.value / 2 -
        ctx.measureText(`Time: ${formatTime(gameTime.value)}`).width / 2,
      canvasHeight.value / 2 + 40
    );
  }
}

function handleResize() {
  canvasWidth.value = window.innerWidth;
  canvasHeight.value = window.innerHeight;
  draw();
}

function formatTime(ms: number) {
  const totalSec = Math.floor(ms / 1000);
  const m = Math.floor(totalSec / 60);
  const s = totalSec % 60;
  const msStr = (ms % 1000).toString().padStart(3, "0");
  return `${m.toString().padStart(2, "0")}:${s
    .toString()
    .padStart(2, "0")}.${msStr}`;
}
</script>

<style>
body,
html {
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
