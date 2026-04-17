# 阶段 3：产物生成

开发代码完成后，根据任务类型自动生成对应产物。**每类产物的触发条件见下表，无需用户手动要求。**

| 产物 | 触发条件 | 必须生成 |
|-----|---------|---------|
| 单元测试 | 始终 | ✅ |
| 集成测试 | 有新接口 | ✅ |
| 接口文档 | 有新增/修改接口 | ✅ |
| 功能说明文档 | 新功能类任务 | ✅ |
| DB Change Log | 涉及表结构变更 | ✅ |
| 变更影响图 | 始终（在开发阶段已生成）| ✅ |

---

## 3.1 单元测试

### 测试框架识别

| 项目特征 | 测试框架 |
|---------|---------|
| `jest.config.*` / `"jest"` in package.json | Jest |
| `pytest.ini` / `conftest.py` | pytest |
| `*_test.go` 文件 | Go testing |
| `src/test/java` + `pom.xml` | JUnit 5 |
| `*spec.rb` | RSpec |
| `tests/Feature` + Laravel | PHPUnit |

### 测试用例生成规则

每个核心方法至少覆盖：

```
1. 正常路径（Happy Path）— 输入合法，期望返回正确结果
2. 边界条件 — 空列表、零值、最大值、最小值
3. 异常路径 — 数据不存在、权限不足、重复提交
4. 参数校验 — 必填字段缺失、格式错误、类型错误
```

### 测试代码风格要求

- 测试描述语言与项目一致（中文/英文）
- Mock 方式与项目一致（jest.mock / MagicMock / Mockito）
- 断言库与项目一致（expect / assert / should）
- 测试文件位置与项目一致（`__tests__/` / `*.spec.ts` / `*_test.py`）

### Jest 测试模板（参考）

```typescript
describe('OrderService', () => {
  let service: OrderService;
  let userService: jest.Mocked<UserService>;
  let orderRepo: jest.Mocked<OrderRepository>;

  beforeEach(async () => {
    // 参考项目现有测试的 module 初始化方式
  });

  describe('createOrder', () => {
    it('正常创建订单', async () => {
      // Arrange
      // Act
      // Assert
    });

    it('用户不存在时抛出异常', async () => {
      // ...
    });

    it('并发创建时不产生重复订单', async () => {
      // ...
    });
  });
});
```

---

## 3.2 接口文档

### 触发条件
有新增或修改的 HTTP 接口时自动生成。

### 优先格式
1. 如项目已有 OpenAPI/Swagger 集成（`@ApiProperty` / `@swagger` / `springdoc`） → 生成符合 OpenAPI 3.0 的注解代码
2. 如项目用 Postman → 生成 Postman Collection JSON
3. 其他情况 → Markdown 格式

### Markdown 接口文档模板

```markdown
## 接口名称：创建订单

**路径：** `POST /api/v1/orders`
**认证：** Bearer Token（JWT）
**权限：** 需登录用户

### 请求参数

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|-------|-----|-----|-----|-----|
| productId | string | 是 | 商品ID（UUID格式）| "abc-123" |
| quantity | number | 是 | 购买数量（1～99）| 2 |
| remark | string | 否 | 备注信息（最长200字）| "尽快发货" |

### 响应示例

**成功（200）**
```json
{
  "code": 0,
  "data": {
    "orderId": "xyz-456",
    "status": "pending",
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "message": "success"
}
```

**失败示例**
```json
{ "code": 40401, "data": null, "message": "商品不存在" }
{ "code": 40301, "data": null, "message": "无权限" }
{ "code": 42201, "data": null, "message": "数量超出库存" }
```

### 错误码

| 错误码 | 含义 |
|-------|-----|
| 40401 | 商品不存在 |
| 40301 | 无权限 |
| 42201 | 库存不足 |
```

---

## 3.3 功能说明文档

### 触发条件
新功能类任务（非 BUG 修复）时自动生成。

### 模板

```markdown
# 功能说明：[功能名称]

## 功能概述
[一句话描述这个功能是什么]

## 业务背景
[为什么需要这个功能，解决什么问题]

## 功能范围
- ✅ 包含：[本次实现的内容]
- ❌ 不包含：[明确不在范围内的内容，避免歧义]

## 核心流程
[用文字或流程图描述主要业务流程]

1. 用户操作 XXX
2. 系统校验 YYY
3. 执行 ZZZ
4. 返回结果

## 数据流转
[如有数据库变更，说明数据如何流转和存储]

## 注意事项
- [边界情况说明]
- [性能注意点]
- [与其他模块的交互说明]

## 验收标准
- [ ] [可量化的验收条件1]
- [ ] [可量化的验收条件2]

## 变更记录
| 日期 | 版本 | 说明 | 作者 |
|-----|-----|-----|-----|
| [今天日期] | 1.0.0 | 初始版本 | — |
```

---

## 3.4 产物汇总输出

所有产物生成完毕后，以如下格式汇总告知用户：

```
## 产物清单

本次开发共生成以下产物：

### 代码文件
- [新增] src/modules/order/order.controller.ts
- [新增] src/modules/order/order.service.ts
- [新增] src/modules/order/order.repository.ts
- [修改] src/modules/order/order.module.ts

### 数据库
- [新增] src/migrations/1700000000000-AddOrderTable.ts
- DB Change Log（见下方）

### 测试
- [新增] src/modules/order/__tests__/order.service.spec.ts（覆盖 8 个用例）

### 文档
- 接口文档：POST /api/v1/orders（Markdown）
- 功能说明：订单创建功能

---
进入 Code Review 阶段...
```
