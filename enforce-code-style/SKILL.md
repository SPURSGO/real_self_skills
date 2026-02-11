---
name: enforce-code-style
description: 在实际编写任何代码时都需要使用，而且是必须遵守的基本准则
---

# 代码风格-编写代码的基本准则

## C++ 代码规范
- 严格遵守 Google C++ Style Guide (https://google.github.io/styleguide/cppguide.html)
- 关键要点：
  - 使用 2 空格缩进，不使用 tab
  - 文件名全小写，可以包含下划线或连字符
  - 类型名使用 PascalCase (如 MyClass)
  - 变量名使用 snake_case (如 table_name)
  - 常量名使用 kCamelCase (如 kDaysInAWeek)
  - 函数名使用 PascalCase (如 AddTableEntry)
  - 命名空间名全小写
  - 头文件使用 #ifndef 保护，格式：<PROJECT>_<PATH>_<FILE>_H_
  - 使用 // 进行单行注释，/* */ 进行多行注释
  - 函数声明和定义的参数要么全在一行，要么每个参数一行
  - 优先使用前置递增/递减 (++i, --i)
  - 使用 nullptr 而不是 NULL 或 0
  - 使用 using 而不是 typedef
  - 头文件包含顺序：相关头文件、C 系统头文件、C++ 标准库头文件、其他库头文件、本项目头文件