功能总览
领域	已实现能力	运行状态
认证与会话	账号密码登录、Access/Refresh Token、登出、会话撤销、设备数限制、API Token	已注册
用户与组织	用户创建、查询、编辑、删改密码、批量删除、角色绑定、基础员工字段	已注册
RBAC	权限字典、角色-权限、角色-菜单、接口权限守卫、通配权限	已注册
菜单	完整菜单树、按用户角色取菜单树、菜单增改删与子节点迁移	已注册
国际化	语言管理、词条 CRUD、批量删除、格式化词条映射、x-lang 请求头	已注册
应用中心	应用列表的分类、关键词和分页查询；首启预置应用	已注册
Mock	按 URL 与方法返回内置 Mock 数据	已注册但默认需认证
员工查询	Sequelize Employee 查询接口	源码存在，未导入根模块
运维支撑	MySQL、Redis、日志轮转、Swagger、Docker、首启分布式锁	已注册
1. 技术架构与启动流程
层次	实现
Web 框架	NestJS 10，所有业务由独立 Module、Controller、Service 组成。
关系数据	TypeORM 连接 MySQL；User、Role、Permission、Menu、Application、Lang、I18 为主要实体。
缓存与会话	ioredis 连接 Redis，保存会话 Token、API Token 和初始化锁。
认证	自定义 JWT 服务使用 HS256 签名；AuthGuard 同时验证 JWT 签名和 Redis 中的有效会话记录。
国际化	nestjs-i18n，使用 x-lang 请求头解析语言；校验错误由全局 I18nValidationPipe 与异常过滤器返回。
日志与文档	Winston 控制台和按小时滚动文件日志；ENABLE_SWAGGER=true 时创建 Swagger 文档。
并发控制	LockerModule 提供 Redis 可重入锁；Token 刷新、签发、撤销按用户 Token 锁串行化。
启动步骤
• 读取 .env 并进行 Joi 配置校验，创建 Nest 应用、全局国际化校验和异常过滤器。
• 注册全局 AuthGuard、RejectRequestGuard、PermissionGuard；所有未标记 @Public 的路由均要求 Bearer Token。
• 通过 Redis 的 LOCK:INSTALL 与 FLAG:INSTALL 控制多实例首启初始化，避免并发重复写入。
• 首次安装读取 locales.json 和 menu-data.json，写入语言/词条、应用、菜单、权限、admin 角色与管理员账号。
• 配置 GEN_MIGRATION 时，程序生成数据库迁移后直接退出；普通模式监听 3000 端口。
2. 数据模型与关系
实体	主要字段	关系/用途
User	id、name、email、password、salt、部门、员工类型、试用/协议日期、地址、状态	与 Role 多对多，关联表 user_role；插入前用随机 salt 和 PBKDF2-SHA256 哈希密码。
Role	id、name	与 Permission 多对多（role_permission）；与 Menu 多对多（role_menu）。
Permission	id、name、desc	接口权限名称，例如 user::query；* 代表拥有所有权限。
Menu	name、path、component、icon、menuType、parentId、order、locale	平面表保存，由服务递归转换为前端菜单树。
Application	name、description、icon、tag、classify	应用中心列表；tag 在读取时尝试 JSON 解析。
Lang	id、name	一种语言对应多个 I18 词条。
I18	id、lang、key、content	国际化词条；同一语言下 key 不能重复。
3. 认证、授权与会话
认证流程
1. 登录时根据 email 查询 User，使用保存的 salt 校验密码。
2. 生成含 jti 的 Access Token 和 Refresh Token；二者的原文及 jti 分别存入 Redis，并设置过期时间。
3. 受保护请求提交 Authorization: Bearer <accessToken>。AuthGuard 验证 JWT 签名后，还检查 Redis 的 at:{uid}:{jti} 是否存在。
4. 刷新令牌时，旧 Access/Refresh Token 会被撤销，再签发新 Token 对；整个过程按用户加锁。
5. 登出、删用户、改密码或改变用户角色时，会撤销该用户的已签发会话。DEVICE_LIMIT 大于 0 时，超过限制会淘汰最早会话。
授权规则
• @Permission 装饰器声明接口所需权限。PermissionGuard 从当前用户全部角色汇总权限；拥有 * 的用户直接放行。
• PREVIEW_MODE=true 时，仅标记 @Reject 的路由会被拒绝，返回预览模式错误。
• @Public 路由跳过全局认证。目前健康检查、登录、刷新 Token、生成 API Token 为公开路由。
API Token
外部系统可通过邮箱和密码调用公开的 API Token 生成接口。Token 使用 JWT，载荷含 type=api；Redis 以 user:{email}:api:{tokenId} 保存。AuthGuard 对 API Token 验证签名并确认 Token 仍在 Redis 中，撤销时删除对应缓存键。默认有效期为 7 天，可由 API_TOKEN_SECONDS 调整。
4. 接口清单
说明：除 @Public 外，均要求 Bearer Token。标记“权限”表示还需对应 Permission；标记“预览拒绝”表示 PREVIEW_MODE=true 时会拒绝请求。
认证模块 /api/auth
方法与路径	访问控制	功能
POST /token/refresh	公开	以 refreshToken 换取新的 Access/Refresh Token 对，同时撤销旧 Token 对。
POST /login	公开	校验 email、password 后创建会话。
POST /logout	认证	撤销请求体 token 对应的 Access/Refresh Token。
POST /api-token	公开（账号密码）	创建供外部系统使用的 API Token，tokenName 可选。
POST /revoke-api-token	认证	按 email 和 tokenId 撤销 API Token。
用户模块 /api/user
方法与路径	访问控制	功能
POST /reg	权限 user::add	创建用户并可绑定 roleIds；支持部门、员工类型、试用期、协议期、地址、状态。
GET /info/:email?	认证	查询指定邮箱或当前会话用户的信息，并附带角色及权限。
DELETE /:email	权限 user::remove；预览拒绝	删除用户并踢出其会话；系统仅剩一个用户时拒绝删除。
PATCH /update	权限 user::update；预览拒绝	更新用户资料与角色；角色变化后撤销用户会话。
GET /	权限 user::query	按 page、limit、name、role、email 分页筛选用户。
PATCH /admin/updatePwd	权限 user::password::force-update；预览拒绝	管理员强制更新指定用户密码并踢出会话。
PATCH /updatePwd	权限 user::update；预览拒绝	校验旧密码后更新密码并踢出会话。
POST /batch	权限 user::batch-remove；预览拒绝	按邮箱数组逐个批量删除，单项错误会被记录后继续。
角色、权限与菜单 /api/role、/api/permission、/api/menu
模块	接口	访问控制与功能
Role	POST /role；GET /role	role::add 创建角色并绑定权限/菜单；role::query 返回角色列表。
Role	GET /role/detail；GET /role/info/:id	detail 需要 role::query，返回分页角色和每个角色的菜单树；info 仅需认证，返回角色详情。
Role	PATCH /role；DELETE /role/:id	分别需 role::update / role::remove，且预览拒绝；存在关联用户的角色不能删除。
Permission	POST、PATCH、GET、DELETE /permission	对应 permission::add、update、get、remove；查询可分页或返回全部。
Menu	GET /menu	需要 menu::query，返回完整菜单树。
Menu	GET /menu/role/:email	仅认证，按用户角色汇总菜单并按 order 排序，再返回菜单树。
Menu	POST、PATCH、DELETE /menu	对应 menu::add、update、remove；更新/删除为预览拒绝。删除时可将子菜单提升到指定父节点或根节点。
国际化与应用 /api/i18、/api/lang、/api/application
模块	接口	访问控制与功能
I18	POST /i18；GET /i18	创建需 i18n::add；查询需 i18n::query，支持 page、limit、all、lang、key、content。
I18	GET /i18/format；GET /i18/:id	format 仅认证，按语言返回 Record<lang, Record<key, content>>；单条查询需 i18n::query。
I18	PATCH、DELETE /i18/:id；POST /i18/batch	需 i18n::update、remove、batch-remove，且均在预览模式下拒绝。
Lang	POST、GET、PATCH、DELETE /lang	分别需 lang::add、query、update、remove；有关联词条的语言不能删除。
Application	GET /application	仅认证；按 classify、keywords 查询应用，返回 data 与 total，tag 尝试转换为 JSON 数组。
其他路由
路径	状态	功能
GET /healthCheck	公开，不带 /api 前缀	返回 success，供探活使用。
GET、POST /api/mock/*	已注册，默认认证	根据内置 data 的 method/url 匹配并返回 Mock 响应；未匹配返回 404。
POST /api/employee/getEmployee；GET /api/employee/getEmployee/:id	未注册	Employee 控制器和 Sequelize 服务存在，但 EmployeesModule 未导入 AppModule。
5. 业务规则与初始化数据
主题	实际规则
用户删除	为了避免系统没有可用账号，UserService 只允许在用户总数大于 1 时删除。
角色删除	RoleService 会先检查是否存在引用该角色的用户；存在则返回 409 Conflict。
菜单删除	删除菜单时遍历全部菜单，原子菜单的直接子节点被转移到指定 parentId；传入 -1 则转为根菜单。
国际化删除	删除 Lang 前先检查是否仍有 I18 词条引用；有引用则返回 409 Conflict。
首次管理员	若 Redis 中不存在 FLAG:INSTALL，则首启创建 admin 角色、全部菜单、* 通配权限和 admin@no-reply.com / admin 管理员账号。
首启幂等	各初始化器会尝试识别已有数据；安装完成后写 FLAG:INSTALL，后续启动不再自动初始化。
6. 配置、部署与可观测性
类别	说明
必需基础依赖	MySQL 和 Redis。默认 Docker Compose 暴露 MySQL 3306、Redis 6379、后端 3000。
数据库配置	DATABASE_HOST、DATABASE_PORT、DATABASE_USERNAME、DATABASE_PASSWORD、DATABASE_NAME、DATABASE_SYNCHRONIZE、DATABASE_AUTOLOADENTITIES。生产环境应关闭 synchronize。
认证配置	AUTH_SECRET 为 JWT 签名密钥；REDIS_SECONDS 为 Access Token 秒数；REFRESH_TOKEN_TTL 为 Refresh Token 毫秒数；DEVICE_LIMIT 为每用户并发会话数。
接口配置	GLOBAL_PREFIX 默认代码兜底为 /api；.env.example 中可覆盖。MOCK_REGEX 用于 Mock 路径截取。
演示与 API 文档	PREVIEW_MODE 默认 true；ENABLE_SWAGGER 默认 false，启用时使用 SWAGGER_TITLE、SWAGGER_DESC、SWAGGER_VERSION。
日志	控制台记录 info；logs/application-YYYY-MM-DD-HH.log 记录 warn/error，logs/info-YYYY-MM-DD-HH.log 记录 info，单文件最大 20 MB，保留 14 天。
运行命令	pnpm -F tinyui-nestjs-server start 启动；start:dev 监听变更；build 构建；migrate:gen 生成迁移；migrate:run 执行 migrate.js。
7. 测试覆盖
仓库已有 Jest 单元测试，覆盖 AuthService/AuthGuard、UserService、RoleService、PermissionService/Guard、MenuService/树转换、I18Service、I18LangService、DbService、RedisService 和 LockerService。NestJS package.json 中的 test:e2e 目前仅输出“未实现 E2E 测试”。本次文档梳理未启动 MySQL、Redis 或后端服务，也未运行测试套件。
8. 当前实现边界与关注点
级别	事项	依据与建议
高	Redis 密码被输出到控制台	main.ts 在启动后直接 console.log REDIS_PASSWORD 及长度。应立即删除明文日志，并检查历史日志和日志采集平台。
高	预览模式并非全面只读	RejectRequestGuard 只拦截标记 @Reject 的路由；创建用户、角色、权限、菜单、语言、词条等 POST 路由未标记，PREVIEW_MODE=true 时仍可写入。应统一策略或补充装饰器。
中	部分查询缺少细粒度权限	GET /user/info/:email?、GET /role/info/:id、GET /menu/role/:email、GET /i18/format、GET /application 仅要求认证，没有 @Permission。应根据业务决定是否限定为本人、管理员或查询权限。
中	员工模块不可达	EmployeesModule 使用 Sequelize，但没有导入 AppModule，且根模块没有 Sequelize 根连接配置。若需该功能，应完成模块注册与数据源配置。
中	默认初始化管理员弱口令	首次启动创建 admin@no-reply.com / admin。部署前应改为环境变量或一次性安全初始化流程，并在首启后强制修改。
中	API Token 生成端点公开	POST /auth/api-token 使用邮箱密码验证后签发 Token，设计上可用，但应配合登录失败限流、审计和强密码策略。
低	Mock 路由默认受认证	MockController 引入了 Public 但未使用。若预期给前端免登录调试，需要显式标记 @Public；若不需要，应移除未使用导入。
低	DTO 字段不一致	CreateUserDto 的 desc 被 @IsNotEmpty 校验但业务服务不使用该字段；调用方需传值才能通过校验。建议删除或纳入实体与业务。
9. 源码依据
关键入口与注册：src/main.ts、src/app.module.ts；业务模块：src/auth、src/user、src/role、src/permission、src/menu、src/i18、src/application、src/mock；基础库：libs/db、libs/redis、libs/jwt、libs/locker、libs/models；部署与参数：package.json、docker-compose.yml、.env.example、config-schema.ts。
