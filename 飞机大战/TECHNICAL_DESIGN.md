# 飞机大战网页小游戏 - 技术设计文档

> **文档目标**：为 Solo Builder 提供一份可直接据此编码的完整技术方案  
> **版本**：v1.0  
> **最后更新**：2026-07-31

---

## 目录

1. [技术选型](#1-技术选型)
   - [1.4 屏幕适配与响应式设计](#14-屏幕适配与响应式设计)
2. [游戏架构设计](#2-游戏架构设计)
   - [2.6 移动端触控控制方案](#26-移动端触控控制方案)
3. [核心玩法机制](#3-核心玩法机制)
4. [Boss战详细设计](#4-boss战详细设计)
5. [科技风视觉规范](#5-科技风视觉规范)
   - [5.4 音频系统设计](#54-音频系统设计)
6. [难度曲线与关卡设计](#6-难度曲线与关卡设计)
7. [性能优化要点](#7-性能优化要点)
8. [开发MVP优先级清单](#8-开发mvp优先级清单)

---

## 1. 技术选型

### 1.1 推荐方案

| 维度 | 选择 | 理由 |
|------|------|------|
| 渲染技术 | **Canvas 2D** | 零依赖、可控性高、社区资源丰富，适合solo builder快速迭代 |
| 编程语言 | **原生 JavaScript (ES2022+)** | 无需编译，热更新友好，直接运行 |
| 构建工具 | **无（纯HTML/CSS/JS）** | 保持极简，避免构建配置复杂度 |
| 音频 | **Web Audio API** | 原生支持，动态合成音效，无需音频文件 |
| 物理 | **自定义AABB碰撞检测** | 弹幕游戏无需复杂物理引擎，手写足够 |

### 1.2 为什么不用Phaser.js？

- 本项目是2D垂直卷轴射击游戏，Canvas 2D性能完全足够
- Phaser引入完整引擎 overhead，增加学习成本
- Solo Builder 需要直接控制渲染细节（粒子、弹幕），原生Canvas更灵活

### 1.3 项目文件结构

```
飞机大战/
├── index.html              # 入口HTML
├── css/
│   └── style.css           # 全局样式（HUD、菜单、动画）
├── js/
│   ├── main.js             # 游戏入口、初始化
│   ├── game/
│   │   ├── Game.js         # 游戏主循环、状态机
│   │   ├── SceneManager.js # 场景管理（菜单/游戏/暂停/结算）
│   │   └── InputManager.js # 键盘/触摸输入管理
│   ├── entities/
│   │   ├── Entity.js       # 实体基类（位置、速度、碰撞）
│   │   ├── Player.js       # 玩家飞机
│   │   ├── Bullet.js       # 子弹基类
│   │   ├── Enemy.js        # 敌机基类
│   │   ├── Boss.js         # Boss基类
│   │   ├── Particle.js     # 粒子特效
│   │   └── PowerUp.js      # 道具/拾取物
│   ├── systems/
│   │   ├── BulletSystem.js # 子弹池管理、碰撞检测
│   │   ├── ParticleSystem.js # 粒子系统
│   │   ├── WaveSystem.js   # 波次生成、关卡逻辑
│   │   ├── CollisionSystem.js # 碰撞检测（空间划分优化）
│   │   └── ScoreSystem.js  # 分数、连击、成就
│   ├── patterns/
│   │   ├── BulletPattern.js # 弹幕模式基类
│   │   ├── patterns/        # 各种弹幕模式实现
│   │   │   ├── SpreadPattern.js      # 散射
│   │   │   ├── SpiralPattern.js      # 螺旋
│   │   │   ├── AimedPattern.js       # 瞄准
│   │   │   ├── LaserPattern.js       # 激光
│   │   │   ├── RingPattern.js        # 环形
│   │   │   └── ComplexPattern.js     # 复合
│   ├── ui/
│   │   ├── HUD.js          # 游戏内UI（血量、能量、分数）
│   │   ├── Menu.js         # 主菜单
│   │   ├── ResultScreen.js # 结算画面
│   │   └── PauseMenu.js    # 暂停菜单
│   ├── audio/
│   │   └── AudioManager.js # 音频管理（BGM、SFX合成）
│   └── config/
│       ├── gameConfig.js   # 全局配置
│       ├── weaponConfig.js # 武器配置
│       └── bossConfig.js   # Boss配置
└── assets/
    └── (动态生成，无需外部资源)
```

### 1.4 屏幕适配与响应式设计

#### 1.4.1 设计原则

```
核心思路：逻辑分辨率固定 + 等比缩放 + 自适应居中

逻辑分辨率（设计基准）：
  - 竖屏模式: 480 × 800 (移动端标准)
  - 横屏模式: 800 × 480 (PC端可选)

适配策略：
  1. 计算可用空间的缩放比例
  2. 保持宽高比进行等比缩放
  3. 居中显示，留白填黑/背景
  4. 使用 Device Pixel Ratio 保证高清渲染
```

#### 1.4.2 Canvas尺寸适配方案

```javascript
class ScreenAdapter {
    // 逻辑分辨率（固定）
    DESIGN_WIDTH = 480;
    DESIGN_HEIGHT = 800;
    DESIGN_RATIO = 0.6;  // 480/800
    
    // 物理分辨率（动态计算）
    screenWidth = 0;
    screenHeight = 0;
    scale = 1.0;
    offsetX = 0;
    offsetY = 0;
    dpr = 1;
    
    // 当前方向
    orientation = 'portrait';  // portrait | landscape
    
    resize(container) {
        // 1. 获取容器尺寸
        const rect = container.getBoundingClientRect();
        const availableW = rect.width;
        const availableH = rect.height;
        
        // 2. 判断方向
        if (availableH >= availableW) {
            this.orientation = 'portrait';
            this.screenWidth = this.DESIGN_WIDTH;
            this.screenHeight = this.DESIGN_HEIGHT;
        } else {
            this.orientation = 'landscape';
            this.screenWidth = this.DESIGN_HEIGHT;
            this.screenHeight = this.DESIGN_WIDTH;
        }
        
        // 3. 计算等比缩放
        const scaleX = availableW / this.screenWidth;
        const scaleY = availableH / this.screenHeight;
        this.scale = Math.min(scaleX, scaleY);
        
        // 4. 计算居中偏移
        this.offsetX = (availableW - this.screenWidth * this.scale) / 2;
        this.offsetY = (availableH - this.screenHeight * this.scale) / 2;
        
        // 5. 设置Canvas物理尺寸（考虑DPR）
        this.dpr = window.devicePixelRatio || 1;
        canvas.width = this.screenWidth * this.scale * this.dpr;
        canvas.height = this.screenHeight * this.scale * this.dpr;
        
        // 6. 设置CSS尺寸
        canvas.style.width = this.screenWidth * this.scale + 'px';
        canvas.style.height = this.screenHeight * this.scale + 'px';
        canvas.style.left = this.offsetX + 'px';
        canvas.style.top = this.offsetY + 'px';
        
        // 7. 缩放绘图上下文
        ctx.setTransform(
            this.scale * this.dpr, 0, 0, 
            this.scale * this.dpr, 0, 0
        );
    }
    
    // 将屏幕坐标转换为逻辑坐标
    screenToLogic(screenX, screenY) {
        return {
            x: (screenX - this.offsetX) / this.scale,
            y: (screenY - this.offsetY) / this.scale
        };
    }
    
    // 监听窗口大小变化
    enableAutoResize(container) {
        const resizeHandler = () => this.resize(container);
        window.addEventListener('resize', resizeHandler);
        window.addEventListener('orientationchange', resizeHandler);
        
        // 监听容器变化（如侧边栏展开收起）
        if (window.ResizeObserver) {
            const observer = new ResizeObserver(resizeHandler);
            observer.observe(container);
        }
    }
}
```

#### 1.4.3 多设备适配配置表

| 设备类型 | 屏幕范围 | 缩放策略 | 特殊处理 |
|---------|---------|---------|---------|
| **小屏手机** | < 360px宽 | 缩小至适配 | 触控按钮放大 |
| **标准手机** | 360-428px宽 | 标准缩放 | HUD紧凑布局 |
| **大屏手机/平板** | > 428px宽 | 放大至适配 | 横屏自动切换 |
| **PC桌面** | > 768px宽 | 放大至全屏 | 键盘优先、鼠标可选 |
| **超高DPI屏** | devicePixelRatio > 2 | 启用高清渲染 | 禁用部分特效节省性能 |

#### 1.4.4 CSS容器样式

```css
/* 游戏容器 - 全屏自适应 */
#game-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: #0A0A1A;
    overflow: hidden;
    display: flex;
    align-items: center;
    justify-content: center;
    touch-action: none;  /* 阻止移动端默认手势 */
    user-select: none;
    -webkit-user-select: none;
    -webkit-tap-highlight-color: transparent;
}

/* Canvas样式 - 由JS动态设置尺寸 */
#game-canvas {
    position: absolute;
    image-rendering: pixelated;  /* 像素风格锐利渲染 */
    image-rendering: -moz-crisp-edges;
    image-rendering: crisp-edges;
}

/* 移动端HUD适配 */
@media (max-width: 480px) {
    .hud-bar {
        font-size: 12px;
        padding: 4px 8px;
    }
    .weapon-slots {
        gap: 4px;
    }
}

/* PC端HUD适配 */
@media (min-width: 768px) {
    .hud-bar {
        font-size: 14px;
        padding: 8px 16px;
    }
}
```

---

## 2. 游戏架构设计

### 2.1 核心设计模式

```
┌─────────────────────────────────────────────┐
│                 Game Engine                 │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  │
│  │  Scene  │  │  System  │  │  Entity   │  │
│  │ Manager │  │  Manager │  │  Manager  │  │
│  └─────────┘  └──────────┘  └───────────┘  │
│         │              │            │        │
│         ▼              ▼            ▼        │
│    ┌─────────────────────────────────────┐  │
│    │          Game Loop (主循环)           │  │
│    │   requestAnimationFrame 驱动          │  │
│    └─────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 2.2 游戏主循环

```javascript
// 核心伪代码
class Game {
    lastTime = 0;
    deltaTime = 0;
    FPS = 60;
    
    loop(currentTime) {
        // 计算时间增量，限制最大deltaTime防止卡顿穿透
        this.deltaTime = Math.min((currentTime - this.lastTime) / 1000, 0.1);
        this.lastTime = currentTime;
        
        // 固定时间步长更新逻辑
        this.update(this.deltaTime);
        
        // 渲染
        this.render();
        
        requestAnimationFrame(t => this.loop(t));
    }
    
    update(dt) {
        // 1. 输入更新
        this.inputManager.update();
        
        // 2. 场景逻辑更新
        this.currentScene.update(dt);
        
        // 3. 所有系统更新
        this.bulletSystem.update(dt);
        this.particleSystem.update(dt);
        this.waveSystem.update(dt);
        this.collisionSystem.update(dt);
        this.scoreSystem.update(dt);
    }
    
    render() {
        // 分层渲染（从下到上）
        this.bgLayer.render();      // 背景层
        this.bulletLayer.render();  // 子弹层
        this.entityLayer.render();  // 实体层
        this.fxLayer.render();      // 特效层
        this.hudLayer.render();     // UI层
    }
}
```

### 2.3 场景状态机

```
┌──────────┐  start   ┌──────────┐  boss出现  ┌──────────┐
│  MENU    │────────►│  PLAYING │─────────►│ BOSS_FIGHT│
└──────────┘         └──────────┘          └──────────┘
     ▲                   │    │                  │
     │                   │    │ pause            │ defeat/victory
     │                   │    ▼                  │
     │                   │ ┌────────┐           │
     │                   │ │PAUSED  │           │
     │                   │ └────────┘           │
     │                   │                      │
     │                   │ gameover/victory     │
     │                   ▼                      ▼
     │              ┌──────────┐          ┌──────────┐
     └──────────────│ RESULT   │◄─────────│ RESULT   │
     restart        └──────────┘  胜利    └──────────┘
```

### 2.4 实体系统设计

```javascript
// 实体基类 - 所有游戏对象的公共属性
class Entity {
    x, y;           // 位置（屏幕坐标）
    vx, vy;         // 速度
    width, height;   // 碰撞盒尺寸
    hp, maxHp;       // 生命值
    alive;          // 是否存活
    type;           // 类型标识 ('player'|'enemy'|'boss'|'bullet'|'powerup')
    hitFlash;       // 受击闪烁计时器
    
    update(dt) {}   // 更新逻辑
    render(ctx) {}  // 渲染
    getBounds() {}  // 获取碰撞矩形
    onHit(damage) {} // 被击中回调
    onDestroy() {}  // 销毁回调（粒子特效、音效等）
}
```

### 2.5 输入系统

```javascript
// 键盘控制
class InputManager {
    keys = {};           // 当前按下的键
    justPressed = {};    // 本帧刚按下的键
    
    // 控制映射
    // 移动: WASD / 方向键
    // 射击: J / 空格 (自动射击模式下可选)
    // 弹幕模式切换: K (短按切换)
    // 炸弹: L (清屏)
    // 暂停: ESC / P
}
```

### 2.6 移动端触控控制方案

#### 2.6.1 触控设计理念

```
核心原则：单手操作、直觉交互、不遮挡视野

控制方式（两种可选，自动检测）：
  1. 触屏拖动控制（主力方案）
     - 玩家手指在屏幕上拖动，飞机跟随移动
     - 松开手指飞机停留在当前位置
     - 适合竖屏单手操作
  
  2. 虚拟摇杆（备选方案）
     - 左下区域虚拟摇杆控制移动
     - 右下区域技能按钮
     - 适合喜欢经典操作的玩家

自动射击：
  - 移动端默认自动射击，无需单独按键
  - 专注于走位和技能释放
```

#### 2.6.2 触屏拖动控制实现

```javascript
class TouchController {
    // 触控状态
    activeTouch = null;       // { id, startX, startY, currentX, currentY }
    playerTarget = null;      // 玩家目标位置（逻辑坐标）
    moveThreshold = 5;        // 触发移动的最小像素阈值
    isDragging = false;
    
    // 手势识别
    lastTapTime = 0;
    doubleTapThreshold = 300; // 双击时间阈值(ms)
    
    init(canvas, screenAdapter) {
        this.canvas = canvas;
        this.adapter = screenAdapter;
        
        canvas.addEventListener('touchstart', (e) => this.onTouchStart(e), { passive: false });
        canvas.addEventListener('touchmove', (e) => this.onTouchMove(e), { passive: false });
        canvas.addEventListener('touchend', (e) => this.onTouchEnd(e));
        canvas.addEventListener('touchcancel', (e) => this.onTouchEnd(e));
    }
    
    onTouchStart(e) {
        e.preventDefault();
        const touch = e.touches[0];
        const logicPos = this.adapter.screenToLogic(touch.clientX, touch.clientY);
        
        this.activeTouch = {
            id: touch.identifier,
            startX: logicPos.x,
            startY: logicPos.y,
            currentX: logicPos.x,
            currentY: logicPos.y
        };
        
        this.isDragging = true;
        
        // 检测双击（切换弹幕模式）
        const now = Date.now();
        if (now - this.lastTapTime < this.doubleTapThreshold) {
            this.emitAction('switchWeapon');
        }
        this.lastTapTime = now;
    }
    
    onTouchMove(e) {
        e.preventDefault();
        if (!this.activeTouch) return;
        
        const touch = Array.from(e.touches).find(
            t => t.identifier === this.activeTouch.id
        );
        if (!touch) return;
        
        const logicPos = this.adapter.screenToLogic(touch.clientX, touch.clientY);
        this.activeTouch.currentX = logicPos.x;
        this.activeTouch.currentY = logicPos.y;
        
        // 更新玩家目标位置
        const dx = this.activeTouch.currentX - this.activeTouch.startX;
        const dy = this.activeTouch.currentY - this.activeTouch.startY;
        const distance = Math.sqrt(dx * dx + dy * dy);
        
        if (distance > this.moveThreshold) {
            this.playerTarget = {
                x: this.player.x + dx,
                y: this.player.y + dy
            };
            // 限制在屏幕范围内
            this.clampToScreen(this.playerTarget);
        }
    }
    
    onTouchEnd(e) {
        e.preventDefault();
        if (!this.activeTouch) return;
        
        const touch = Array.from(e.changedTouches).find(
            t => t.identifier === this.activeTouch.id
        );
        if (!touch) return;
        
        // 手指抬起，保持当前位置
        this.isDragging = false;
        this.activeTouch = null;
        
        // 检查是否为短按（单击）
        const now = Date.now();
        if (now - this.lastTapTime >= this.doubleTapThreshold) {
            // 单击可用于释放炸弹等
            // this.emitAction('bomb');
        }
    }
    
    clampToScreen(pos) {
        pos.x = Math.max(20, Math.min(this.adapter.screenWidth - 20, pos.x));
        pos.y = Math.max(30, Math.min(this.adapter.screenHeight - 30, pos.y));
    }
    
    update(player, dt) {
        if (this.isDragging && this.playerTarget) {
            // 平滑移动到目标位置（而非瞬间跳转）
            const lerpSpeed = 15;  // 插值速度
            player.x += (this.playerTarget.x - player.x) * lerpSpeed * dt;
            player.y += (this.playerTarget.y - player.y) * lerpSpeed * dt;
        }
    }
    
    emitAction(action) {
        // 触发游戏动作
        this.onAction?.(action);
    }
}
```

#### 2.6.3 移动端UI布局（竖屏）

```
┌─────────────────────────────────────────┐
│  HP ████████░░ 80/100  │  SCORE 128,500 │
│  SHIELD ████░░░░ 40    │  COMBO x128     │
├─────────────────────────────────────────┤
│                                         │
│                                         │
│              游戏区域                    │
│         (手指可操作区域)                  │
│                                         │
│                                         │
│  ┌─┐  ┌─┐  ┌─┐                         │
│  │散│  │聚│  │跟│   ← 武器槽(点击切换)    │
│  └─┘  └─┘  └─┘                         │
├─────────────────────────────────────────┤
│  [💣 BOMB]  [⚡ 激光]  [🏆 分数加成]    │
│   ← 技能按钮(底部固定)                   │
└─────────────────────────────────────────┘

触控交互说明：
  - 单指拖动: 飞机移动
  - 双击屏幕: 切换弹幕模式
  - 点击武器槽: 切换对应武器
  - 点击技能按钮: 释放对应技能
  - 双指轻点: 暂停游戏
```

#### 2.6.4 移动端虚拟摇杆方案（可选备选）

```javascript
class VirtualJoystick {
    // 摇杆属性
    baseX = 80;          // 摇杆基准X位置
    baseY = 650;         // 摇杆基准Y位置（左下）
    radius = 60;         // 摇杆活动半径
    knobRadius = 25;     // 摇杆柄半径
    active = false;
    currentVector = { x: 0, y: 0 };  // 归一化方向向量
    
    // 技能按钮布局（右下）
    skillButtons = [
        { id: 'bomb', x: 400, y: 680, radius: 30, label: '💣' },
        { id: 'weapon', x: 350, y: 620, radius: 25, label: '🔄' },
        { id: 'pause', x: 420, y: 580, radius: 22, label: '⏸️' }
    ];
    
    render(ctx) {
        if (this.active) {
            // 绘制摇杆底盘
            ctx.beginPath();
            ctx.arc(this.baseX, this.baseY, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = 'rgba(0, 255, 255, 0.1)';
            ctx.fill();
            ctx.strokeStyle = 'rgba(0, 255, 255, 0.5)';
            ctx.lineWidth = 2;
            ctx.stroke();
            
            // 绘制摇杆柄
            const knobX = this.baseX + this.currentVector.x * this.radius * 0.8;
            const knobY = this.baseY + this.currentVector.y * this.radius * 0.8;
            ctx.beginPath();
            ctx.arc(knobX, knobY, this.knobRadius, 0, Math.PI * 2);
            ctx.fillStyle = 'rgba(0, 255, 255, 0.6)';
            ctx.fill();
        }
        
        // 绘制技能按钮
        for (const btn of this.skillButtons) {
            ctx.beginPath();
            ctx.arc(btn.x, btn.y, btn.radius, 0, Math.PI * 2);
            ctx.fillStyle = 'rgba(255, 255, 255, 0.2)';
            ctx.fill();
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
            ctx.stroke();
            ctx.fillStyle = '#FFFFFF';
            ctx.font = '16px Arial';
            ctx.textAlign = 'center';
            ctx.fillText(btn.label, btn.x, btn.y + 5);
        }
    }
    
    updatePlayer(player, dt) {
        if (!this.active) {
            this.currentVector = { x: 0, y: 0 };
            return;
        }
        
        // 基于摇杆方向更新玩家移动
        const speed = player.speed * this.currentVector.length();
        player.x += this.currentVector.x * speed * dt;
        player.y += this.currentVector.y * speed * dt;
    }
}
```

#### 2.6.5 触控输入管理器整合

```javascript
class UnifiedInputManager {
    constructor() {
        this.isTouchDevice = 'ontouchstart' in window;
        this.keyboard = new KeyboardController();
        this.touch = this.isTouchDevice 
            ? new TouchController() 
            : null;
        this.joystick = this.isTouchDevice
            ? new VirtualJoystick()
            : null;
        
        // 根据设备类型选择控制方案
        this.controlMode = this.isTouchDevice 
            ? 'touch_drag'   // 或 'virtual_joystick'
            : 'keyboard';
    }
    
    getMovementVector() {
        if (this.isTouchDevice && this.controlMode === 'virtual_joystick') {
            return this.joystick.currentVector;
        }
        // 键盘模式或触屏拖动由TouchController直接处理位置
        return this.keyboard.getMovementVector();
    }
    
    isActionTriggered(action) {
        // 统一的动作查询接口
        if (this.keyboard.isPressed(action)) return true;
        if (this.touch?.isActionTriggered(action)) return true;
        if (this.joystick?.isSkillPressed(action)) return true;
        return false;
    }
}
```

#### 2.6.6 触控体验优化清单

```
[ ] 1. 触觉反馈
    - 释放炸弹时震动 (navigator.vibrate)
    - 受击时短震动
    - 切换武器时微震动
    
[ ] 2. 防误触
    - 技能按钮设为按下触发+确认区
    - 滑动阈值5px防止手抖误触
    - 暂停按钮放边角区域
    
[ ] 3. 操作反馈
    - 手指下方显示半透明操作指示
    - 拖动时显示移动范围提示
    - 武器切换有视觉+听觉反馈
    
[ ] 4. 多指支持
    - 允许多指同时操作（移动+技能）
    - 区分不同手指的触发区域
    
[ ] 5. 横屏支持
    - 自动检测方向并调整UI
    - 横屏时按钮重新布局
    - 提供竖屏优先提示
```

---

## 3. 核心玩法机制

### 3.1 参考游戏与借鉴机制

| 参考游戏 | 借鉴机制 | 融合方案 |
|---------|---------|---------|
| **东方Project** | 密集弹幕、擦弹判定、符卡系统 | 引入「弹幕模式」切换，高速移动时子弹变密，擦弹加分 |
| **雷霆战机** | 装备合成、僚机系统、关卡制Boss | 武器升级树 + 僚机配置系统 |
| **怒之风暴** | 多武器切换、炸弹清屏、分数连击 | 多弹幕模式切换（散射/激光/跟踪），连击系统 |
| **Cave Story** | 武器等级系统、蓄力攻击 | 武器随连击升级，蓄力释放特殊攻击 |
| **Deathsmiles** | 难度自适、 gothic风格、多路线 | 动态难度调整，分支关卡选择 |

### 3.2 玩家核心系统

#### 3.2.1 移动与操作

```
┌─────────────────────────────────────────────────────┐
│                    玩家飞机控制                       │
├─────────────────────────────────────────────────────┤
│  WASD/方向键: 8方向自由移动                           │
│  Shift: 慢速精准移动（躲避弹幕时使用）                │
│  空格/J: 射击（默认自动射击，按键加强射速）           │
│  K键: 切换弹幕模式（3种模式）                        │
│  L键: 释放炸弹（清屏+无敌3秒）                       │
│  ESC/P: 暂停游戏                                    │
└─────────────────────────────────────────────────────┘
```

#### 3.2.2 弹幕模式系统（核心创新点）

玩家可在3种弹幕模式间自由切换，每种模式有不同的弹道特性：

```javascript
const WEAPON_MODES = {
    SPREAD: {
        name: '散射',
        desc: '8方向弹幕，近距离覆盖范围大',
        damage: 15,
        fireRate: 0.08,       // 秒/发
        bulletCount: 5,       // 同时发射5发
        spreadAngle: 60,      // 散射角度
        color: '#00FFFF',
        bulletSpeed: 600,
        special: '近距离伤害+50%'
    },
    
    FOCUS: {
        name: '聚焦',
        desc: '高速穿透直线，远距离高伤害',
        damage: 40,
        fireRate: 0.12,
        bulletCount: 1,
        spreadAngle: 0,
        color: '#FF00FF',
        bulletSpeed: 900,
        special: '穿透3个敌人'
    },
    
    TRACK: {
        name: '跟踪',
        desc: '自动锁定最近敌人，智能追踪',
        damage: 25,
        fireRate: 0.15,
        bulletCount: 3,
        color: '#FFFF00',
        bulletSpeed: 500,
        special: '自动锁定+穿透'
    }
};
```

**切换逻辑**：
- 每次切换有0.5秒冷却
- 切换瞬间发射一次"切换冲击波"（伤害+视觉反馈）
- HUD显示当前模式图标和剩余冷却

#### 3.2.3 武器升级系统

```
武器等级（每级提升伤害/射速/子弹数）:
  Lv.1 → Lv.2 → Lv.3 → Lv.4 → Lv.MAX (Lv.5)

升级方式：
  1. 拾取「升级道具」（金色菱形）
  2. 连击达到100/300/500时自动升级
  3. Boss掉落专属升级

每级提升：
  伤害 +20%
  射速 +10%
  子弹数 +1（上限8发）
```

#### 3.2.4 护盾/生命值系统

```
玩家状态条:
┌─────────────────────────────────────────┐
│  HP: ████████░░ 80/100                  │
│  Shield: ████░░░░ 40/100 (自动恢复)     │
│  Energy: ████████████ 100/100 (炸弹)   │
│  Combo: x128 🔥                         │
│  Weapon: [散射 Lv.3]                   │
└─────────────────────────────────────────┘

机制说明：
- HP: 受击扣除，归零则GameOver
- Shield: 自动恢复（每秒5点），优先于HP扣除
- Energy: 累积满100可释放炸弹（清屏+3秒无敌）
- Combo: 连续击杀不中断，连击越高分数加成越多
```

#### 3.2.5 炸弹系统

```javascript
class Bomb {
    activate() {
        // 1. 全屏清除所有敌方子弹
        // 2. 对全屏敌人造成大量伤害（1000点）
        // 3. 玩家3秒无敌
        // 4. 触发全屏粒子爆炸特效
        // 5. 屏幕震动
        // 6. 分数奖励（当前连击x10）
    }
}
```

### 3.3 敌人系统

#### 3.3.1 敌人类型

| 类型 | 特点 | 弹幕模式 | 生命值 | 分数 |
|------|------|---------|--------|------|
| **小兵 (Grunt)** | 直线移动，偶尔变向 | 单发直线弹 | 30 | 100 |
| **巡逻兵 (Patrol)** | 左右蛇形移动 | 散射3连发 | 60 | 200 |
| **神射手 (Sniper)** | 固定位置瞄准玩家 | 蓄力瞄准射击 | 100 | 400 |
| **轰炸机 (Bomber)** | 慢速高血量 | 圆形弹幕爆发 | 200 | 800 |
| **精英 (Elite)** | 快速不规则移动 | 复合弹幕模式 | 500 | 2000 |

#### 3.3.2 敌人生成系统

```javascript
class WaveSystem {
    // 关卡波次配置
    waves = [
        { time: 0,   enemies: [5个小兵] },
        { time: 5,   enemies: [3个巡逻兵, 2个小兵] },
        { time: 10,  enemies: [2个神射手, 4个小兵] },
        { time: 15,  enemies: [1个轰炸机, 3个巡逻兵] },
        { time: 20,  enemies: [5个神射手, 2个巡逻兵] },
        { time: 25,  enemies: [2个轰炸机, 4个小兵] },
        { time: 30,  enemies: [1个精英] },
        { time: 35,  enemies: [3个神射手, 2个轰炸机] },
        { time: 40,  enemies: [2个精英, 4个巡逻兵] },
        { time: 45,  enemies: [BOSS出现] }
    ];
    
    // 动态难度：根据玩家表现调整
    dynamicDifficulty: {
        // 玩家剩余HP越高 → 敌人攻击越频繁
        // 连击越高 → 敌人血量略增
        // 死亡次数 → 减少敌人数量
    }
}
```

### 3.4 弹幕模式库

#### 3.4.1 基础弹幕类型

```javascript
// 1. 散射弹幕
class SpreadPattern {
    createBullets(origin, target) {
        const bullets = [];
        const baseAngle = Math.atan2(target.y - origin.y, target.x - origin.x);
        const count = 5;
        const spread = Math.PI / 4; // 45度散射
        
        for (let i = 0; i < count; i++) {
            const angle = baseAngle - spread/2 + (spread * i / (count-1));
            bullets.push(new Bullet(
                origin.x, origin.y,
                Math.cos(angle) * speed,
                Math.sin(angle) * speed
            ));
        }
        return bullets;
    }
}

// 2. 螺旋弹幕（东方风格）
class SpiralPattern {
    createBullets(origin, time) {
        // 持续旋转发射，适合Boss
        const angle = time * this.rotationSpeed;
        const bullets = [];
        for (let i = 0; i < this.arms; i++) {
            const a = angle + (Math.PI * 2 * i / this.arms);
            bullets.push(new Bullet(
                origin.x, origin.y,
                Math.cos(a) * speed,
                Math.sin(a) * speed
            ));
        }
        return bullets;
    }
}

// 3. 环形爆发弹幕
class RingPattern {
    createBullets(origin) {
        const bullets = [];
        const count = 16;
        for (let i = 0; i < count; i++) {
            const angle = (Math.PI * 2 * i / count);
            bullets.push(new Bullet(
                origin.x, origin.y,
                Math.cos(angle) * speed,
                Math.sin(angle) * speed
            ));
        }
        return bullets;
    }
}

// 4. 瞄准激光
class LaserPattern {
    createBeam(start, target) {
        // 即时激光，有预警线
        // 1. 显示2秒预警线
        // 2. 发射激光持续0.5秒
        // 3. 造成持续伤害
    }
}

// 5. 复合弹幕（多种组合）
class ComplexPattern {
    // 同时发射散射+环形+瞄准
    createBullets(origin, target, time) {
        const bullets = [];
        // 组合多种基础pattern
        bullets.push(...spread.createBullets(origin, target));
        bullets.push(...ring.createBullets(origin));
        bullets.push(...spiral.createBullets(origin, time));
        return bullets;
    }
}
```

#### 3.4.2 弹幕配置参数

```javascript
const BULLET_PATTERNS = {
    // 敌人基础弹幕
    enemy_basic: {
        bulletSpeed: 250,
        bulletSize: 6,
        bulletColor: '#FF6B6B',
        damage: 10
    },
    // Boss弹幕（更大更慢但伤害高）
    boss_normal: {
        bulletSpeed: 180,
        bulletSize: 10,
        bulletColor: '#FF00FF',
        damage: 20
    },
    boss_fast: {
        bulletSpeed: 400,
        bulletSize: 5,
        bulletColor: '#FFFF00',
        damage: 15
    },
    boss_warning: {
        bulletSpeed: 0,  // 预警线
        warningTime: 1.5,
        color: '#FF0000'
    }
};
```

### 3.5 连击与分数系统

```javascript
class ScoreSystem {
    score = 0;
    combo = 0;
    maxCombo = 0;
    comboTimer = 0;          // 连击计时器（2秒内未击杀则重置）
    comboMultiplier = 1.0;    // 分数倍率
    
    // 连击加成表
    comboTable = {
        10:  { multiplier: 1.5, label: 'GOOD!' },
        30:  { multiplier: 2.0, label: 'GREAT!' },
        50:  { multiplier: 3.0, label: 'EXCELLENT!' },
        100: { multiplier: 5.0, label: 'AMAZING!' },
        200: { multiplier: 8.0, label: 'INCREDIBLE!' },
        500: { multiplier: 15.0, label: 'LEGENDARY!' }
    };
    
    onEnemyKilled(enemy) {
        this.combo++;
        this.comboTimer = 2.0;
        this.score += enemy.score * this.comboMultiplier;
        this.checkComboMilestone();
    }
    
    onBulletGraze() {
        // 擦弹加分（擦弹：子弹接近玩家但未命中）
        this.score += 50 * this.comboMultiplier;
    }
}
```

### 3.6 道具系统

```javascript
const POWERUP_TYPES = {
    // 武器升级
    WEAPON_UPGRADE: {
        color: '#FFD700',
        shape: 'diamond',
        effect: 'weapon_level +1',
        duration: '永久直到死亡'
    },
    // 护盾恢复
    SHIELD_RESTORE: {
        color: '#00BFFF',
        shape: 'circle',
        effect: 'shield +50',
        duration: '即时'
    },
    // 生命恢复
    HP_RESTORE: {
        color: '#FF4444',
        shape: 'heart',
        effect: 'hp +30',
        duration: '即时'
    },
    // 炸弹补给
    BOMB_SUPPLY: {
        color: '#FF6600',
        shape: 'bomb',
        effect: 'energy +50',
        duration: '即时'
    },
    // 僚机召唤
    SUMMON_WINGMAN: {
        color: '#9900FF',
        shape: 'wing',
        effect: 'wingman +1 (最多2个)',
        duration: '永久直到死亡'
    },
    // 分数加成
    SCORE_BOOST: {
        color: '#00FF7F',
        shape: 'star',
        effect: 'score_multiplier x2 for 10s',
        duration: '10秒'
    }
};
```

---

## 4. Boss战详细设计

### 4.1 Boss通用设计框架

```javascript
class Boss extends Entity {
    phases = [];              // 多阶段配置
    currentPhase = 0;
    phaseTimer = 0;
    attackTimer = 0;
    attackPatterns = [];      // 当前阶段的弹幕模式池
    isInvulnerable = false;
    warningText = '';
    
    // Boss血条（顶部）
    maxHp;
    currentHp;
    
    // 阶段转换条件
    phaseSwitchConditions = [
        { hpPercent: 0.7, nextPhase: 1 },  // HP70%时切换
        { hpPercent: 0.4, nextPhase: 2 },  // HP40%时切换
        { hpPercent: 0.15, nextPhase: 3 }  // HP15%时切换（狂暴）
    ];
}
```

### 4.2 Boss设计示例：虚空指挥官（第一关Boss）

#### 4.2.1 外观与主题

```
名称：虚空指挥官 (Void Commander)
主题：深空指挥舰，紫色能量核心
尺寸：约120x80像素（比玩家大4倍）
视觉特效：
  - 核心持续脉动发光
  - 受损时外壳迸裂露出内部
  - 狂暴模式下全舰变红
```

#### 4.2.2 多阶段行为状态机

```
┌─────────────────────────────────────────────────────────┐
│                    BOSS PHASES                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Phase 1 - 侦察模式 (100% → 70% HP)                    │
│  ├── 移动: 缓慢左右巡逻                                 │
│  ├── 攻击: 单发瞄准弹 (低频)                            │
│  ├── 弹幕: [瞄准直线, 扇形3连发]                        │
│  └── 难度: ★☆☆☆☆                                       │
│                                                         │
│  Phase 2 - 警戒模式 (70% → 40% HP)                     │
│  ├── 移动: 快速V字移动                                   │
│  ├── 攻击: 散射弹幕 + 瞄准                              │
│  ├── 弹幕: [散射5连发, 环形8连发, 螺旋4臂]             │
│  ├── 新技能: 召唤2个小兵                                │
│  └── 难度: ★★★☆☆                                       │
│                                                         │
│  Phase 3 - 进攻模式 (40% → 15% HP)                     │
│  ├── 移动: 冲刺攻击 + 快速闪避                          │
│  ├── 攻击: 高密度弹幕                                  │
│  ├── 弹幕: [复合散射, 激光预警, 全弹雨]                │
│  ├── 新技能: 激光扫射 + 召唤4个巡逻兵                   │
│  └── 难度: ★★★★☆                                       │
│                                                         │
│  Phase 4 - 狂暴模式 (15% → 0% HP)                      │
│  ├── 移动: 疯狂不规则移动                               │
│  ├── 攻击: 终极弹幕风暴                                │
│  ├── 弹幕: [全方向螺旋, 多重环形, 交叉激光]             │
│  ├── 特殊: 持续释放伤害区                               │
│  └── 难度: ★★★★★                                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 4.2.3 详细弹幕Pattern配置

```javascript
const VOID_COMMANDER_PHASES = [
    // ========== Phase 1 ==========
    {
        name: '侦察模式',
        duration: 'until 70% HP',
        movement: {
            type: 'patrol',      // patrol | v_motion | dash | erratic
            speed: 60,
            pattern: 'horizontal',
            range: [100, 700]    // x范围
        },
        attacks: [
            {
                type: 'aimed_single',
                interval: 1.2,
                pattern: BulletPatterns.Aimed.create(1, 250),
                damage: 15
            },
            {
                type: 'fan_3',
                interval: 2.0,
                pattern: BulletPatterns.Spread.create(3, 200, 30),
                damage: 12
            }
        ],
        special: null
    },
    
    // ========== Phase 2 ==========
    {
        name: '警戒模式',
        duration: 'until 40% HP',
        movement: {
            type: 'v_motion',
            speed: 120,
            pattern: 'v_shape',
            amplitude: 150
        },
        attacks: [
            {
                type: 'spread_5',
                interval: 0.8,
                pattern: BulletPatterns.Spread.create(5, 280, 45),
                damage: 15
            },
            {
                type: 'ring_8',
                interval: 3.0,
                pattern: BulletPatterns.Ring.create(8, 200),
                damage: 12,
                delay: 0.5  // 攻击间隔内的额外延迟
            },
            {
                type: 'spiral_4',
                interval: 1.5,
                pattern: BulletPatterns.Spiral.create(4, 200, 2.0),
                damage: 10
            }
        ],
        special: {
            type: 'summon_minions',
            interval: 8.0,
            count: 2,
            enemyType: 'grunt'
        }
    },
    
    // ========== Phase 3 ==========
    {
        name: '进攻模式',
        duration: 'until 15% HP',
        movement: {
            type: 'dash',
            speed: 180,
            dashInterval: 4.0,
            dashDistance: 200
        },
        attacks: [
            {
                type: 'complex_burst',
                interval: 1.0,
                pattern: BulletPatterns.Complex.create([
                    BulletPatterns.Spread.create(7, 300, 60),
                    BulletPatterns.Ring.create(12, 180)
                ]),
                damage: 18
            },
            {
                type: 'laser_warning',
                interval: 3.5,
                pattern: BulletPatterns.Laser.create(300, 0.3),
                damage: 25,
                warningTime: 1.5
            },
            {
                type: 'aimed_spread',
                interval: 2.0,
                pattern: BulletPatterns.Spread.create(5, 280, 80),
                damage: 16
            }
        ],
        special: {
            type: 'summon_minions',
            interval: 6.0,
            count: 4,
            enemyType: 'patrol'
        }
    },
    
    // ========== Phase 4 ==========
    {
        name: '狂暴模式',
        duration: 'until 0% HP',
        movement: {
            type: 'erratic',
            speed: 220,
            changeInterval: 0.8
        },
        attacks: [
            {
                type: 'spiral_8_fast',
                interval: 0.3,
                pattern: BulletPatterns.Spiral.create(8, 280, 4.0),
                damage: 12
            },
            {
                type: 'double_ring',
                interval: 1.2,
                pattern: BulletPatterns.Complex.create([
                    BulletPatterns.Ring.create(16, 250),
                    BulletPatterns.Ring.create(12, 180)
                ], 0.3),
                damage: 15
            },
            {
                type: 'cross_laser',
                interval: 4.0,
                pattern: BulletPatterns.Laser.create([45, 135, 225, 315], 0.5),
                damage: 30,
                warningTime: 1.0
            },
            {
                type: 'damage_zone',
                interval: 2.0,
                effect: '持续伤害区域',
                damage: 5
            }
        ],
        special: {
            type: 'berserk',
            hpDrain: 0.02  // 每秒消耗2%玩家HP
        }
    }
];
```

#### 4.2.4 Boss战流程伪代码

```javascript
class BossFight {
    start(boss) {
        // 1. Boss入场动画（从屏幕上方缓慢出现）
        boss.playEnterAnimation();
        
        // 2. 显示Boss名称（中央闪烁3秒）
        ui.showBossName('虚空指挥官', 3);
        
        // 3. 开始Phase 1
        boss.setPhase(0);
        boss.startAttacking();
        
        // 4. Boss血条出现
        ui.showBossHealthBar(boss);
    }
    
    update(dt) {
        // Boss攻击逻辑
        boss.attackTimer -= dt;
        if (boss.attackTimer <= 0) {
            boss.executeRandomAttack();
            boss.attackTimer = boss.currentAttackInterval;
        }
        
        // 阶段转换检测
        const hpPercent = boss.currentHp / boss.maxHp;
        if (hpPercent <= boss.nextPhaseSwitch.hpPercent) {
            this.switchPhase(boss);
        }
        
        // 特殊技能检测
        if (boss.special && boss.specialTimer <= 0) {
            boss.executeSpecial();
        }
    }
    
    switchPhase(boss) {
        // 1. 停止所有攻击
        boss.clearAllBullets();
        
        // 2. 显示阶段转换提示
        ui.showPhaseWarning(boss.currentPhase + 1);
        
        // 3. 1.5秒后切换
        setTimeout(() => {
            boss.currentPhase++;
            boss.applyPhaseConfig();
            boss.resumeAttacking();
        }, 1500);
    }
    
    onBossDefeated(boss) {
        // 1. 停止攻击
        boss.stopAttacking();
        
        // 2. 播放爆炸动画（分多次爆炸）
        for (let i = 0; i < 5; i++) {
            setTimeout(() => {
                boss.explodeParticles(i * 0.5);
            }, i * 300);
        }
        
        // 3. 掉落战利品
        this.dropLoot(boss);
        
        // 4. 显示胜利画面
        setTimeout(() => {
            ui.showVictoryScreen();
        }, 3000);
        
        // 5. 计算分数
        scoreSystem.addBossScore(boss);
    }
}
```

### 4.3 多Boss路线设计

```
关卡进度:
  Stage 1: 训练场 → 中Boss: 虚空指挥官 (已设计)
  Stage 2: 小行星带 → 中Boss: 冰晶守护者 (冰属性弹幕)
  Stage 3: 太空站 → 中Boss: 钢铁巨像 (激光/火箭)
  Stage 4: 虫洞 → 中Boss: 时空撕裂者 (随机传送)
  Stage 5: 核心 → 最终Boss: 创世主 (三阶段终极形态)
```

---

## 5. 科技风视觉规范

### 5.1 配色方案

```
主色调（深空背景）:
  - 深空黑:     #0A0A1A (背景)
  - 星尘蓝:     #1A1A3A (次要背景)
  - 星云紫:     #2D1B4E (渐变)

强调色（霓虹效果）:
  - 电光青:     #00FFFF (玩家子弹)
  - 能量紫:     #9D00FF (Boss弹幕)
  - 警报红:     #FF3333 (危险区域)
  - 活力黄:     #FFD700 (奖励/金色)

功能色:
  - 生命绿:     #00FF7F (HP/恢复)
  - 护盾蓝:     #00BFFF (Shield)
  - 能量橙:     #FF6B35 (Bomb)
```

### 5.2 视觉特效

#### 5.2.1 粒子系统

```javascript
class ParticleSystem {
    // 粒子类型
    types = {
        EXPLOSION: {
            // 爆炸: 中心向外扩散的圆形粒子
            shape: 'circle',
            colors: ['#FF6B35', '#FFD700', '#FF3333'],
            size: [2, 8],
            speed: [50, 200],
            lifetime: [0.3, 0.8]
        },
        SPARK: {
            // 火花: 小而快速的线条
            shape: 'line',
            colors: ['#FFFF00', '#FFFFFF'],
            size: [1, 3],
            speed: [300, 500],
            lifetime: [0.1, 0.3]
        },
        TRAIL: {
            // 拖尾: 跟随物体的尾迹
            shape: 'circle',
            colors: ['#00FFFF'],
            size: [1, 4],
            speed: [0, 50],
            lifetime: [0.2, 0.5]
        },
        ENERGY: {
            // 能量: 脉动的能量球
            shape: 'ring',
            colors: ['#9D00FF', '#00FFFF'],
            size: [5, 15],
            speed: [20, 80],
            lifetime: [0.5, 1.5]
        }
    };
}
```

#### 5.2.2 背景效果

```javascript
class BackgroundSystem {
    // 多层视差滚动
    layers = [
        {
            // 远景: 缓慢移动的星云
            speed: 10,
            objects: 'nebulae',
            count: 5
        },
        {
            // 中景: 星星群
            speed: 50,
            objects: 'stars',
            count: 100
        },
        {
            // 近景: 快速移动的小行星
            speed: 150,
            objects: 'asteroids',
            count: 5
        }
    ];
    
    // 动态星云（使用Canvas绘制）
    nebulaColors = ['#2D1B4E', '#1A0A3A', '#0A0A2A'];
    
    // 星空闪烁效果
    starTwinkle: true;
    twinkleSpeed: 2.0;
}
```

#### 5.2.3 屏幕效果

```javascript
class ScreenEffect {
    // 受击红屏
    hitFlash: {
        color: 'rgba(255, 0, 0, 0.3)',
        duration: 0.15
    };
    
    // 炸弹白屏
    bombFlash: {
        color: 'rgba(255, 255, 255, 0.8)',
        duration: 0.5
    };
    
    // 屏幕震动
    shake: {
        intensity: [3, 8],     // 像素偏移
        duration: [0.1, 0.3]
    };
    
    // 扫描线效果（科技风）
    scanlines: {
        color: 'rgba(0, 255, 255, 0.03)',
        height: 2,
        gap: 4
    };
    
    // 光晕效果
    bloom: {
        enabled: true,
        intensity: 0.5
    }
}
```

### 5.3 UI风格

```
HUD布局:
┌─────────────────────────────────────────────────────┐
│  HP: ████████░░ 80/100   │  SCORE: 128,500          │
│  SHIELD: ████░░░░ 40    │  COMBO: x128 🔥           │
│  ENERGY: ████████ 100   │  STAGE: 1-3               │
│                                                         │
│  [散射 Lv.3]  [聚焦 Lv.2]  [跟踪 Lv.1]  ← 武器槽   │
├─────────────────────────────────────────────────────┤
│                                                         │
│                   GAME AREA                             │
│                                                         │
├─────────────────────────────────────────────────────┤
│  激光槽: ░░░░░░░  │  僚机: ✈✈  │  BOMB: [L键]       │
└─────────────────────────────────────────────────────┘

UI字体: 'Orbitron' (Google Fonts - 科技风)
UI效果: 
  - 霓虹发光边框 (box-shadow)
  - 动态闪烁指示灯
  - 切角几何形状 (clip-path)
```

### 5.4 音频系统设计

#### 5.4.1 音频架构概览

```
技术方案：Web Audio API 动态合成（零外部音频文件依赖）

架构设计：
┌─────────────────────────────────────────────────────────┐
│                    AudioManager                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   BGM 合成器  │  │   SFX 合成器  │  │  音量管理器   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              AudioContext 核心                     │   │
│  │    (OscillatorNode + GainNode + FilterNode)       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘

音频触发时机：
  - 用户首次交互后才初始化AudioContext（浏览器策略要求）
  - BGM循环播放，战斗/菜单切换时淡入淡出
  - SFX按需播放，支持并发
```

#### 5.4.2 BGM 音乐风格定义

```
整体风格：Synthwave / Darkwave 科技电子风
参考：《Outrun》《Hyper Light Drifter》游戏原声带

BGM曲目列表：

┌──────────────────────────────────────────────────────────────┐
│ Track 1: 主菜单 - "Starlight Gateway"                         │
│ 风格: 氛围感合成器，缓慢律动                                    │
│ BPM: 85                                                       │
│ 乐器: 复古合成器贝斯 + 琶音和弦 + 太空音效                     │
│ 情绪: 神秘、期待、史诗感                                      │
│ 时长: 循环60秒                                                │
│                                                              │
│ Track 2: 普通关卡 - "Void Runner"                             │
│ 风格: 快节奏电子，紧张感                                       │
│ BPM: 130                                                      │
│ 乐器: 驱动贝斯 + 合成器主调 + 鼓机节拍                         │
│ 情绪: 紧张、动感、战斗欲                                      │
│ 时长: 循环60秒                                                │
│                                                              │
│ Track 3: Boss战 - "Overload Protocol"                         │
│ 风格: 重型电子 + 工业噪音                                      │
│ BPM: 150                                                      │
│ 乐器: 失真合成器 + 重金属贝斯 + 密集鼓点 + 警报音效            │
│ 情绪: 压迫、危险、肾上腺素                                    │
│ 时长: 循环45秒                                                │
│                                                              │
│ Track 4: 胜利结算 - "Victory Signal"                          │
│ 风格: 胜利号角，振奋人心                                       │
│ BPM: 120                                                      │
│ 乐器: 合成铜管 + 胜利节拍 + 欢呼声                             │
│ 情绪: 成就、荣耀、庆祝                                        │
│ 时长: 30秒（非循环）                                          │
│                                                              │
│ Track 5: 游戏结束 - "System Shutdown"                        │
│ 风格: 低沉哀鸣，系统崩溃感                                     │
│ BPM: 60                                                       │
│ 乐器: 低频合成器 + 故障噪音 + 缓慢弦乐                         │
│ 情绪: 失落、科技末日、神秘                                    │
│ 时长: 20秒（非循环）                                          │
└──────────────────────────────────────────────────────────────┘
```

#### 5.4.3 BGM 合成实现

```javascript
class BGMSynthesizer {
    audioContext;
    masterGain;
    currentTrack = null;
    isPlaying = false;
    
    // BGM曲目配置（MIDI音高 + 波形类型 + 节奏）
    tracks = {
        menu: {
            bpm: 85,
            waveform: 'sine',
            // 和弦进行 (Am - F - C - G)
            chordProgression: [
                [220.00, 261.63, 329.63],  // A minor
                [174.61, 220.00, 261.63],  // F major
                [261.63, 329.63, 392.00],  // C major
                [196.00, 246.94, 293.66]   // G major
            ],
            melody: [
                440, 523, 659, 523, 440, 392, 440, 523,
                392, 440, 523, 659, 784, 659, 523, 440
            ],
            volume: 0.15
        },
        
        battle: {
            bpm: 130,
            waveform: 'sawtooth',
            bassline: [
                110, 110, 146.83, 110, 
                164.81, 110, 146.83, 130.81
            ],
            leadMelody: [
                329, 392, 440, 523, 440, 392, 329, 294,
                330, 392, 494, 587, 494, 392, 330, 294
            ],
            volume: 0.2
        },
        
        boss: {
            bpm: 150,
            waveform: 'square',
            // 低音riff
            bassRiff: [
                65.41, 65.41, 82.41, 65.41, 
                98.00, 65.41, 82.41, 73.42,
                65.41, 65.41, 98.00, 65.41,
                110.00, 65.41, 82.41, 73.42
            ],
            // 警报效果
            alarmFrequency: 880,
            alarmRate: 4,
            volume: 0.25
        }
    };
    
    playTrack(trackName) {
        if (this.currentTrack === trackName && this.isPlaying) return;
        
        this.stop();
        const config = this.tracks[trackName];
        if (!config) return;
        
        this.currentTrack = trackName;
        this.startLoop(config);
    }
    
    startLoop(config) {
        const beatDuration = 60 / config.bpm;
        const noteDuration = beatDuration / 2;  // 八分音符
        
        const scheduleLoop = () => {
            let step = 0;
            const maxSteps = 32;
            
            const scheduleNote = () => {
                if (!this.isPlaying) return;
                
                const freq = config.bassline 
                    ? config.bassline[step % config.bassline.length]
                    : config.melody[step % config.melody.length];
                
                this.playNote(freq, config.waveform, noteDuration * 0.9, config.volume);
                
                // 旋律层（延迟一个八分音符）
                if (config.leadMelody) {
                    setTimeout(() => {
                        const leadFreq = config.leadMelody[step % config.leadMelody.length];
                        this.playNote(leadFreq, 'triangle', noteDuration * 0.8, config.volume * 0.7);
                    }, noteDuration * 500);
                }
                
                step++;
                if (step >= maxSteps) step = 0;
                
                setTimeout(scheduleNote, noteDuration * 1000);
            };
            
            scheduleNote();
        };
        
        this.isPlaying = true;
        scheduleLoop();
    }
    
    playNote(frequency, type, duration, volume) {
        const osc = this.audioContext.createOscillator();
        const gain = this.audioContext.createGain();
        
        osc.type = type;
        osc.frequency.setValueAtTime(frequency, this.audioContext.currentTime);
        
        // ADSR包络（Attack-Decay-Sustain-Release）
        const now = this.audioContext.currentTime;
        gain.gain.setValueAtTime(0, now);
        gain.gain.linearRampToValueAtTime(volume, now + 0.02);  // Attack
        gain.gain.exponentialRampToValueAtTime(volume * 0.7, now + duration * 0.3);  // Decay
        gain.gain.setValueAtTime(volume * 0.7, now + duration * 0.7);  // Sustain
        gain.gain.linearRampToValueAtTime(0, now + duration);  // Release
        
        osc.connect(gain);
        gain.connect(this.masterGain);
        
        osc.start(now);
        osc.stop(now + duration + 0.05);
    }
    
    stop() {
        this.isPlaying = false;
        this.currentTrack = null;
    }
    
    // 淡入淡出切换
    fadeSwitch(targetTrack, fadeTime = 0.5) {
        const oldGain = this.masterGain.gain.value;
        const startContext = this.audioContext.currentTime;
        
        // 淡出当前BGM
        this.masterGain.gain.cancelScheduledValues(startContext);
        this.masterGain.gain.setValueAtTime(oldGain, startContext);
        this.masterGain.gain.linearRampToValueAtTime(0, startContext + fadeTime);
        
        // 淡入新BGM
        setTimeout(() => {
            this.stop();
            this.masterGain.gain.setValueAtTime(0, this.audioContext.currentTime);
            this.playTrack(targetTrack);
            this.masterGain.gain.linearRampToValueAtTime(
                this.bgmVolume, 
                this.audioContext.currentTime + fadeTime
            );
        }, fadeTime * 1000);
    }
}
```

#### 5.4.4 SFX 音效列表与实现

```
SFX音效清单（全部Web Audio API合成）：

┌──────────────────────────────────────────────────────────────┐
│ 分类一：武器音效                                              │
│ ─────────────────────────────                                │
│ SFX_SHOOT:        玩家射击声（短促激光声）                    │
│ SFX_SPREAD:       散射模式射击声（多音叠加）                  │
│ SFX_FOCUS:        聚焦模式射击声（高频尖锐）                  │
│ SFX_TRACK:        跟踪模式射击声（脉冲式）                    │
│ SFX_SWITCH:       切换武器声（电子咔哒）                      │
│ SFX_CHARGING:     蓄力音效（持续低频嗡鸣）                    │
│                                                              │
│ 分类二：敌人音效                                              │
│ ─────────────────────────────                                │
│ SFX_ENEMY_SHOOT:  敌机射击声（低沉点击）                      │
│ SFX_ENEMY_EXPL:   敌机爆炸（金属破碎+低频）                  │
│ SFX_BOSS_ENTER:   Boss入场（低频冲击+警报）                   │
│ SFX_BOSS_ATTACK:  Boss攻击音效（多层咆哮）                    │
│ SFX_BOSS_EXPL:    Boss爆炸（多层爆炸+回响）                   │
│ SFX_SUMMON:       召唤小兵声（诡异召唤音）                    │
│ SFX_WARNING:      激光预警线音效（急促警报）                   │
│                                                              │
│ 分类三：玩家反馈                                              │
│ ─────────────────────────────                                │
│ SFX_PLAYER_HIT:   玩家受击（闷响+故障声）                    │
│ SFX_PLAYER_EXPL:  玩家爆炸（金属爆炸）                       │
│ SFX_SHIELD_HIT:   护盾抵挡（清脆金属声）                     │
│ SFX_SHIELD_REGEN: 护盾恢复（柔和电子声）                     │
│ SFX_BOMB_ACT:     炸弹释放（能量爆发）                       │
│ SFX_INVINCIBLE:   无敌状态（持续光环音）                     │
│                                                              │
│ 分类四：界面/UI音效                                           │
│ ─────────────────────────────                                │
│ SFX_PICKUP:       拾取道具（清脆叮声）                       │
│ SFX_POWERUP:      升级音效（上升音阶）                       │
│ SFX_COMBO:        连击达成（欢呼声+打击音）                   │
│ SFX_GRAZE:        擦弹音效（短促电子音）                     │
│ SFX_CLICK:        按钮点击（软咔哒）                        │
│ SFX_PAUSE:        暂停/继续（系统提示音）                    │
│ SFX_STAGE_START:  关卡开始（号角声）                         │
│ SFX_STAGE_CLEAR:  关卡通关（胜利和弦）                       │
│ SFX_GAME_OVER:    游戏结束（故障音+低沉）                    │
│ SFX_ACHIEVEMENT:  成就解锁（金色叮声+音阶）                  │
└──────────────────────────────────────────────────────────────┘
```

#### 5.4.5 SFX 合成实现

```javascript
class SFXSynthesizer {
    audioContext;
    sfxVolume = 0.7;
    activeSounds = [];  // 并发音效管理
    maxConcurrent = 16;
    
    // SFX合成预设
    presets = {
        SFX_SHOOT: {
            create: (ctx, vol) => {
                const osc = ctx.createOscillator();
                const gain = ctx.createGain();
                const filter = ctx.createBiquadFilter();
                
                osc.type = 'square';
                osc.frequency.setValueAtTime(800, ctx.currentTime);
                osc.frequency.exponentialRampToValueAtTime(200, ctx.currentTime + 0.08);
                
                filter.type = 'lowpass';
                filter.frequency.value = 2000;
                
                gain.gain.setValueAtTime(vol * 0.3, ctx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.1);
                
                osc.connect(filter);
                filter.connect(gain);
                gain.connect(ctx.destination);
                
                osc.start();
                osc.stop(ctx.currentTime + 0.1);
            },
            duration: 0.1
        },
        
        SFX_EXPLOSION: {
            create: (ctx, vol) => {
                const bufferSize = ctx.sampleRate * 0.3;
                const buffer = ctx.createBuffer(1, bufferSize, ctx.sampleRate);
                const data = buffer.getChannelData(0);
                
                // 白噪声
                for (let i = 0; i < bufferSize; i++) {
                    data[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / bufferSize, 2);
                }
                
                const noise = ctx.createBufferSource();
                noise.buffer = buffer;
                
                const filter = ctx.createBiquadFilter();
                filter.type = 'lowpass';
                filter.frequency.setValueAtTime(1000, ctx.currentTime);
                filter.frequency.exponentialRampToValueAtTime(100, ctx.currentTime + 0.3);
                
                const gain = ctx.createGain();
                gain.gain.setValueAtTime(vol * 0.5, ctx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.3);
                
                noise.connect(filter);
                filter.connect(gain);
                gain.connect(ctx.destination);
                
                noise.start();
            },
            duration: 0.3
        },
        
        SFX_BOMB: {
            create: (ctx, vol) => {
                // 多层合成：低频冲击 + 高频爆裂 + 持续回响
                const now = ctx.currentTime;
                
                // 低频冲击
                const osc1 = ctx.createOscillator();
                const gain1 = ctx.createGain();
                osc1.type = 'sine';
                osc1.frequency.setValueAtTime(100, now);
                osc1.frequency.exponentialRampToValueAtTime(20, now + 0.5);
                gain1.gain.setValueAtTime(vol, now);
                gain1.gain.exponentialRampToValueAtTime(0.001, now + 0.5);
                osc1.connect(gain1);
                gain1.connect(ctx.destination);
                osc1.start(now);
                osc1.stop(now + 0.5);
                
                // 高频爆裂
                setTimeout(() => {
                    const bufferSize = ctx.sampleRate * 0.5;
                    const buffer = ctx.createBuffer(1, bufferSize, ctx.sampleRate);
                    const data = buffer.getChannelData(0);
                    for (let i = 0; i < bufferSize; i++) {
                        data[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / bufferSize, 1.5);
                    }
                    const noise = ctx.createBufferSource();
                    noise.buffer = buffer;
                    const filter = ctx.createBiquadFilter();
                    filter.type = 'highpass';
                    filter.frequency.value = 2000;
                    const gain = ctx.createGain();
                    gain.gain.setValueAtTime(vol * 0.8, ctx.currentTime);
                    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.8);
                    noise.connect(filter);
                    filter.connect(gain);
                    gain.connect(ctx.destination);
                    noise.start();
                }, 100);
            },
            duration: 1.5
        },
        
        SFX_POWERUP: {
            create: (ctx, vol) => {
                const notes = [523, 659, 784, 1047];  // C5-E5-G5-C6
                notes.forEach((freq, i) => {
                    const osc = ctx.createOscillator();
                    const gain = ctx.createGain();
                    osc.type = 'sine';
                    osc.frequency.value = freq;
                    
                    const startTime = ctx.currentTime + i * 0.05;
                    gain.gain.setValueAtTime(0, startTime);
                    gain.gain.linearRampToValueAtTime(vol * 0.4, startTime + 0.01);
                    gain.gain.exponentialRampToValueAtTime(0.001, startTime + 0.2);
                    
                    osc.connect(gain);
                    gain.connect(ctx.destination);
                    osc.start(startTime);
                    osc.stop(startTime + 0.25);
                });
            },
            duration: 0.5
        },
        
        SFX_COMBO: {
            create: (ctx, vol) => {
                // 欢呼式音效
                const osc = ctx.createOscillator();
                const gain = ctx.createGain();
                osc.type = 'triangle';
                osc.frequency.setValueAtTime(600, ctx.currentTime);
                osc.frequency.linearRampToValueAtTime(1200, ctx.currentTime + 0.15);
                osc.frequency.linearRampToValueAtTime(800, ctx.currentTime + 0.3);
                gain.gain.setValueAtTime(vol * 0.3, ctx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.4);
                osc.connect(gain);
                gain.connect(ctx.destination);
                osc.start();
                osc.stop(ctx.currentTime + 0.4);
            },
            duration: 0.4
        },
        
        SFX_WARNING: {
            create: (ctx, vol) => {
                // 急促警报
                let beepCount = 0;
                const beepInterval = setInterval(() => {
                    if (beepCount >= 3) {
                        clearInterval(beepInterval);
                        return;
                    }
                    const osc = ctx.createOscillator();
                    const gain = ctx.createGain();
                    osc.type = 'square';
                    osc.frequency.value = 880;
                    gain.gain.setValueAtTime(vol * 0.2, ctx.currentTime);
                    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.15);
                    osc.connect(gain);
                    gain.connect(ctx.destination);
                    osc.start();
                    osc.stop(ctx.currentTime + 0.15);
                    beepCount++;
                }, 200);
            },
            duration: 0.8
        }
    };
    
    play(sfxName) {
        // 并发控制
        if (this.activeSounds.length >= this.maxConcurrent) {
            // 丢弃最旧的音效
            this.activeSounds.shift();
        }
        
        const preset = this.presets[sfxName];
        if (!preset) return;
        
        try {
            preset.create(this.audioContext, this.sfxVolume);
            this.activeSounds.push({
                name: sfxName,
                time: Date.now(),
                duration: preset.duration
            });
        } catch (e) {
            console.warn(`SFX ${sfxName} play failed:`, e);
        }
        
        // 清理过期音效
        this.cleanup();
    }
    
    cleanup() {
        const now = Date.now();
        this.activeSounds = this.activeSounds.filter(
            s => now - s.time < s.duration * 1000
        );
    }
}
```

#### 5.4.6 音频管理器整合

```javascript
class AudioManager {
    // 单例模式
    static instance = null;
    
    audioContext = null;
    bgm = null;
    sfx = null;
    
    // 用户设置（持久化到localStorage）
    settings = {
        bgmVolume: 0.7,
        sfxVolume: 0.8,
        mute: false
    };
    
    // 初始化（必须在用户交互后调用）
    init() {
        if (this.audioContext) return;
        
        try {
            this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
            this.bgm = new BGMSynthesizer(this.audioContext);
            this.sfx = new SFXSynthesizer(this.audioContext);
            
            // 加载设置
            const saved = localStorage.getItem('game_audio_settings');
            if (saved) {
                this.settings = { ...this.settings, ...JSON.parse(saved) };
            }
        } catch (e) {
            console.warn('AudioContext init failed:', e);
        }
    }
    
    // 音量控制
    setBGMVolume(vol) {
        this.settings.bgmVolume = Math.max(0, Math.min(1, vol));
        this.bgm?.setVolume(this.settings.bgmVolume);
        this.saveSettings();
    }
    
    setSFXVolume(vol) {
        this.settings.sfxVolume = Math.max(0, Math.min(1, vol));
        this.sfx && (this.sfx.sfxVolume = this.settings.sfxVolume);
        this.saveSettings();
    }
    
    toggleMute() {
        this.settings.mute = !this.settings.mute;
        if (this.settings.mute) {
            this.bgm?.stop();
        } else {
            this.bgm?.resume();
        }
        this.saveSettings();
    }
    
    saveSettings() {
        localStorage.setItem('game_audio_settings', 
            JSON.stringify(this.settings));
    }
    
    // 便捷方法
    playBGM(track) { this.bgm?.playTrack(track); }
    fadeBGM(track) { this.bgm?.fadeSwitch(track); }
    playSFX(name) { 
        if (!this.settings.mute) {
            this.sfx?.play(name); 
        }
    }
}
```

#### 5.4.7 音频触发映射表

```
游戏事件 → 音频触发映射：

┌──────────────────────────────────────────────────────────────┐
│ 事件类型                │ BGM切换        │ SFX播放          │
├──────────────────────────────────────────────────────────────┤
│ 游戏启动                │ → menu        │ SFX_CLICK        │
│ 进入关卡                │ → battle      │ SFX_STAGE_START  │
│ Boss战开始              │ → boss        │ SFX_BOSS_ENTER   │
│ 击败Boss                │ → victory     │ SFX_BOSS_EXPL    │
│ 关卡通关                │ → menu        │ SFX_STAGE_CLEAR  │
│ 游戏结束                │ → gameover    │ SFX_GAME_OVER    │
│                                                              │
│ 玩家射击(散射)           │               │ SFX_SPREAD       │
│ 玩家射击(聚焦)           │               │ SFX_FOCUS        │
│ 玩家射击(跟踪)           │               │ SFX_TRACK        │
│ 切换武器                │               │ SFX_SWITCH       │
│ 拾取道具                │               │ SFX_PICKUP       │
│ 武器升级                │               │ SFX_POWERUP      │
│ 连击达成(10/30/50...)   │               │ SFX_COMBO        │
│ 擦弹成功                │               │ SFX_GRAZE        │
│ 玩家受击                │               │ SFX_PLAYER_HIT   │
│ 护盾抵挡                │               │ SFX_SHIELD_HIT   │
│ 炸弹释放                │               │ SFX_BOMB_ACT     │
│ 激光预警                │               │ SFX_WARNING      │
│ 敌人爆炸                │               │ SFX_ENEMY_EXPL   │
│ Boss攻击                │               │ SFX_BOSS_ATTACK  │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. 难度曲线与关卡设计

### 6.1 动态难度调整（DDA）

```javascript
class DifficultyManager {
    // 玩家表现追踪
    stats = {
        totalDeaths: 0,
        avgCombo: 0,
        maxCombo: 0,
        damageTaken: 0,
        timePerLevel: 0
    };
    
    // 难度等级 (0-10)
    currentLevel = 5;
    
    // 调整逻辑
    calculateDifficulty() {
        // 如果玩家死亡超过3次 → 降低难度
        if (this.stats.totalDeaths > 3) {
            this.currentLevel = Math.max(1, this.currentLevel - 1);
        }
        
        // 如果平均连击>100 → 提高难度
        if (this.stats.avgCombo > 100) {
            this.currentLevel = Math.min(10, this.currentLevel + 1);
        }
        
        // 应用难度系数
        return {
            enemyHp: 1 + (this.currentLevel - 5) * 0.1,
            enemyDamage: 1 + (this.currentLevel - 5) * 0.08,
            bulletSpeed: 1 + (this.currentLevel - 5) * 0.05,
            spawnRate: 1 + (this.currentLevel - 5) * 0.03
        };
    }
}
```

### 6.2 关卡配置表

```javascript
const LEVELS = [
    {
        stage: 1,
        name: '训练场',
        duration: 60,          // 秒
        bgTheme: 'space',
        enemyTypes: ['grunt', 'patrol'],
        spawnRate: 1.5,        // 敌/秒
        boss: 'void_commander',
        difficulty: 1.0
    },
    {
        stage: 2,
        name: '小行星带',
        duration: 75,
        bgTheme: 'asteroid',
        enemyTypes: ['grunt', 'patrol', 'sniper'],
        spawnRate: 2.0,
        boss: 'ice_guardian',
        difficulty: 1.3
    },
    {
        stage: 3,
        name: '太空站',
        duration: 90,
        bgTheme: 'station',
        enemyTypes: ['patrol', 'sniper', 'bomber'],
        spawnRate: 2.5,
        boss: 'iron_titan',
        difficulty: 1.6
    },
    {
        stage: 4,
        name: '虫洞',
        duration: 100,
        bgTheme: 'wormhole',
        enemyTypes: ['sniper', 'bomber', 'elite'],
        spawnRate: 3.0,
        boss: 'time_ripper',
        difficulty: 2.0
    },
    {
        stage: 5,
        name: '核心',
        duration: 120,
        bgTheme: 'core',
        enemyTypes: ['bomber', 'elite'],
        spawnRate: 3.5,
        boss: 'creator',
        difficulty: 2.5
    }
];
```

---

## 7. 性能优化要点

### 7.1 对象池模式

```javascript
class ObjectPool {
    // 子弹、粒子等频繁创建销毁的对象使用对象池
    
    constructor(factory, maxSize = 500) {
        this.factory = factory;
        this.pool = [];
        this.maxSize = maxSize;
    }
    
    acquire() {
        return this.pool.pop() || this.factory();
    }
    
    release(obj) {
        if (this.pool.length < this.maxSize) {
            this.pool.push(obj);
        } else {
            obj.destroy();
        }
    }
}

// 使用示例
const bulletPool = new ObjectPool(() => new Bullet(), 1000);
const particlePool = new ObjectPool(() => new Particle(), 2000);
```

### 7.2 碰撞检测优化

```javascript
class CollisionSystem {
    // 空间网格划分（QuadTree简化版）
    gridSize = 100;  // 100x100像素的格子
    grid = new Map();
    
    checkCollisions(bullets, enemies) {
        // 只检查相邻格子的碰撞，避免全局检测
        for (const bullet of bullets) {
            const cellKey = this.getCellKey(bullet.x, bullet.y);
            const neighbors = this.getNeighborCells(cellKey);
            
            for (const enemy of enemies) {
                if (neighbors.includes(this.getCellKey(enemy.x, enemy.y))) {
                    this.checkCollision(bullet, enemy);
                }
            }
        }
    }
}
```

### 7.3 渲染优化

```javascript
// 1. 离屏Canvas缓存
const bgCanvas = document.createElement('canvas');
const bgCtx = bgCanvas.getContext('2d');
// 只绘制一次背景，缓存使用

// 2. 批量渲染
ctx.save();
for (const bullet of bullets) {
    bullet.render(ctx);
}
ctx.restore();

// 3. 脏矩形更新（可选优化）
const dirtyRects = [];

// 4. 粒子数量控制
const maxParticles = 500;
if (activeParticles > maxParticles) {
    // 跳过最小粒子的渲染
    particles.sort((a, b) => a.size - b.size);
    particles.slice(0, maxParticles).forEach(p => p.render());
}
```

### 7.4 帧率监控

```javascript
class PerformanceMonitor {
    fps = 60;
    frameTime = 1000 / 60;  // 16.67ms
    slowFrameCount = 0;
    
    update(deltaTime) {
        this.fps = 1 / deltaTime;
        
        if (deltaTime > this.frameTime * 1.5) {
            this.slowFrameCount++;
            if (this.slowFrameCount > 10) {
                this.triggerOptimization();
            }
        } else {
            this.slowFrameCount = 0;
        }
    }
    
    triggerOptimization() {
        // 动态降级：减少粒子数量、降低特效质量
        this.particleSystem.reduceMaxCount(100);
        this.effectSystem.disableBloom();
    }
}
```

---

## 8. 开发MVP优先级清单

### Phase 1: 核心玩法（MVP）- 预估2天

```
优先级 P0:
[ ] 1. 项目结构搭建
    - index.html + style.css
    - js/ 模块化目录结构
    - 入口文件 main.js
    
[ ] 2. 游戏主循环
    - requestAnimationFrame循环
    - 场景状态机（菜单/游戏/暂停/结算）
    - deltaTime计算
    
[ ] 3. 玩家控制
    - WASD/方向键移动
    - 碰撞盒与碰撞检测
    - 受击、死亡、复活逻辑
    
[ ] 4. 基础射击
    - 自动射击
    - 子弹创建与移动
    - 子弹与敌人碰撞检测
    
[ ] 5. 敌人系统
    - 小兵生成与移动
    - 敌人基础弹幕
    - 死亡与分数
    
[ ] 6. 擦弹系统
    - 擦弹检测
    - 擦弹加分反馈
```

### Phase 2: 武器与弹幕（MVP+）- 预估2天

```
优先级 P1:
[ ] 7. 弹幕模式切换系统
    - 散射/聚焦/跟踪三种模式
    - 切换视觉反馈
    
[ ] 8. 武器升级系统
    - 5级武器等级
    - 升级道具掉落
    
[ ] 9. 更多弹幕Pattern
    - 散射、环形、螺旋
    - 瞄准、激光预警
    
[ ] 10. 连击与分数系统
    - 连击计时器
    - 倍率加成
    - 连击UI反馈
```

### Phase 3: Boss战 - 预估3天

```
优先级 P2:
[ ] 11. Boss框架
    - 多阶段系统
    - 阶段转换逻辑
    - Boss血条UI
    
[ ] 12. 第一个Boss（虚空指挥官）
    - Phase 1-4 行为
    - 专属弹幕Pattern
    - 出场/死亡动画
    
[ ] 13. 更多敌人类型
    - 巡逻兵、神射手、轰炸机
    - 每种敌人的弹幕模式
    
[ ] 14. 波次生成系统
    - WaveSystem
    - 波次配置
    - 动态难度
```

### Phase 4: 视觉打磨 - 预估3天

```
优先级 P3:
[ ] 15. 粒子系统
    - 爆炸、火花、拖尾
    - 能量粒子
    
[ ] 16. 背景系统
    - 多层视差滚动
    - 动态星云
    - 星空效果
    
[ ] 17. 屏幕效果
    - 受击闪屏
    - 炸弹白屏
    - 屏幕震动
    - 扫描线效果
    
[ ] 18. UI完善
    - HUD动态更新
    - 菜单动画
    - 结算画面
```

### Phase 5: 深度玩法 - 预估3天

```
优先级 P4:
[ ] 19. 道具系统
    - 护盾恢复、HP恢复
    - 炸弹补给
    - 僚机召唤
    
[ ] 20. 炸弹系统
    - 清屏效果
    - 无敌时间
    - 粒子特效
    
[ ] 21. 僚机系统
    - 僚机AI
    - 僚机弹幕
    
[ ] 22. 音频系统
    - BGM合成
    - 射击音效
    - 爆炸音效
```

### Phase 6: 扩展内容 - 预估3天

```
优先级 P5:
[ ] 23. 更多关卡
    - 5个关卡设计
    - 分支路线
    
[ ] 24. 更多Boss
    - 5个独特Boss
    - 不同主题弹幕
    
[ ] 25. 成就系统
    - 连击成就
    - 无伤通关
    - 高分记录
    
[ ] 26. 本地存储
    - 最高分记录
    - 通关进度
```

---

## 附录A：代码模板

### A.1 实体基类模板

```javascript
// js/entities/Entity.js
export class Entity {
    constructor(x, y, width, height) {
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
        this.vx = 0;
        this.vy = 0;
        this.alive = true;
        this.hp = 1;
        this.maxHp = 1;
        this.hitFlash = 0;
        this.type = 'entity';
    }
    
    getBounds() {
        return {
            x: this.x - this.width / 2,
            y: this.y - this.height / 2,
            w: this.width,
            h: this.height
        };
    }
    
    update(dt) {
        if (this.hitFlash > 0) this.hitFlash -= dt;
    }
    
    render(ctx) {
        // 由子类实现
    }
    
    onHit(damage) {
        this.hp -= damage;
        this.hitFlash = 0.1;
        if (this.hp <= 0) {
            this.die();
        }
    }
    
    die() {
        this.alive = false;
        this.onDestroy();
    }
    
    onDestroy() {
        // 由子类实现爆炸特效等
    }
}
```

### A.2 弹幕Pattern模板

```javascript
// js/patterns/BulletPattern.js
export class BulletPattern {
    constructor(config) {
        this.config = config;
        this.bulletsPerShot = config.bulletsPerShot || 1;
        this.bulletSpeed = config.bulletSpeed || 200;
        this.bulletDamage = config.bulletDamage || 10;
        this.fireRate = config.fireRate || 1.0;
        this.timer = 0;
    }
    
    update(dt, shooter, target) {
        this.timer -= dt;
        if (this.timer <= 0) {
            this.fire(shooter, target);
            this.timer = 1 / this.fireRate;
        }
    }
    
    fire(shooter, target) {
        // 由子类实现具体弹幕发射逻辑
    }
    
    createBullet(x, y, vx, vy) {
        return new Bullet(x, y, vx, vy, this.bulletDamage);
    }
}

// js/patterns/SpreadPattern.js
export class SpreadPattern extends BulletPattern {
    constructor(config) {
        super(config);
        this.spreadAngle = config.spreadAngle || Math.PI / 4;
    }
    
    fire(shooter, target) {
        const baseAngle = Math.atan2(
            target.y - shooter.y,
            target.x - shooter.x
        );
        
        const bullets = [];
        for (let i = 0; i < this.bulletsPerShot; i++) {
            const t = this.bulletsPerShot === 1 
                ? 0.5 
                : i / (this.bulletsPerShot - 1);
            const angle = baseAngle - this.spreadAngle / 2 
                         + this.spreadAngle * t;
            
            bullets.push(this.createBullet(
                shooter.x,
                shooter.y,
                Math.cos(angle) * this.bulletSpeed,
                Math.sin(angle) * this.bulletSpeed
            ));
        }
        
        return bullets;
    }
}
```

### A.3 游戏配置模板

```javascript
// js/config/gameConfig.js
export const GameConfig = {
    canvas: {
        width: 480,
        height: 800,
        backgroundColor: '#0A0A1A'
    },
    
    player: {
        width: 40,
        height: 50,
        speed: 300,
        slowSpeed: 150,
        maxHp: 100,
        maxShield: 100,
        shieldRegenRate: 5,  // 每秒
        invincibleTime: 1.5  // 复活无敌时间
    },
    
    bomb: {
        maxEnergy: 100,
        energyPerKill: 2,
        energyPerPickup: 50,
        duration: 3.0,      // 爆炸持续时间
        damage: 2000,       // 对敌人伤害
        invincibility: 3.0  // 玩家无敌时间
    },
    
    combo: {
        timeout: 2.0,        // 连击超时
        milestones: [
            { count: 10, multiplier: 1.5 },
            { count: 30, multiplier: 2.0 },
            { count: 50, multiplier: 3.0 },
            { count: 100, multiplier: 5.0 },
            { count: 200, multiplier: 8.0 },
            { count: 500, multiplier: 15.0 }
        ]
    }
};
```

---

> **文档维护说明**：  
> 本文档为活文档，开发过程中需持续更新。建议每完成一个Phase后，补充实现细节和调整说明。  
>  
> **Solo Builder 使用指南**：  
> 1. 按Phase顺序实现，每个Phase完成后测试  
> 2. 弹幕Pattern可作为独立模块开发，便于调试  
> 3. Boss战建议在Phase 3后集中实现  
> 4. 视觉特效在基础玩法完成后叠加

---
**END OF DOCUMENT**
