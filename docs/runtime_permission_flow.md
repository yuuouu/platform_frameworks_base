# Android Runtime Permission Flow (frameworks/base)

## 1) 源码目录定位
- App 侧入口：`core/java/android/app/Activity.java` 中 `requestPermissions`/`onRequestPermissionsResult` 及并发请求防抖。
- Intent 生成：`core/java/android/content/pm/PackageManager.java#buildRequestPermissionsIntent` 创建 `ACTION_REQUEST_PERMISSIONS` 发送至 PermissionController。
- 权限检查：`core/java/android/app/ContextImpl.java#checkSelfPermission` 调用系统检查（含弃权处理）。
- 系统服务：`services/core/java/com/android/server/pm/permission/PermissionManagerService*.java`（AIDL `IPermissionManager` 的实现包装与核心逻辑 `PermissionManagerServiceImpl`）。
- 权限状态存储：`services/core/java/com/android/server/pm/permission/UidPermissionState.java` 管理 UID 的授予/回收及 flag。
- Binder 接口：`core/java/android/permission/IPermissionManager.aidl` 定义授权、撤销、rationale 查询等 IPC。

## 2) 调用链概览
1. **调用/检查**：App 通过 `Context.checkSelfPermission` 读取当前授予状态。
2. **请求入口**：`Activity.requestPermissions` 校验参数/并发 → 通过 `PackageManager.buildRequestPermissionsIntent` 生成 Intent → `startActivityForResult` 跳转到 PermissionController UI。
3. **用户交互**：PermissionController（独立系统应用）呈现授权界面，基于 Intent 中的权限列表决策结果。
4. **IPC 回调**：PermissionController 使用 `IPermissionManager.grantRuntimePermission`/`revokeRuntimePermission` 等 Binder API 与系统服务通信。
5. **系统授权**：`PermissionManagerService` 作为 Binder Stub 校验调用者并委托给 `PermissionManagerServiceImpl`，后者检查包状态、权限类型、flag/限制，更新 `UidPermissionState` 并触发回调（GID 更新、metrics）。
6. **结果回送**：Activity 通过 `onRequestPermissionsResult` 接收结果（被打断时空数组表示取消）。

## 3) 核心类/方法要点
- **`Activity.requestPermissions`**：
  - 防重复：`mHasCurrentPermissionsRequest` 置位；若已有请求则直接回调空结果。 
  - 校验弃权权限，选择对应 `PackageManager`（主/虚拟设备），构造授权 Intent 并启动 UI。【F:core/java/android/app/Activity.java†L5701-L5799】
  - 回调：`onRequestPermissionsResult`（含 deviceId 重载）默认空实现供应用覆写。【F:core/java/android/app/Activity.java†L5801-L5847】

- **`PackageManager.buildRequestPermissionsIntent`**：
  - 校验权限数组非空，创建 `ACTION_REQUEST_PERMISSIONS` Intent，附加权限列表并指定 PermissionController 包名。【F:core/java/android/content/pm/PackageManager.java†L7247-L7254】

- **`ContextImpl.checkSelfPermission`**：
  - 空值校验，若权限被 renounce 视为拒绝；否则调用 `checkPermission` 查询当前进程 UID/PID 权限结果。【F:core/java/android/app/ContextImpl.java†L2451-L2461】

- **`PermissionManagerService` / `IPermissionManager`**：
  - Binder stub (`IPermissionManager.Stub`) 初始化核心实现 `PermissionManagerServiceImpl` 或接入 access-checking service；AIDL 定义授权、撤销、rationale 查询、跨用户检查等接口。【F:services/core/java/com/android/server/pm/permission/PermissionManagerService.java†L102-L168】【F:core/java/android/permission/IPermissionManager.aidl†L28-L111】

- **`PermissionManagerServiceImpl.grantRuntimePermissionInternal`**：
  - 权限/用户/包存在性校验、跨用户与 GRANT 权限验证。 
  - 检查权限类型（runtime/development/role/soft/hard restricted）、flag（SYSTEM_FIXED/POLICY_FIXED 等）、instant-app 限制、targetSdk < M 早退。 
  - 更新 `UidPermissionState` 授权，记录 metrics，回调 GID/安装权限通知。【F:services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java†L1332-L1502】

- **`UidPermissionState`**：
  - 提供 `grantPermission`/`revokePermission` 更新单个 UID 权限状态及 flag，维护缓存。【F:services/core/java/com/android/server/pm/permission/UidPermissionState.java†L233-L255】

## 4) Binder / IPC 交互点
- App → PermissionController：通过 `ACTION_REQUEST_PERMISSIONS` Intent 交互（PackageManager 指定控制器包）。
- PermissionController → System：调用 `IPermissionManager` AIDL（`grantRuntimePermission`、`revokeRuntimePermission`、`shouldShowRequestPermissionRationale` 等）。
- System 内部：`PermissionManagerService` stub 将 Binder 调用转发至 `PermissionManagerServiceImpl`，后者操作 `UidPermissionState` 并通知回调。

## 5) 状态/流程图（文字版）
```
App                     System (framework)                    PermissionController
---------------------------------------------------------------------------------
checkSelfPermission() --binder--> PermissionManagerService.checkPermission()
                                      ↓ state from UidPermissionState
requestPermissions()                                                     \
  - debounce, renounce check                                               \
  - intent = PackageManager.buildRequestPermissionsIntent(perms)            \
  - startActivityForResult(intent) --> [UI shown by PermissionController] ---> user decision
                                        ↑ RESULT via IPermissionManager.grantRuntimePermission
                                        PermissionManagerServiceImpl:
                                          * validate caller/user/flags
                                          * ensure permission type allowed
                                          * update UidPermissionState.grantPermission()
                                          * callbacks (metrics, gids)
                                        ↓
Activity.onRequestPermissionsResult(grantResults)
```

## 6) 典型生命周期示例
1. 应用检查：`checkSelfPermission(READ_CONTACTS)` → 若返回 `PERMISSION_DENIED` 则进入下一步。【F:core/java/android/app/ContextImpl.java†L2451-L2461】
2. 发起请求：`requestPermissions([READ_CONTACTS], rc)` 生成授权 Intent 并跳转 PermissionController；重复请求会立即收到空结果表示取消。【F:core/java/android/app/Activity.java†L5770-L5799】
3. 用户同意：PermissionController 调用 `IPermissionManager.grantRuntimePermission`。Binder 进入 `PermissionManagerService` → `PermissionManagerServiceImpl.grantRuntimePermissionInternal` 完成校验与授予。【F:services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java†L1332-L1502】
4. 状态写入：`UidPermissionState.grantPermission` 标记权限为 granted 并保留 flags，必要时发出 GID 变更回调。【F:services/core/java/com/android/server/pm/permission/UidPermissionState.java†L233-L255】
5. 结果回调：Activity 收到 `onRequestPermissionsResult`（空实现，应用覆写）。【F:core/java/android/app/Activity.java†L5801-L5847】

## 7) 状态迁移要点
- 未授予 → 请求中（`mHasCurrentPermissionsRequest` 防并发） → 用户允许 → 系统校验通过 → `UidPermissionState` 标记 granted → 回调应用。
- 若权限 SYSTEM_FIXED / POLICY_FIXED / hard/soft restricted 且未豁免，则流程在系统侧被拒绝并记录日志，不更新状态。【F:services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java†L1377-L1502】
- targetSdk < M 的 legacy 应用危险权限保持兼容：授权请求直接返回，不执行授予。【F:services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java†L1429-L1479】
```
state: DENIED -> (request start) -> WAITING -> {ALLOW -> GRANTED | DENY -> DENIED}
flags: SYSTEM_FIXED/POLICY_FIXED/hard|soft restricted gate transitions
```

