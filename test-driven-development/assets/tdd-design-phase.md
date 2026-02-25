# TDD 设计阶段学习参考

---

## 一、核心理念转变

TDD 设计阶段最根本的思维转变只有一句话：

> **传统设计**：先想"怎么实现"，再想"怎么验证"
> **TDD 设计**：先想"怎么测试"，再反推"怎么实现"

这个顺序的颠倒，会让设计的每一个决策都围绕"可测试性"展开。

---

## 二、与传统设计的核心差异

### 2.1 接口设计的出发点不同

**传统设计**：从实现角度定义接口，接口反映内部结构。

```cpp
// 暴露了内部实现细节
class PromotionManager {
    void LoadConfigFromRegistry();
    void ParseCloudControlJson(const std::string& json);
    bool CheckInstalledApps(std::vector<std::string>& list);
};
```

**TDD 设计**：从测试（即使用者）角度定义接口，接口反映业务行为。

```cpp
// 只暴露业务意图，内部怎么实现不关心
class IAvoidanceChecker {
    virtual bool ShouldPromoteClient() = 0;
    virtual bool ShouldPromotePlugin() = 0;
};
```

**判断标准**：写测试时你希望怎么调用它？接口就应该长那样。

---

### 2.2 依赖处理方式不同

**传统设计**：依赖是实现细节，设计阶段不重点考虑。

**TDD 设计**：**依赖是设计的核心议题**。每一个外部依赖都必须在设计阶段识别并决策。

识别外部依赖的思路：

```
凡是"你的代码控制不了的东西"，都是外部依赖：
  - 操作系统 API（注册表、文件系统、进程列表）
  - 网络请求（云控下发）
  - 时间（time(), GetTickCount()）
  - 浏览器状态（已安装插件列表）
  - 用户界面（弹窗、托盘）
  - 随机数
```

每个外部依赖的设计决策：

| 依赖类型 | 处理方式 | 说明 |
|---------|---------|------|
| OS/系统 API | 抽象成接口 + Mock | 无法在单元测试中直接控制 |
| 网络请求 | 抽象成接口 + Stub | 返回预设数据，不走真实网络 |
| 系统时间 | 注入时钟接口 | 边界条件测试必须能固定时间 |
| 文件/本地存储 | 可用 Fake 替代 | 内存实现替代真实文件读写 |
| 随机数 | 注入随机数接口 | 让测试结果可预期 |

> **时间依赖是新手最容易忽视的**。任何涉及"X天内/X天后"的业务逻辑，
> 必须设计时钟接口，否则边界条件无法测试。

---

### 2.3 模块拆分的粒度标准不同

**传统设计**：按功能模块拆分。

**TDD 设计**：按**可独立测试**的职责拆分。

判断一个类是否需要继续拆分的标准：

> 如果这个类的单元测试需要 Mock 超过 3 个依赖，说明职责过多，需要拆分。

---

### 2.4 设计完成的标志不同

| | 传统设计 | TDD 设计 |
|---|---|---|
| 覆盖所有需求 | ✓ | ✓ |
| 逻辑自洽 | ✓ | ✓ |
| 每条 AC 有明确测试入口 | ✗ | ✓ |
| 所有外部依赖已识别并有 Mock 方案 | ✗ | ✓ |
| 接口可在不启动完整程序的情况下测试 | ✗ | ✓ |

---

## 三、TDD 设计的具体注意事项

### 3.1 可测试性优先于"优雅"

遇到"这样设计更优雅，但不好测"vs"这样设计好测，但稍显冗余"时，**选择好测的**。

可测试性是一等公民。一个无法被测试的"优雅"设计，在 TDD 中是失败的设计。

---

### 3.2 避免静态方法和单例

单例和静态方法无法被替换，是 TDD 的天敌。

```cpp
// 危险：无法 Mock
bool PromotionModule::CheckAvoidance() {
    auto& cfg = CloudConfig::GetInstance(); // 单例，测试无法替换
    auto now = time(nullptr);               // 静态调用，测试无法控制时间
}

// 安全：通过构造函数注入依赖
class AvoidanceChecker {
public:
    AvoidanceChecker(ICloudConfig* config, IClock* clock)
        : config_(config), clock_(clock) {}
private:
    ICloudConfig* config_;
    IClock* clock_;
};
```

---

### 3.3 先写接口，再写实现

设计阶段的产出是**接口契约**，不是实现代码。

实现是在 Red-Green 循环中被测试驱动出来的，不是设计出来的。

设计阶段产出物：
- 接口定义（纯虚类）
- 数据结构定义
- 各模块职责说明
- AC → 类/方法 的映射表

---

### 3.4 AC 到测试入口的映射要显式

设计文档中应该能明确看出每条 AC 对应哪个类的哪个方法，以及测试文件在哪里。

示例：

| AC | 负责类 | 测试方法名 |
|----|--------|-----------|
| AC-03-2 | `ClientAvoidanceChecker` | `ShouldNotPromote_WhenUninstalledLessThan30Days` |
| AC-04-1 | `PluginAvoidanceChecker` | `ShouldNotPromote_WhenAvoidancePluginInstalled` |
| AC-08-1 | `PromotionSelector` | `ShouldSelectBoth_WhenBothClientAndPluginPass` |

---

### 3.5 识别"接缝"（Seam）

接缝是指代码中**可以在不修改源代码的情况下改变行为**的点。

TDD 设计的本质，就是**刻意设计接缝**，让测试可以从这些点介入，替换真实依赖为测试替身。

C++ 中常见的接缝类型：

| 接缝类型 | 实现方式 | 示例 |
|---------|---------|------|
| 对象接缝 | 虚函数 + 依赖注入 | 通过构造函数传入 `IClock*` |
| 链接接缝 | 替换链接目标 | 测试时链接 Mock 库 |
| 预处理接缝 | 宏替换 | `#define NOW MockClock::Now()` |

**优先使用对象接缝**（虚函数 + 依赖注入），最清晰、最安全。

---

## 四、测试替身的分类

设计阶段需要决定每个依赖用哪种测试替身，概念不清会导致测试设计混乱。

| 类型 | 作用 | 使用场景 |
|------|------|---------|
| **Dummy** | 占位，从不使用 | 满足参数列表，但测试中不关心它 |
| **Stub** | 返回预设数据 | 控制输入，让代码走特定分支 |
| **Fake** | 有简单实现的轻量替代 | 内存版数据库、内存版文件系统 |
| **Spy** | 记录调用情况，可事后验证 | 验证某方法是否被调用、调用了几次 |
| **Mock** | 预先设定期望，自动验证 | 严格验证交互行为 |

> 常见误区：把所有测试替身都叫"Mock"。
> Stub 控制**状态**，Mock 验证**交互**，用途不同，不要混淆。

---

## 五、两种 TDD 推进方向

### 5.1 Outside-In（由外向内）

从最外层的接口（用户/调用方视角）开始设计和测试，逐步向内实现。

```
AC → 高层接口测试（先写，跑红）
  → 设计内部组件接口
    → 内部组件测试（跑红）
      → 实现内部组件（跑绿）
    → 实现高层接口（跑绿）
```

**优点**：始终以使用者视角驱动设计，不会过度设计内部细节。
**适合**：需求清晰、接口边界明确的场景。

### 5.2 Inside-Out（由内向外）

从最底层、最稳定的核心逻辑开始，逐步向外构建。

```
核心业务逻辑 → 测试并实现
  → 组合成更高层的组件 → 测试并实现
    → 组装成完整功能
```

**优点**：底层基础扎实，不需要大量 Mock。
**适合**：核心算法复杂、底层逻辑不依赖外部的场景。

> 本项目（电商推广模块）规避判断逻辑相对独立，推荐 **Inside-Out** 起步，
> 先把规避判断、安装记录等核心逻辑测试完，再向外组装弹窗和安装流程。

---

## 六、Humble Object 模式

### 6.1 起源

Humble Object 由 Gerard Meszaros 在《xUnit Test Patterns》中提出，专门解决一个痛点：

> **有些代码天生难以测试**，不是因为逻辑复杂，而是因为它与"不可控的环境"紧密耦合。

---

### 6.2 核心思路

把一个"难以测试的对象"一分为二：

```
        原始对象（难以测试）
              │
     ┌────────┴────────┐
     ▼                 ▼
Humble Object       逻辑对象
（谦逊对象）        （可测试）
  薄壳，无逻辑      所有业务逻辑
  不需要测试        充分测试
```

**谦逊对象**只做一件事：在"不可控环境"和"业务逻辑"之间做最简单的搬运，自身不含任何判断。

---

### 6.3 什么叫"难以测试的环境"

| 环境 | 原因 |
|------|------|
| UI 窗口、弹窗 | 需要消息循环，自动化测试难以驱动 |
| 系统托盘 | 依赖 Windows Shell，测试环境无显示器 |
| 网络 IO | 耗时、不稳定、需要真实服务器 |
| 硬件设备 | 测试机不一定有设备 |
| 多线程定时器 | 时序不可控 |

---

### 6.4 重构示例

**重构前**：弹窗逻辑和业务逻辑混在一起，无法单元测试

```cpp
// 这个类无法单元测试：
// - 测试时无法创建真实窗口
// - 无法模拟用户点击
// - 无法验证"通过规避才显示"这条逻辑
class PromotionDialog {
public:
    void Show() {
        // 业务逻辑混在 UI 代码里
        if (avoidanceChecker_.ShouldPromote()) {
            if (cloudConfig_.GetTimeout() > 0) {
                StartTimer(cloudConfig_.GetTimeout());
            }
            CreateWindow(...);
        }
    }

    void OnCloseButtonClicked() {
        if (cloudConfig_.IsCloseButtonPromotion()) {
            installer_.Install();  // 业务判断混在事件处理里
        }
        DestroyWindow(...);
    }
};
```

**重构后**：用 Humble Object 模式拆分

```cpp
// ① Humble Object：谦逊对象，薄壳，不测
// 只负责"创建窗口、传递事件"，不含任何 if/else
class PromotionDialog {
public:
    explicit PromotionDialog(PromotionPresenter* presenter)
        : presenter_(presenter) {}

    void Show() {
        CreateWindow(...);           // 纯 UI 操作
        auto timeout = presenter_->GetTimeoutMs();
        if (timeout > 0) StartTimer(timeout);
        SetContent(presenter_->GetTitle(), presenter_->GetImage());
    }

    // 事件发生时，原样转发给 Presenter，自己不做判断
    void OnCloseButtonClicked()   { presenter_->OnClose();   }
    void OnConfirmButtonClicked() { presenter_->OnConfirm(); }
    void OnTimeout()              { presenter_->OnTimeout(); }

private:
    PromotionPresenter* presenter_;
};


// ② 逻辑对象：Presenter，所有业务逻辑在这里，完全可测
class PromotionPresenter {
public:
    PromotionPresenter(IAvoidanceChecker* checker,
                       ICloudConfig*      config,
                       IInstaller*        installer)
        : checker_(checker), config_(config), installer_(installer) {}

    int GetTimeoutMs() {
        return config_->GetTimeoutMs();  // 可测：验证读取云控超时时长
    }

    void OnClose() {
        if (config_->IsCloseButtonPromotion()) {
            installer_->Install();       // 可测：验证云控开关决定是否安装
        }
    }

    void OnConfirm() {
        installer_->Install();           // 可测：点击确认直接安装
    }

    void OnTimeout() {
        if (config_->IsTimeoutPromotion()) {
            installer_->Install();       // 可测：验证超时云控开关
        }
    }
};
```

测试只需针对 `PromotionPresenter`，完全不需要创建窗口：

```cpp
TEST(PresenterTest, OnClose_ShouldInstall_WhenCloudConfigEnabled) {
    MockCloudConfig config;
    MockInstaller   installer;
    EXPECT_CALL(config, IsCloseButtonPromotion()).WillOnce(Return(true));
    EXPECT_CALL(installer, Install()).Times(1);

    PromotionPresenter presenter(nullptr, &config, &installer);
    presenter.OnClose();
}

TEST(PresenterTest, OnClose_ShouldNotInstall_WhenCloudConfigDisabled) {
    MockCloudConfig config;
    MockInstaller   installer;
    EXPECT_CALL(config, IsCloseButtonPromotion()).WillOnce(Return(false));
    EXPECT_CALL(installer, Install()).Times(0);

    PromotionPresenter presenter(nullptr, &config, &installer);
    presenter.OnClose();
}
```

---

### 6.5 本项目中的应用位置

| 组件 | Humble Object | 逻辑对象 |
|------|--------------|---------|
| 推广弹窗 | `PromotionDialog`（Win32 窗口操作） | `PromotionPresenter`（按钮/超时逻辑） |
| 系统托盘 | `TrayIcon`（Shell_NotifyIcon 调用） | `TrayPresenter`（进度模拟逻辑） |
| 注册表访问 | `RegistryReader`（真实注册表读写） | 业务层通过 `IRegistryReader` 接口访问 |
| 云控网络请求 | `CloudConfigFetcher`（真实 HTTP） | 业务层通过 `ICloudConfig` 接口访问 |

---

### 6.6 一句话记住它

> **Humble Object 的本质：把"我没法控制的环境"和"我的业务逻辑"之间砌一道墙。墙的这边全是可测试的纯逻辑，墙的那边是薄薄的一层胶水，薄到没有逻辑可言。**

---

## 七、CQS 原则与可测试性

**命令查询分离（Command Query Separation）**：

- **Query（查询）**：返回数据，不改变状态。测试时只需验证返回值。
- **Command（命令）**：改变状态，不返回数据。测试时验证状态变化或交互。

设计接口时遵守 CQS，测试会变得更简单清晰：

```cpp
// 好：Query 和 Command 分离
bool ShouldPromoteClient();           // Query：只查询，可直接断言返回值
void RecordPluginInstallation(...);   // Command：只写入，验证副作用

// 差：混合了查询和命令，测试难以聚焦
bool CheckAndRecord(...);             // 又查又写，测试需要同时关注两件事
```

---

## 八、设计阶段检查清单

在进入编写测试阶段之前，用以下清单验证设计是否完备：

- [ ] 所有外部依赖已识别，每个都有对应的抽象接口
- [ ] 系统时间依赖已抽象为可注入的时钟接口
- [ ] 没有无法替换的单例或静态方法依赖
- [ ] 每条 AC 都能找到对应的类和方法作为测试入口
- [ ] 每个类的测试所需 Mock 数量不超过 3 个
- [ ] UI/系统调用等难测代码已用 Humble Object 模式隔离
- [ ] 接口设计反映业务意图，而非实现细节
- [ ] 数据结构已定义，AC 中的字段（插件ID、安装时间等）都有明确对应
