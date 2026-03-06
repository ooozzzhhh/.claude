# Java 异常处理与报错规范

供应链计划系统（DFP/SOP/MPS/MRP/MDS/AMS 等）所有 Java 模块**必须遵循**本规范，覆盖前端交互、业务逻辑、数据访问全层级。使用平台统一异常类 `BusinessException`。

## 1. 抛出层级

业务异常**仅在 Service 层抛出**，Controller/DAO 不得直接抛 BusinessException 或 BaseResponse.error。DAO 异常必须在 Service 层捕获并转换。

```java
// ❌ BAD - Controller 层
@PostMapping
public BaseResponse<?> save(@RequestBody DTO dto) {
    if (dto.getName() == null) {
        return BaseResponse.error("名称不能为空");
    }
    return service.save(dto);
}

// ✅ GOOD - Service 层
public void save(DTO dto) {
    if (dto.getName() == null) {
        throw new BusinessException(BusinessErrorEnum.XXX_PARAM_MISSING_01.getI18nErrordesc());
    }
    // ...
}
```

## 2. 异常类型与枚举

禁止 `RuntimeException`/`Exception`。必须使用 `BusinessErrorEnum` + 国际化。**必须使用 `getI18nErrordesc()` 获取国际化文案**，禁止使用 `getDesc()`。禁止在循环/高频调用中频繁抛出异常（异常仅用于异常场景，不用于业务逻辑判断）。

**业务大类编码**：DFP（需求计划）、SOP（产销协同）、MPS（主生产计划）、MRP（物料需求计划）、MDS（主需求计划）、SCH（生产排程）、SYS（系统通用）。编码格式：`业务大类前缀+3位序号`，如 DFP001、SOP001、MPS001、MRP001、MDS001、SCH001、SYS001。**注意编码唯一性，新增编码时要和已有的编码区分。**

**错误码与文案维护位置**：

| 维护项 | 维护位置 |
|--------|----------|
| 错误码定义 | `BusinessErrorEnum` 枚举，`code` 字段（关联国际化配置 key） |
| 错误码与文案映射 | `messages_zh_CN.properties`（中文）、`messages_en_US.properties`（英文），key = 错误码，value = 文案模板（支持 `{0}`、`{1}` 占位符）。**value 中应包含错误码**，如 `MDS001=[MDS001] 物料不存在，请确认后重试。`，确保返回前端的报错语句含错误码。 |
| 加载方式 | 通过 `I18nUtil.getString(code)` 加载，`MessageFormat.format` 填充占位符 |

新增错误码时，须同时在枚举中定义、并在上述 properties 文件中补充对应 key-value。

```java
// ❌ BAD
throw new BusinessException("用户不存在!");
throw new BusinessException(e.getMessage());
throw new RuntimeException("操作失败", e);

// ✅ GOOD
throw new BusinessException(BusinessErrorEnum.MDS_MATERIAL_NOT_EXIST.getI18nErrordesc(materialCode));
```

## 3. 禁止技术信息透传

`e.getMessage()`、`e.toString()`、堆栈、SQL 仅写日志，**禁止**返回前端。

```java
// ❌ BAD
catch (Exception e) {
    return BaseResponse.error("操作失败:" + e.getMessage());
}
throw new BusinessException(e.toString());

// ✅ GOOD
catch (Exception e) {
    log.error("操作失败, errorCode={}", "XXX_SYS_UNKNOWN", e);
    throw new BusinessException(BusinessErrorEnum.SYS_UNKNOWN.getI18nErrordesc());
}
```

## 4. 占位符与参数匹配

模板 `{0}{1}` 必须传入对应数量参数，禁止将 `e.getMessage()` 填入占位符。

```java
// ❌ BAD
throw new BusinessException(BusinessErrorEnum.XXX.getI18nErrordesc(e.getMessage()));

// ✅ GOOD
throw new BusinessException(BusinessErrorEnum.MDS_MATERIAL_NOT_EXIST.getI18nErrordesc(materialCode));
```

## 5. 报错文案规范与模板

### 5.1 规范与指导

- **业务导向**：报错文案必须让业务侧能「看懂、知道下一步该做什么」，避免只给出技术字段名或内部实现信息。
- **信息要素**（能说明白就不再额外堆技术细节）：
  - **是什么问题**：参数缺失、格式不合法、配置缺失、数据不存在/已存在、状态不允许、系统异常等；
  - **作用对象**：清晰点出「哪个业务对象/参数/配置项」；
  - **下一步动作**：提示「请检查…」「请前往…维护」「请稍后重试」等可执行建议。
- **报错语句必须包含错误码**：返回前端的报错文案中**必须显式包含错误码**（如 MDS001、SOP002），便于用户反馈、日志排查与问题定位。推荐格式：`[错误码] 业务文案`，如「[MDS001] 物料不存在，请确认后重试。」；或由框架在返回时统一拼接错误码与文案。
- **统一文案结构**：
  - 业务类文案推荐结构：**「对象/参数」 + 问题 + 下一步指引**，如「尚未配置『需求版本』，请前往『需求版本设置』完成配置后重试」；
  - 技术类异常一律收敛为业务理解层面的提示，如「系统繁忙，请稍后重试。如有问题请联系管理员，并提供错误码与追踪ID」。
- **禁止行为**：
  - 返回前端的报错语句**不包含错误码**；
  - 仅返回「运行失败」「操作失败」等**空泛结论**，不给出任何下一步指引；
  - 参数缺失时只说「参数不能为空」，不点明「哪个参数」；
  - 配置缺失时不指具体「配置项」和「菜单路径」；
  - 状态不允许时只说「不允许」或「失败」，不指出当前状态与被禁止的操作；
  - 直接暴露内部术语或实现细节（如 DTO/VO 类名、Service 名称、SQL 原文等）。
- **术语与文风统一**：
  - 统一使用「物料」（禁用：产品、商品）；「任务」（禁用：作业、Job）；「参数」（禁用：字段、变量）；
  - 句尾统一中文标点（。或！），避免「半角+全角」混用；
  - 参数名、配置项名用「」包裹，便于界面与文案对应，例如：「需求版本」「计划期间」「算法方案」。

### 5.2 推荐模板示例

以下模板均需在最终报错语句中**包含错误码**，格式示例：`[错误码] 业务文案`。properties 中可写为 `错误码 = [错误码] 业务文案`，或由框架统一拼接。

| 场景 | 推荐模板 |
|------|----------|
| 参数缺失 | [错误码] 「{参数名}」不能为空。 |
| 参数格式/范围 | [错误码] 「{参数名}」格式不正确或超出有效范围，请检查后重试。 |
| 配置缺失 | [错误码] 尚未配置「{配置项}」，请前往「{菜单路径}」完成配置后重试。 |
| 数据不存在 | [错误码] {业务对象}不存在，请确认后重试。 |
| 数据已存在 | [错误码] 「{对象标识}」已存在，请勿重复添加。 |
| 状态不允许 | [错误码] {业务对象}已{状态}，无法{操作}。 |
| 删除限制 | [错误码] {业务对象}存在关联数据，无法删除，请先解除关联。 |
| 并发冲突 | [错误码] {业务对象}已被他人修改，请刷新后重试。 |
| 权限不足 | [错误码] 无权限执行该操作，请联系管理员。 |
| 账号状态 | [错误码] 账号已{停用/锁定}，请联系管理员。 |
| 密码相关 | [错误码] 密码{过期/不符合要求}，请先修改密码。 |
| 导入/导出失败 | [错误码] {操作}失败，请下载错误明细并按提示修正后重试。 |
| 模板/格式不支持 | [错误码] 当前{模板/文件格式}不支持在本模块使用，请使用正确模板后重试。 |
| 任务运行中 | [错误码] 计划/任务正在运行中，请稍后再试。 |
| 计算/排程失败 | [错误码] {计划/排程}计算失败，请检查配置与数据后重试。 |
| 流程异常 | [错误码] 流程{启动/执行}失败，请稍后重试或联系管理员。 |
| 外部服务失败 | [错误码] 调用{服务名}失败，请稍后重试。 |
| 请求超时 | [错误码] 请求超时，请稍后重试。 |
| 系统错误 | [错误码] 系统繁忙，请稍后重试。如有问题请联系管理员，并提供错误码与追踪ID。 |

示例中没提及的报错语句，生成时均要遵守**错误码 + 问题 + 解决办法**的规则，且报错语句中必须包含错误码。