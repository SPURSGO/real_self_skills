# TDD 工作流程实践指南

## 🎯 什么是 TDD？

**测试驱动开发（Test-Driven Development, TDD）** 是一种软件开发方法，核心理念是**先写测试，后写代码**。

### TDD 的三大定律

1. **不写测试就不写产品代码**
2. **只写刚好能够失败的测试**
3. **只写刚好能让测试通过的产品代码**

## 🔄 红-绿-重构循环

TDD 的核心是 **Red-Green-Refactor** 循环：

```
   🔴 RED              🟢 GREEN           🔵 REFACTOR
(写失败的测试)   →  (让测试通过)   →   (优化代码)
      ↑                                        ↓
      ←────────────────────────────────────────
```

### 🔴 RED - 写失败的测试

**目标**：编写一个会失败的测试

```cpp
// tests/calculator_test.cpp
TEST_F(CalculatorTest, Power_PositiveNumbers_ReturnsCorrectResult) {
    EXPECT_EQ(calc->power(2, 3), 8);   // 2^3 = 8
    EXPECT_EQ(calc->power(5, 2), 25);  // 5^2 = 25
}
```

**运行测试**：确认测试失败（因为 `power` 方法还不存在）

```powershell
.\build\tests\Debug\AllTests.exe --gtest_filter=*Power*
```

### 🟢 GREEN - 让测试通过

**目标**：编写最简单的代码让测试通过

#### Step 1: 在头文件中声明方法

```cpp
// include/calculator.h
class Calculator {
public:
    // ... 其他方法 ...

    /**
     * @brief 计算乘方
     * @param base 底数
     * @param exponent 指数（必须非负）
     * @return base的exponent次方
     */
    int power(int base, int exponent) const;
};
```

#### Step 2: 实现方法

```cpp
// src/calculator.cpp
int Calculator::power(int base, int exponent) const {
    if (exponent == 0) return 1;

    int result = 1;
    for (int i = 0; i < exponent; ++i) {
        result *= base;
    }
    return result;
}
```

#### Step 3: 重新构建并运行测试

```powershell
# 构建项目
cmake --build build --config Debug

# 运行测试
.\build\tests\Debug\AllTests.exe --gtest_filter=*Power*
```

**结果**：测试应该通过 ✅

### 🔵 REFACTOR - 重构代码

**目标**：在保持测试通过的前提下，优化代码

可能的重构：
- 提取重复代码
- 改进命名
- 优化算法
- 添加注释

**重要**：每次重构后都要运行测试，确保没有破坏功能！

```powershell
.\build\tests\Debug\AllTests.exe
```

## 📝 完整 TDD 实战示例

假设我们要添加一个 `factorial` 方法来计算阶乘。

### Step 1: 🔴 写失败的测试

```cpp
// tests/calculator_test.cpp

// ==================== 阶乘测试 ====================

TEST_F(CalculatorTest, Factorial_OfZero_ReturnsOne) {
    EXPECT_EQ(calc->factorial(0), 1);  // 0! = 1
}

TEST_F(CalculatorTest, Factorial_OfPositiveNumbers_ReturnsCorrectResult) {
    EXPECT_EQ(calc->factorial(1), 1);   // 1! = 1
    EXPECT_EQ(calc->factorial(5), 120); // 5! = 120
    EXPECT_EQ(calc->factorial(3), 6);   // 3! = 6
}

TEST_F(CalculatorTest, Factorial_OfNegativeNumber_ThrowsException) {
    EXPECT_THROW(calc->factorial(-1), std::invalid_argument);
}
```

**运行测试**（会失败，因为方法不存在）：

```powershell
cmake --build build --config Debug
.\build\tests\Debug\AllTests.exe --gtest_filter=*Factorial*
```

### Step 2: 🟢 让测试通过

#### 在头文件中添加声明

```cpp
// include/calculator.h
/**
 * @brief 计算阶乘
 * @param n 非负整数
 * @return n的阶乘
 * @throws std::invalid_argument 当n为负数时抛出异常
 */
int factorial(int n) const;
```

#### 实现方法

```cpp
// src/calculator.cpp
int Calculator::factorial(int n) const {
    if (n < 0) {
        throw std::invalid_argument("阶乘的参数必须是非负整数");
    }

    if (n == 0 || n == 1) {
        return 1;
    }

    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}
```

#### 重新构建并运行测试

```powershell
cmake --build build --config Debug
.\build\tests\Debug\AllTests.exe --gtest_filter=*Factorial*
```

**结果**：所有测试应该通过 ✅

### Step 3: 🔵 重构（可选）

代码已经很清晰了，可以考虑：
- 添加更多边界测试（如大数阶乘溢出处理）
- 使用递归实现（如果合适）

## 🎮 VSCode 中的 TDD 工作流

### 方法一：使用快捷键（推荐）

1. **编写测试** → 保存文件 (`Ctrl+S`)
2. **运行构建** → 按 `Ctrl+Shift+B`
3. **查看测试结果** → 在终端中查看输出
4. **如果失败** → 编写代码 → 重复步骤1-3
5. **如果通过** → 进入重构阶段

### 方法二：使用任务

1. 按 `Ctrl+Shift+P` 打开命令面板
2. 输入 `Tasks: Run Task`
3. 选择 `Build and Run Tests`

### 方法三：使用调试器

1. 在测试代码中设置断点（`F9`）
2. 按 `F5` 启动调试
3. 单步调试（`F10` 跳过，`F11` 进入）

## 📊 测试覆盖的黄金法则

### FIRST 原则

好的单元测试应该遵循 **FIRST** 原则：

- **F**ast（快速）：测试应该快速运行
- **I**ndependent（独立）：测试之间不应相互依赖
- **R**epeatable（可重复）：测试应该在任何环境下都能重复运行
- **S**elf-Validating（自验证）：测试应该有明确的通过/失败结果
- **T**imely（及时）：测试应该及时编写（在代码之前）

### 测试命名三段式

```cpp
TEST_F(TestFixture, MethodName_Scenario_ExpectedBehavior)
```

**示例**：

```cpp
TEST_F(CalculatorTest, Divide_ByZero_ThrowsException)
TEST_F(CalculatorTest, Add_PositiveNumbers_ReturnsSum)
TEST_F(CalculatorTest, Factorial_OfZero_ReturnsOne)
```

### AAA 测试模式

每个测试应该遵循 **Arrange-Act-Assert** 模式：

```cpp
TEST_F(CalculatorTest, ExampleTest) {
    // Arrange（准备）- 设置测试数据
    int a = 10;
    int b = 5;

    // Act（执行）- 执行要测试的操作
    int result = calc->subtract(a, b);

    // Assert（断言）- 验证结果
    EXPECT_EQ(result, 5);
}
```

## 🚀 实用技巧

### 1. 使用测试过滤器

```powershell
# 运行特定测试套件
.\build\tests\Debug\AllTests.exe --gtest_filter=CalculatorTest.*

# 运行特定测试
.\build\tests\Debug\AllTests.exe --gtest_filter=CalculatorTest.AddPositiveNumbers

# 排除特定测试
.\build\tests\Debug\AllTests.exe --gtest_filter=-CalculatorTest.Slow*

# 运行多个测试
.\build\tests\Debug\AllTests.exe --gtest_filter=*Add*:*Subtract*
```

### 2. 显示详细输出

```powershell
# 显示所有输出
.\build\tests\Debug\AllTests.exe --gtest_verbose

# 失败时显示完整信息
.\build\tests\Debug\AllTests.exe --gtest_print_time=1
```

### 3. 重复运行测试（用于发现不稳定的测试）

```powershell
.\build\tests\Debug\AllTests.exe --gtest_repeat=10
```

### 4. 随机顺序运行测试

```powershell
.\build\tests\Debug\AllTests.exe --gtest_shuffle
```

## 🎯 TDD 最佳实践

### ✅ 应该做的

1. **先写测试，再写代码**
2. **每次只做一件小事**
3. **频繁运行测试**
4. **测试要简单明了**
5. **测试边界条件**
6. **测试异常情况**
7. **保持测试独立**
8. **使用有意义的测试名称**

### ❌ 不应该做的

1. **不要跳过测试直接写代码**
2. **不要写太大的测试**
3. **不要让测试相互依赖**
4. **不要忽略失败的测试**
5. **不要过度测试私有方法**
6. **不要在测试中使用随机数据（除非是专门测试随机性）**
7. **不要让测试依赖外部资源（数据库、网络等）**

## 📈 TDD 的好处

1. **更少的 Bug**：在编写代码之前就考虑边界条件
2. **更好的设计**：TDD 促使你编写可测试的代码
3. **即时反馈**：快速知道代码是否正确
4. **重构信心**：有测试保护，放心重构
5. **文档作用**：测试本身就是最好的使用示例
6. **减少调试时间**：问题更容易定位

---

记住：**TDD 不仅仅是测试，更是一种设计方法！**

通过先写测试，你会被迫思考：
- 这个功能应该如何使用？
- 接口应该如何设计？
- 边界情况有哪些？

