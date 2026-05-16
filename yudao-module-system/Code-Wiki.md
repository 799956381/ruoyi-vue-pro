# yudao-module-system Code Wiki

## 一、项目概述

`yudao-module-system` 是芋道（ruoyi-vue-pro）快速开发平台的 **系统管理基础模块**，提供用户、角色、权限、部门、租户、OAuth2、字典、通知、短信、邮件、社交登录等核心系统能力。作为整个平台的基础支撑层，其他业务模块（如 IoT、CRM、ERP 等）均依赖此模块完成认证授权与基础数据管理。

### 核心能力

| 能力 | 说明 |
|------|------|
| 认证授权 | 账号密码登录、短信登录、社交登录、OAuth2 令牌管理 |
| 权限管理 | RBAC 角色-菜单权限模型、数据权限（部门级别） |
| 用户管理 | 用户 CRUD、密码管理、状态管理、导入导出 |
| 部门岗位 | 树形部门管理、岗位管理 |
| 多租户 | 租户隔离、租户套餐、租户级别数据权限 |
| OAuth2 | 客户端管理、授权码模式、令牌管理 |
| 字典管理 | 字典类型 + 字典数据，全局缓存 |
| 通知系统 | 站内信模板 + 站内信消息 |
| 短信系统 | 短信渠道 + 模板 + 发送 + 验证码 |
| 邮件系统 | 邮件账号 + 模板 + 发送 |
| 社交登录 | 微信公众号/小程序、企业微信等社交账号绑定 |
| 日志审计 | 登录日志、操作日志 |

---

## 二、整体架构

### 2.1 架构图

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          管理后台 / 用户 App                              │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │ HTTP
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      Controller 层 (REST API)                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │   Auth   │ │   User   │ │   Role   │ │   Dept   │ │  Tenant  │ ...  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘      │
└───────┼─────────────┼────────────┼────────────┼────────────┼────────────┘
        │             │            │            │            │
        ▼             ▼            ▼            ▼            ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                       Service 层 (业务逻辑)                               │
│  ┌──────────────┐ ┌───────────────┐ ┌──────────────┐ ┌──────────────┐  │
│  │AdminAuth     │ │AdminUser      │ │Permission    │ │OAuth2Token   │  │
│  │Service       │ │Service        │ │Service       │ │Service       │  │
│  └──────┬───────┘ └──────┬────────┘ └──────┬───────┘ └──────┬───────┘  │
│         │                │                  │                 │          │
│  ┌──────┴───────┐ ┌──────┴────────┐ ┌──────┴───────┐ ┌──────┴───────┐  │
│  │SmsSend       │ │DeptService    │ │RoleService   │ │OAuth2Client  │  │
│  │Service       │ │               │ │MenuService   │ │Service       │  │
│  └──────────────┘ └───────────────┘ └──────────────┘ └──────────────┘  │
│  ┌──────────────┐ ┌───────────────┐ ┌──────────────┐                    │
│  │MailSend      │ │NotifySend     │ │SocialUser    │                    │
│  │Service       │ │Service        │ │Service       │                    │
│  └──────────────┘ └───────────────┘ └──────────────┘                    │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        DAL 层 (数据访问)                                  │
│  ┌────────────────────┐ ┌─────────────────┐ ┌────────────────────────┐  │
│  │  MySQL (MyBatis+)  │ │  Redis (缓存)   │ │  MQ (短信/邮件异步)    │  │
│  │  20+ Mapper        │ │  12+ Cache Key  │ │  Redis Stream          │  │
│  └────────────────────┘ └─────────────────┘ └────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
```

### 2.2 认证授权数据流

```
用户登录请求 → AuthController.login()
    → AdminAuthService.authenticate(username, password)
        → AdminUserService.getUserByUsername()
        → BCryptPasswordEncoder.matches() 密码校验
    → AdminAuthService.login()
        → OAuth2TokenService.createAccessToken(userId, userType, clientId, scopes)
            → 生成 accessToken + refreshToken
            → 存储到 MySQL + Redis 缓存
        → 记录登录日志
    → 返回 AuthLoginRespVO (accessToken, refreshToken, expiresTime)

后续请求 → Security Filter 拦截
    → 从 Header/Parameter 提取 Token
    → OAuth2TokenService.checkAccessToken()
        → 先查 Redis 缓存，未命中则查 MySQL
    → 构建 SecurityContext (LoginUser)
    → 权限校验 @PreAuthorize("@ss.hasPermission('xxx')")
        → PermissionService.hasAnyPermissions()
```

### 2.3 权限模型

```
用户(AdminUserDO) ──N:N── 角色(RoleDO)        ──N:N── 菜单(MenuDO)
       │                      │
       │                      └── 数据权限(DataScopeEnum)
       │
       └── 部门(DeptDO) ── 树形结构

关联表：
  UserRoleDO      → 用户-角色关联
  RoleMenuDO      → 角色-菜单关联
```

---

## 三、包结构详解

### 3.1 整体包结构

```
cn.iocoder.yudao.module.system
├── api/                    # 对外 API 接口（供其他模块 RPC 调用）
│   ├── dept/               # 部门/岗位 API
│   ├── dict/               # 字典 API
│   ├── logger/             # 日志 API
│   ├── mail/               # 邮件 API
│   ├── notify/             # 站内信 API
│   ├── oauth2/             # OAuth2 API
│   ├── permission/         # 权限/角色 API
│   ├── sms/                # 短信 API
│   ├── social/             # 社交用户 API
│   ├── tenant/             # 租户 API
│   └── user/               # 用户 API
├── controller/             # Controller 层
│   ├── admin/              # 管理后台 API
│   │   ├── auth/           # 认证（登录/登出/注册/社交登录）
│   │   ├── captcha/        # 验证码
│   │   ├── dept/           # 部门/岗位
│   │   ├── dict/           # 字典
│   │   ├── ip/             # 地区
│   │   ├── logger/         # 日志
│   │   ├── mail/           # 邮件
│   │   ├── notice/         # 通知公告
│   │   ├── notify/         # 站内信
│   │   ├── oauth2/         # OAuth2
│   │   ├── permission/     # 权限/角色/菜单
│   │   ├── sms/            # 短信
│   │   ├── socail/         # 社交
│   │   ├── tenant/         # 租户
│   │   └── user/           # 用户
│   └── app/                # 用户 App API
│       ├── dict/           # 字典查询
│       ├── ip/             # 地区查询
│       └── tenant/         # 租户查询
├── convert/                # 对象转换（MapStruct）
├── dal/                    # 数据访问层
│   ├── dataobject/         # 数据对象（DO）
│   │   ├── dept/           # DeptDO, PostDO, UserPostDO
│   │   ├── dict/           # DictDataDO, DictTypeDO
│   │   ├── logger/         # LoginLogDO, OperateLogDO
│   │   ├── mail/           # MailAccountDO, MailLogDO, MailTemplateDO
│   │   ├── notice/         # NoticeDO
│   │   ├── notify/         # NotifyMessageDO, NotifyTemplateDO
│   │   ├── oauth2/         # OAuth2AccessTokenDO, OAuth2ClientDO, OAuth2CodeDO, ...
│   │   ├── permission/     # MenuDO, RoleDO, RoleMenuDO, UserRoleDO
│   │   ├── sms/            # SmsChannelDO, SmsCodeDO, SmsLogDO, SmsTemplateDO
│   │   ├── social/         # SocialClientDO, SocialUserBindDO, SocialUserDO
│   │   ├── tenant/         # TenantDO, TenantPackageDO
│   │   └── user/           # AdminUserDO
│   ├── mysql/              # MyBatis Plus Mapper
│   └── redis/              # Redis DAO + Key 常量
├── enums/                  # 枚举定义
│   ├── common/             # SexEnum
│   ├── logger/             # LoginLogTypeEnum, LoginResultEnum
│   ├── mail/               # MailSendStatusEnum
│   ├── notice/             # NoticeTypeEnum
│   ├── notify/             # NotifyTemplateTypeEnum
│   ├── oauth2/             # OAuth2GrantTypeEnum, OAuth2ClientConstants
│   ├── permission/         # DataScopeEnum, MenuTypeEnum, RoleCodeEnum, RoleTypeEnum
│   ├── sms/                # SmsSceneEnum, SmsSendStatusEnum, ...
│   └── social/             # SocialTypeEnum
├── framework/              # 框架扩展
│   ├── captcha/            # 验证码配置与实现
│   ├── datapermission/     # 数据权限配置
│   ├── justauth/           # 社交登录（JustAuth）配置
│   ├── operatelog/         # 操作日志解析函数
│   ├── sms/                # 短信客户端工厂与实现
│   └── web/                # Web 配置
├── job/                    # 定时任务
│   └── token/              # TokenCleanJob（过期令牌清理）
├── mq/                     # 消息队列
│   ├── consumer/           # 消费者（邮件/短信异步发送）
│   ├── message/            # 消息体
│   └── producer/           # 生产者
├── service/                # Service 层
│   ├── auth/               # AdminAuthService
│   ├── dept/               # DeptService, PostService
│   ├── dict/               # DictDataService, DictTypeService
│   ├── logger/             # LoginLogService, OperateLogService
│   ├── mail/               # MailAccountService, MailLogService, MailSendService, MailTemplateService
│   ├── member/             # MemberService（会员桥接）
│   ├── notice/             # NoticeService
│   ├── notify/             # NotifyMessageService, NotifySendService, NotifyTemplateService
│   ├── oauth2/             # OAuth2ApproveService, OAuth2ClientService, OAuth2CodeService, OAuth2GrantService, OAuth2TokenService
│   ├── permission/         # MenuService, PermissionService, RoleService
│   ├── sms/                # SmsChannelService, SmsCodeService, SmsLogService, SmsSendService, SmsTemplateService
│   ├── social/             # SocialClientService, SocialUserService
│   ├── tenant/             # TenantService, TenantPackageService
│   └── user/               # AdminUserService
└── util/                   # 工具类
    └── oauth2/             # OAuth2Utils
```

---

## 四、核心模块详解

### 4.1 认证授权模块（auth）

#### 关键类

| 类名 | 职责 |
|------|------|
| `AuthController` | 认证入口，提供登录/登出/注册/短信登录/社交登录/刷新令牌等接口 |
| `AdminAuthService` | 认证核心逻辑：密码校验、令牌创建、登录日志记录 |
| `OAuth2TokenService` | OAuth2 令牌管理：创建/刷新/校验/移除 accessToken 和 refreshToken |
| `OAuth2GrantService` | OAuth2 授权模式分发：根据 grantType 调用不同的授权逻辑 |
| `OAuth2CodeService` | 授权码模式：生成/校验 authorization code |
| `OAuth2ApproveService` | 用户授权确认：记录用户对客户端的授权决策 |
| `OAuth2ClientService` | OAuth2 客户端管理：CRUD + 缓存 |

#### 登录方式

| 方式 | 接口 | 说明 |
|------|------|------|
| 账号密码 | `POST /system/auth/login` | 支持 BCrypt 加密密码校验 |
| 短信验证码 | `POST /system/auth/sms-login` | 先发送验证码，再登录 |
| 社交登录 | `POST /system/auth/social-login` | 微信/企业微信等社交账号授权码登录 |
| 刷新令牌 | `POST /system/auth/refresh-token` | 使用 refreshToken 换取新 accessToken |
| 注册 | `POST /system/auth/register` | 自助注册新用户 |

#### OAuth2 授权类型

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| authorization_code | `AUTHORIZATION_CODE` | 授权码模式（最安全，推荐） |
| implicit | `IMPLICIT` | 简化模式（已不推荐） |
| password | `PASSWORD` | 密码模式 |
| client_credentials | `CLIENT_CREDENTIALS` | 客户端模式 |
| refresh_token | `REFRESH_TOKEN` | 刷新令牌模式 |

### 4.2 权限管理模块（permission）

#### 关键类

| 类名 | 职责 |
|------|------|
| `PermissionService` | 权限核心：用户-角色关联、角色-菜单关联、数据权限计算 |
| `RoleService` | 角色管理：CRUD + 缓存 + 数据权限设置 |
| `MenuService` | 菜单管理：CRUD + 缓存 + 过滤禁用菜单 |

#### 菜单类型（MenuTypeEnum）

| 类型 | 值 | 说明 |
|------|---|------|
| DIR | 1 | 目录（一级菜单分组） |
| MENU | 2 | 菜单（有对应路由页面） |
| BUTTON | 3 | 按钮（权限标识，如 `system:user:create`） |

#### 数据权限（DataScopeEnum）

| 范围 | 值 | 说明 |
|------|---|------|
| ALL | 1 | 全部数据权限 |
| DEPT_CUSTOM | 2 | 指定部门数据权限（自定义部门列表） |
| DEPT_ONLY | 3 | 本部门数据权限 |
| DEPT_AND_CHILD | 4 | 本部门及以下数据权限 |
| SELF | 5 | 仅本人数据权限 |

数据权限通过 `DataPermissionConfiguration` 配置，自动在 SQL 查询中追加部门/用户过滤条件。

### 4.3 用户管理模块（user）

#### 关键类

| 类名 | 职责 |
|------|------|
| `AdminUserService` | 用户管理：CRUD、密码管理、状态管理、导入导出 |
| `UserController` | 管理后台用户接口 |
| `UserProfileController` | 用户个人信息接口 |

#### AdminUserDO 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 用户 ID |
| username | String | 用户账号 |
| password | String | BCrypt 加密密码 |
| nickname | String | 用户昵称 |
| deptId | Long | 所属部门 |
| postIds | Set\<Long\> | 岗位编号数组（JSON 存储） |
| email | String | 邮箱 |
| mobile | String | 手机号 |
| sex | Integer | 性别 |
| status | Integer | 状态（启用/禁用） |
| loginIp | String | 最后登录 IP |
| loginDate | LocalDateTime | 最后登录时间 |

### 4.4 部门管理模块（dept）

#### 关键类

| 类名 | 职责 |
|------|------|
| `DeptService` | 部门管理：CRUD、子部门递归查询、缓存 |
| `PostService` | 岗位管理：CRUD |

#### DeptDO 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 部门 ID |
| name | String | 部门名称 |
| parentId | Long | 父部门 ID（树形结构） |
| sort | Integer | 显示顺序 |
| leaderUserId | Long | 负责人用户 ID |
| status | Integer | 状态 |

### 4.5 租户管理模块（tenant）

#### 关键类

| 类名 | 职责 |
|------|------|
| `TenantService` | 租户管理：CRUD、租户校验、角色菜单同步 |
| `TenantPackageService` | 租户套餐管理：CRUD、套餐变更时同步租户菜单 |

#### TenantDO 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 租户编号 |
| name | String | 租户名（唯一） |
| contactUserId | Long | 联系人用户 ID |
| status | Integer | 状态 |
| websites | List\<String\> | 绑定域名列表 |
| packageId | Long | 租户套餐编号 |
| expireTime | LocalDateTime | 过期时间 |
| accountCount | Integer | 账号数量上限 |

#### 多租户架构

```
请求 → TenantFilter（从 Header/域名 提取 tenantId）
    → TenantContextHolder（ThreadLocal 存储 tenantId）
    → MyBatis Plus TenantLineInnerInterceptor（自动追加 tenant_id 条件）
    → SQL 执行（自动过滤租户数据）
```

- 所有继承 `TenantBaseDO` 的表自动包含 `tenant_id` 字段
- 使用 `@TenantIgnore` 注解可跳过租户过滤（如 TenantDO 自身）
- 租户套餐（TenantPackageDO）定义了每个租户可用的菜单集合

### 4.6 字典管理模块（dict）

| 类名 | 职责 |
|------|------|
| `DictTypeService` | 字典类型管理 |
| `DictDataService` | 字典数据管理，支持缓存解析 |

字典数据通过 Redis 缓存，提供 `parseDictData(dictType, label)` 方法根据标签反查值。

### 4.7 短信模块（sms）

#### 关键类

| 类名 | 职责 |
|------|------|
| `SmsSendService` | 短信发送：同步创建日志 + 异步 MQ 发送 |
| `SmsCodeService` | 验证码管理：发送/使用/校验 |
| `SmsChannelService` | 短信渠道管理 |
| `SmsTemplateService` | 短信模板管理 |
| `SmsClientFactory` | 短信客户端工厂（根据渠道类型创建客户端） |

#### 支持的短信渠道

| 渠道 | 实现类 | 说明 |
|------|--------|------|
| 阿里云 | `AliyunSmsClient` | 阿里云短信服务 |
| 腾讯云 | `TencentSmsClient` | 腾讯云短信服务 |
| 华为云 | `HuaweiSmsClient` | 华为云短信服务 |
| 七牛云 | `QiniuSmsClient` | 七牛云短信服务 |
| 钉钉调试 | `DebugDingTalkSmsClient` | 钉钉机器人（开发调试用） |

#### 短信发送流程

```
业务调用 SmsSendService.sendSingleSms()
    → 校验模板 + 参数
    → 创建 SmsLogDO（状态：发送中）
    → SmsProducer 发送 MQ 消息
        → SmsSendConsumer 消费
            → SmsSendService.doSendSms()
                → SmsClient.sendSms()（调用云服务 API）
                → 更新 SmsLogDO（状态：已发送/发送失败）
```

### 4.8 邮件模块（mail）

| 类名 | 职责 |
|------|------|
| `MailSendService` | 邮件发送：同步创建日志 + 异步 MQ 发送 |
| `MailTemplateService` | 邮件模板管理 |
| `MailAccountService` | 邮件账号管理 |

邮件发送流程与短信类似，采用 MQ 异步发送模式。

### 4.9 站内信模块（notify）

| 类名 | 职责 |
|------|------|
| `NotifySendService` | 站内信发送 |
| `NotifyTemplateService` | 站内信模板管理 |
| `NotifyMessageService` | 站内信消息管理 |

### 4.10 社交登录模块（social）

| 类名 | 职责 |
|------|------|
| `SocialUserService` | 社交用户管理：绑定/解绑/授权 |
| `SocialClientService` | 社交客户端管理：微信/企业微信等配置 |

#### 支持的社交类型（SocialTypeEnum）

| 类型 | 说明 |
|------|------|
| 微信公众号 | 微信公众号授权登录 |
| 微信小程序 | 微信小程序授权登录 |
| 企业微信 | 企业微信授权登录 |
| 钉钉 | 钉钉授权登录 |
| 飞书 | 飞书授权登录 |

社交登录基于 [JustAuth](https://justauth.wiki/) 库实现统一封装。

### 4.11 日志模块（logger）

| 类名 | 职责 |
|------|------|
| `LoginLogService` | 登录日志记录 |
| `OperateLogService` | 操作日志记录 |

#### 登录日志类型（LoginLogTypeEnum）

| 类型 | 值 | 说明 |
|------|---|------|
| LOGIN_USERNAME | 100 | 账号密码登录 |
| LOGIN_MOBILE | 101 | 手机号登录 |
| LOGIN_SOCIAL | 102 | 社交登录 |
| LOGOUT_SELF | 200 | 主动登出 |
| LOGOUT_TIMEOUT | 201 | 超时登出 |

### 4.12 通知公告模块（notice）

| 类名 | 职责 |
|------|------|
| `NoticeService` | 通知公告 CRUD |

---

## 五、数据对象总览

| DO | 对应表 | 继承 | 说明 |
|----|--------|------|------|
| `AdminUserDO` | `system_users` | TenantBaseDO | 管理后台用户 |
| `DeptDO` | `system_dept` | TenantBaseDO | 部门（树形） |
| `PostDO` | `system_post` | TenantBaseDO | 岗位 |
| `UserPostDO` | `system_user_post` | BaseDO | 用户-岗位关联 |
| `RoleDO` | `system_role` | TenantBaseDO | 角色（含数据权限） |
| `MenuDO` | `system_menu` | BaseDO | 菜单（目录/菜单/按钮） |
| `RoleMenuDO` | `system_role_menu` | BaseDO | 角色-菜单关联 |
| `UserRoleDO` | `system_user_role` | BaseDO | 用户-角色关联 |
| `DictTypeDO` | `system_dict_type` | TenantBaseDO | 字典类型 |
| `DictDataDO` | `system_dict_data` | TenantBaseDO | 字典数据 |
| `TenantDO` | `system_tenant` | BaseDO | 租户 |
| `TenantPackageDO` | `system_tenant_package` | BaseDO | 租户套餐 |
| `OAuth2ClientDO` | `system_oauth2_client` | BaseDO | OAuth2 客户端 |
| `OAuth2AccessTokenDO` | `system_oauth2_access_token` | TenantBaseDO | 访问令牌 |
| `OAuth2RefreshTokenDO` | `system_oauth2_refresh_token` | TenantBaseDO | 刷新令牌 |
| `OAuth2CodeDO` | `system_oauth2_code` | TenantBaseDO | 授权码 |
| `OAuth2ApproveDO` | `system_oauth2_approve` | TenantBaseDO | 授权确认 |
| `SmsChannelDO` | `system_sms_channel` | TenantBaseDO | 短信渠道 |
| `SmsTemplateDO` | `system_sms_template` | TenantBaseDO | 短信模板 |
| `SmsLogDO` | `system_sms_log` | TenantBaseDO | 短信日志 |
| `SmsCodeDO` | `system_sms_code` | BaseDO | 短信验证码 |
| `MailAccountDO` | `system_mail_account` | TenantBaseDO | 邮件账号 |
| `MailTemplateDO` | `system_mail_template` | TenantBaseDO | 邮件模板 |
| `MailLogDO` | `system_mail_log` | TenantBaseDO | 邮件日志 |
| `NotifyTemplateDO` | `system_notify_template` | TenantBaseDO | 站内信模板 |
| `NotifyMessageDO` | `system_notify_message` | TenantBaseDO | 站内信消息 |
| `SocialClientDO` | `system_social_client` | TenantBaseDO | 社交客户端 |
| `SocialUserDO` | `system_social_user` | TenantBaseDO | 社交用户 |
| `SocialUserBindDO` | `system_social_user_bind` | BaseDO | 社交用户绑定 |
| `LoginLogDO` | `system_login_log` | BaseDO | 登录日志 |
| `OperateLogDO` | `system_operate_log` | BaseDO | 操作日志 |
| `NoticeDO` | `system_notice` | TenantBaseDO | 通知公告 |

---

## 六、缓存策略

| 缓存 Key | 说明 | 过期策略 |
|----------|------|---------|
| `role:{id}` | 角色信息 | 主动失效 |
| `user_role_ids:{userId}` | 用户角色编号集合 | 主动失效 |
| `menu_role_ids:{menuId}` | 拥有指定菜单的角色编号 | 主动失效 |
| `permission_menu_ids:{permission}` | 权限对应的菜单编号 | 主动失效 |
| `dept_children_ids:{id}` | 部门子部门编号集合 | 主动失效 |
| `oauth2_access_token:{token}` | 访问令牌信息 | 动态过期（与令牌一致） |
| `oauth_client:{id}` | OAuth2 客户端信息 | 主动失效 |
| `notify_template:{code}` | 站内信模板 | 主动失效 |
| `mail_account:{id}` | 邮件账号 | 主动失效 |
| `mail_template:{code}` | 邮件模板 | 主动失效 |
| `sms_template:{id}` | 短信模板 | 主动失效 |
| `wxa_subscribe_template:{userType}` | 小程序订阅模板 | 主动失效 |

---

## 七、API 接口概览

### 7.1 管理后台 API

| 模块 | 路径前缀 | 主要接口 |
|------|---------|---------|
| 认证 | `/system/auth/` | login, logout, refresh-token, register, sms-login, social-login, get-permission-info, send-sms-code, reset-password |
| 用户 | `/system/user/` | CRUD, update-password, update-status, import, export-excel, simple-list |
| 用户信息 | `/system/user-profile/` | get, update, update-password |
| 角色 | `/system/role/` | CRUD, simple-list, export-excel |
| 菜单 | `/system/menu/` | CRUD, list, simple-list |
| 权限 | `/system/permission/` | assign-role-menu, assign-user-role, assign-role-data-scope |
| 部门 | `/system/dept/` | CRUD, list, simple-list |
| 岗位 | `/system/post/` | CRUD, page, simple-list |
| 字典类型 | `/system/dict-type/` | CRUD, page, simple-list |
| 字典数据 | `/system/dict-data/` | CRUD, page, simple-list, clean |
| 租户 | `/system/tenant/` | CRUD, page, get-id-by-name, get-by-website, simple-list |
| 租户套餐 | `/system/tenant-package/` | CRUD, page, simple-list |
| OAuth2 客户端 | `/system/oauth2/client/` | CRUD, page |
| OAuth2 令牌 | `/system/oauth2/token/` | page, delete |
| OAuth2 开放 | `/system/oauth2/open/` | authorize, token, check-token, revoke |
| 短信渠道 | `/system/sms/channel/` | CRUD, page, simple-list |
| 短信模板 | `/system/sms/template/` | CRUD, page, send-sms |
| 短信日志 | `/system/sms/log/` | page |
| 短信回调 | `/system/sms/callback/` | 渠道回调 |
| 邮件账号 | `/system/mail/account/` | CRUD, page, simple-list |
| 邮件模板 | `/system/mail/template/` | CRUD, page, send-mail, simple-list |
| 邮件日志 | `/system/mail/log/` | page |
| 站内信模板 | `/system/notify/template/` | CRUD, page |
| 站内信消息 | `/system/notify/message/` | page, my-page, get-detail, read, read-all |
| 通知公告 | `/system/notice/` | CRUD, page |
| 登录日志 | `/system/login-log/` | page |
| 操作日志 | `/system/operate-log/` | page, delete |
| 地区 | `/system/area/` | tree |
| 验证码 | `/system/captcha/` | get-image |
| 社交用户 | `/system/social-user/` | page, bind, unbind |
| 社交客户端 | `/system/social-client/` | CRUD, page |

### 7.2 用户 App API

| 模块 | 路径前缀 | 主要接口 |
|------|---------|---------|
| 字典 | `/system/dict-data/` | simple-list（App 端字典查询） |
| 地区 | `/system/area/` | tree（App 端地区查询） |
| 租户 | `/system/tenant/` | get-by-website（App 端租户查询） |

### 7.3 对外 RPC API（供其他模块调用）

| API 接口 | 方法 | 说明 |
|----------|------|------|
| `DeptApi` | getDept, getDeptList | 部门查询 |
| `PostApi` | getPostList | 岗位查询 |
| `DictDataApi` | validateDictDataList, getDictData, parseDictData | 字典校验与解析 |
| `AdminUserApi` | getUser, getUserList | 用户查询 |
| `RoleApi` | getRoleList | 角色查询 |
| `PermissionApi` | getUserRoleIdListByUserId, hasAnyPermissions, hasAnyRoles, getDeptDataPermission | 权限查询 |
| `LoginLogApi` | createLoginLog | 创建登录日志 |
| `OperateLogApi` | createOperateLog, getOperateLogPage | 操作日志 |
| `SmsSendApi` | sendSingleSmsToAdmin, sendSingleSmsToMember | 短信发送 |
| `SmsCodeApi` | sendSmsCode, useSmsCode, validateSmsCode | 验证码 |
| `MailSendApi` | sendSingleMailToAdmin, sendSingleMailToMember | 邮件发送 |
| `NotifyMessageSendApi` | sendSingleMessageToAdmin, sendSingleMessageToMember | 站内信发送 |
| `SocialUserApi` | getSocialUser, bindSocialUser, unbindSocialUser | 社交用户 |
| `SocialClientApi` | getAuthorizeUrl, getSocialUser, refreshSocialUser | 社交客户端 |
| `OAuth2TokenApiImpl` | createAccessToken, checkAccessToken, removeAccessToken | OAuth2 令牌 |
| `TenantApiImpl` | getTenant, validateTenant | 租户校验 |

---

## 八、框架扩展

### 8.1 数据权限（DataPermission）

通过 `DataPermissionConfiguration` 配置，自动为 `AdminUserDO` 和 `DeptDO` 注册数据权限规则：

- `rule.addDeptColumn(AdminUserDO.class)` → 查询用户时自动追加部门过滤
- `rule.addDeptColumn(DeptDO.class, "id")` → 查询部门时自动追加部门过滤
- `rule.addUserColumn(AdminUserDO.class, "id")` → 查询用户时自动追加用户过滤

使用 `@DataPermission(enable = false)` 注解可临时禁用数据权限。

### 8.2 社交登录（JustAuth）

通过 `YudaoJustAuthConfiguration` 和 `AuthRequestFactory` 集成 JustAuth 库，支持动态创建社交登录的 `AuthRequest` 实例，配置来源于数据库（`SocialClientDO`）而非静态配置文件。

### 8.3 短信客户端（SmsClient）

通过 `SmsClientFactory` 和 `SmsConfiguration` 实现短信渠道的动态管理：

- `AbstractSmsClient` 定义了短信客户端的通用模板
- 各渠道实现（阿里云/腾讯云/华为云/七牛云/钉钉调试）继承抽象类
- 渠道配置来源于数据库（`SmsChannelDO`），支持运行时动态增减

### 8.4 操作日志解析（OperateLog）

通过自定义 `ParseFunction` 实现 mzt-biz-log 操作日志的变量解析：

| 解析函数 | 说明 |
|----------|------|
| `AdminUserParseFunction` | 解析用户名称 |
| `DeptParseFunction` | 解析部门名称 |
| `PostParseFunction` | 解析岗位名称 |
| `SexParseFunction` | 解析性别 |
| `BooleanParseFunction` | 解析布尔值 |
| `AreaParseFunction` | 解析地区名称 |

### 8.5 验证码（Captcha）

通过 `YudaoCaptchaConfiguration` 集成 AJ-Captcha 验证码组件，支持滑块验证和文字点选验证，验证码存储使用 Redis。

---

## 九、消息队列

### 9.1 短信发送

```
SmsProducer.sendSmsSendMessage()
    → Redis Stream: sms_send
        → SmsSendConsumer.onMessage()
            → SmsSendService.doSendSms()
```

### 9.2 邮件发送

```
MailProducer.sendMailSendMessage()
    → Redis Stream: mail_send
        → MailSendConsumer.onMessage()
            → MailSendService.doSendMail()
```

---

## 十、定时任务

| 任务 | 类名 | 说明 |
|------|------|------|
| 令牌清理 | `TokenCleanJob` | 定期清理过期的 OAuth2 访问令牌和刷新令牌 |
| 示例任务 | `DemoJob` | 演示用定时任务 |

---

## 十一、依赖关系

### 11.1 模块依赖

```
yudao-module-system ──→ yudao-module-infra
                      ──→ yudao-spring-boot-starter-biz-data-permission
                      ──→ yudao-spring-boot-starter-biz-tenant
                      ──→ yudao-spring-boot-starter-biz-ip
                      ──→ yudao-spring-boot-starter-security
                      ──→ yudao-spring-boot-starter-mybatis
                      ──→ yudao-spring-boot-starter-redis
                      ──→ yudao-spring-boot-starter-job
                      ──→ yudao-spring-boot-starter-mq
                      ──→ yudao-spring-boot-starter-excel
                      ──→ yudao-spring-boot-starter-test
                      ──→ spring-boot-starter-mail
                      ──→ spring-boot-starter-validation
                      ──→ JustAuth (社交登录)
                      ──→ justauth-spring-boot-starter
                      ──→ wx-java-mp-spring-boot-starter (微信公众号)
                      ──→ wx-java-miniapp-spring-boot-starter (微信小程序)
                      ──→ captcha-spring-boot-starter (验证码)
```

### 11.2 外部中间件依赖

| 中间件 | 用途 | 必需 |
|--------|------|------|
| MySQL | 关系数据存储 | 是 |
| Redis | 缓存 + 消息队列（短信/邮件异步发送） | 是 |

### 11.3 核心框架依赖

| 框架 | 用途 |
|------|------|
| Spring Boot | 基础框架 |
| Spring Security | 安全认证与权限控制 |
| MyBatis Plus | ORM 框架 |
| Spring Data Redis | 缓存与消息队列 |
| JustAuth | 社交登录统一封装 |
| WxJava | 微信公众号/小程序 SDK |
| AJ-Captcha | 验证码 |
| MapStruct | 对象转换 |
| mzt-biz-log | 操作日志 |

---

## 十二、项目运行方式

### 12.1 作为主程序模块运行

`yudao-module-system` 作为 `yudao-server` 的依赖模块运行，不需要单独启动。

**启动步骤：**

1. 准备中间件：MySQL、Redis
2. 初始化数据库：执行 `sql/mysql/ruoyi-vue-pro.sql` 中的系统模块相关表
3. 配置 `application-local.yaml`：
   - MySQL 数据源
   - Redis 连接
   - 短信/邮件渠道配置（按需）
4. 启动 `YudaoServerApplication`

### 12.2 初始数据

系统启动后需要以下初始数据：

| 数据 | 说明 |
|------|------|
| 超级管理员角色 | `role_code = "super_admin"`，拥有所有权限 |
| 系统租户 | `package_id = 0`，系统内置租户 |
| OAuth2 客户端 | `client_id = "default"`，默认客户端 |
| 字典数据 | 系统通用字典（性别、状态、类型等） |
| 菜单数据 | 系统管理菜单树 |

---

## 十三、错误码体系

系统模块使用 `1-002-xxx-xxx` 段错误码，按子模块划分：

| 模块 | 错误码段 | 示例 |
|------|---------|------|
| 认证 | 1_002_000_xxx | AUTH_LOGIN_BAD_CREDENTIALS |
| 菜单 | 1_002_001_xxx | MENU_NOT_EXISTS |
| 角色 | 1_002_002_xxx | ROLE_NOT_EXISTS |
| 用户 | 1_002_003_xxx | USER_NOT_EXISTS |
| 部门 | 1_002_004_xxx | DEPT_NOT_FOUND |
| 岗位 | 1_002_005_xxx | POST_NOT_FOUND |
| 字典类型 | 1_002_006_xxx | DICT_TYPE_NOT_EXISTS |
| 字典数据 | 1_002_007_xxx | DICT_DATA_NOT_EXISTS |
| 通知公告 | 1_002_008_xxx | NOTICE_NOT_FOUND |
| 短信渠道 | 1_002_011_xxx | SMS_CHANNEL_NOT_EXISTS |
| 短信模板 | 1_002_012_xxx | SMS_TEMPLATE_NOT_EXISTS |
| 短信发送 | 1_002_013_xxx | SMS_SEND_MOBILE_NOT_EXISTS |
| 短信验证码 | 1_002_014_xxx | SMS_CODE_NOT_FOUND |
| 租户 | 1_002_015_xxx | TENANT_NOT_EXISTS |
| 租户套餐 | 1_002_016_xxx | TENANT_PACKAGE_NOT_EXISTS |
| 社交用户 | 1_002_018_xxx | SOCIAL_USER_AUTH_FAILURE |
| OAuth2 客户端 | 1_002_020_xxx | OAUTH2_CLIENT_NOT_EXISTS |
| OAuth2 授权 | 1_002_021_xxx | OAUTH2_GRANT_CLIENT_ID_MISMATCH |
| OAuth2 授权码 | 1_002_022_xxx | OAUTH2_CODE_NOT_EXISTS |
| 邮箱账号 | 1_002_023_xxx | MAIL_ACCOUNT_NOT_EXISTS |
| 邮件模板 | 1_002_024_xxx | MAIL_TEMPLATE_NOT_EXISTS |
| 邮件发送 | 1_002_025_xxx | MAIL_SEND_MAIL_NOT_EXISTS |
| 站内信模板 | 1_002_026_xxx | NOTIFY_TEMPLATE_NOT_EXISTS |
| 站内信发送 | 1_002_028_xxx | NOTIFY_SEND_TEMPLATE_PARAM_MISS |

---

## 十四、技术栈总结

| 层次 | 技术选型 |
|------|---------|
| 基础框架 | Spring Boot 2.x |
| 安全框架 | Spring Security + OAuth2 |
| ORM 框架 | MyBatis Plus |
| 缓存 | Redis (Spring Data Redis + Redisson) |
| 消息队列 | Redis Stream |
| 社交登录 | JustAuth + WxJava |
| 短信服务 | 阿里云/腾讯云/华为云/七牛云 SDK |
| 邮件服务 | Spring Boot Mail |
| 验证码 | AJ-Captcha |
| 操作日志 | mzt-biz-log |
| 对象转换 | MapStruct |
| 接口文档 | Swagger 3 / Knife4j |
| 工具类 | Hutool |
| Excel | EasyExcel |
