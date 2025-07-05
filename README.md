# ai-commodity-sales-backend
# Commodity Sales System 后端（Spring Boot）

本项目为一个基于 Spring Boot + MySQL 的商品销售系统后端，支持用户注册、登录、商品浏览、购物车、下单等功能。前后端分离，适配 Vue 前端。

---

## 一、环境要求
- JDK 11（强烈建议，不支持 JDK 21）
- Maven 3.6+
- MySQL 5.7/8.0

---

## 二、启动步骤

### 1. 克隆项目
```
git clone <项目地址>
```

### 2. 初始化数据库
1. 启动 MySQL 服务。
2. 用命令行或工具导入 `init_db.sql`：
   - 命令行方式：
     ```sql
     mysql -u root -p < C:/Users/甄/CascadeProjects/commodity-sales-system/backend/init_db.sql
     ```
   - 或用 Navicat、DBeaver 等工具执行 SQL 文件。
3. 数据库名：`commodity_sales`（已在 SQL 文件中自动创建）。

### 3. 配置数据库连接
- 编辑 `src/main/resources/application.yml`，修改 `username` 和 `password` 为你的 MySQL 账号密码。

### 4. 编译并启动后端服务
```
# 确保已切换到 JDK 11
java -version    # 显示 11.x 即可

# 编译打包
mvn clean package

# 启动服务
java -jar target/commodity-sales-backend-0.0.1-SNAPSHOT.jar
```
- 默认端口：8080
- 启动无报错即访问成功

---

## 三、后端详细 API 接口文档

所有接口返回统一结构：
```json
{
  "code": 200,
  "message": "success",
  "data": ...
}
```
如出错：
```json
{
  "code": 401,
  "message": "用户名或密码错误",
  "data": null
}
```

---

### 1. 用户与权限相关接口

#### 1.1 获取图片验证码
- **GET** `/api/captcha`
- 返回：JPEG图片，自动写入session，前端需展示图片并填写验证码

#### 1.2 用户登录
- **POST** `/api/users/login`
- 请求参数（JSON）：
```json
{
  "username": "admin",
  "password": "123456",
  "captcha": "abcd"
}
```
- 返回：
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "username": "admin",
    "role": "ADMIN",
    "createTime": "2025-06-21T14:00:00"
  }
}
```
- 失败情况：
  - 验证码错误：`code: 400, message: "验证码错误"`
  - 账户锁定/禁用：`code: 403, message: "账户已锁定，请10分钟后再试"` 或 `"账户已被禁用"`
  - 密码错误：`code: 401, message: "用户名或密码错误"`

#### 1.3 注册/创建用户
- **POST** `/api/users`
- 请求参数（JSON）：
```json
{
  "username": "newuser",
  "password": "123456",
  "role": "USER"
}
```
- 返回：新用户信息（同上）

#### 1.4 修改用户信息
- **PUT** `/api/users/{id}`
- 权限：需本人或管理员
- 请求参数：同注册
- 返回：修改后用户信息

#### 1.5 删除用户
- **DELETE** `/api/users/{id}`
- 权限：仅管理员（`@PreAuthorize("hasRole('ADMIN')")`）
- 返回：`code: 200, data: null`

#### 1.6 分页/角色过滤查询用户
- **GET** `/api/users?page=1&size=10&role=ADMIN`
- 权限：管理员/普通用户均可
- 返回：用户DTO列表

#### 1.7 根据ID查询用户
- **GET** `/api/users/{id}`
- 权限：管理员/本人
- 返回：用户DTO

#### 1.8 修改密码
- **POST** `/api/users/{id}/change-password`
- 权限：本人
- 请求参数：
```json
{
  "oldPassword": "123456",
  "newPassword": "654321"
}
```
- 返回：修改结果

#### 1.9 获取所有角色
- **GET** `/api/users/roles`
- 权限：管理员/普通用户均可
- 返回：
```json
{
  "code": 200,
  "data": ["ADMIN", "USER"]
}
```

#### 1.10 为用户分配角色
- **POST** `/api/users/{id}/assign-role`
- 权限：仅管理员（`@PreAuthorize("hasRole('ADMIN')")`）
- 请求参数：
```json
{
  "role": "ADMIN"
}
```
- 返回：`code: 200, data: "角色分配成功"`

---

### 2. 账户安全与登录防护说明
- 登录接口集成验证码校验（需先调用 `/api/captcha` 获取图片，用户输入后再登录）。
- 连续5次登录失败自动锁定账户10分钟。
- 账户被禁用（status=0）或锁定（locked_until）期间禁止登录。
- 登录成功自动重置失败计数。

---

### 3. 统一响应结构
- 所有接口返回 `ApiResponse<T>`，字段：
  - `code`：状态码，200为成功，其他为错误
  - `message`：提示信息
  - `data`：数据内容

---

### 4. Swagger UI
- 启动后访问 [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html) 可在线查看和调试所有接口。

---

### 5. 商品多图与主图标记功能

#### 数据库设计
- 新增 `product_image` 表，支持一个商品多张图片（含排序、主图标记字段）。
- `ProductImage` 实体新增 `isMain` 字段，标记是否为主图（每个商品建议仅有一张主图）。
- `Product` 实体 images 字段为 List<ProductImage>，自动级联保存。

#### 图片上传
- **POST** `/api/upload`
- 参数：`file`（图片文件），`type=product`
- 返回：图片URL（如 `/images/product/2025/06/24/xxx.jpg`）
- 前端上传多张图片后，将所有图片URL作为 images 数组提交到商品接口。

#### 商品新增/编辑
- **POST** `/api/products`  
- **PUT** `/api/products/{id}`
- 参数中包含 `images` 字段（数组），每个元素为 `{imageUrl: "...", sortOrder: 0, isMain: true/false}`。
- 前端可在多图中选择一张设为主图（isMain=true），其余为副图。
- 后端自动保存所有图片，并建议每个商品仅有一张主图。

#### 图片访问
- 通过 `/images/product/年月日/xxx.jpg` 直接访问图片。

#### 图片删除/批量删除
- 单张图片：`DELETE /api/images?url=xxx`
- 批量图片：`DELETE /api/images/batch`，body为URL数组。

#### 前端交互建议
- 支持多文件选择和预览，上传后将所有图片URL同步到商品表单的 images 字段。
- 编辑商品时可增删图片、调整顺序、设置主图（如单选框/按钮），保存时一并提交。
- 展示商品时建议优先显示主图（isMain=true），其余为轮播或缩略图。

---
如需其它业务模块（商品、订单、购物车等）详细接口文档，请补充需求。

### 5. 订单
- `GET    /api/orders/{userId}`  查询用户所有订单
- `POST   /api/orders`           创建订单（结算）
- `GET    /api/orders/detail/{orderId}` 查询订单详情

---

## 四、常见问题
- **JDK版本不兼容**：请务必使用 JDK 11。
- **数据库连接失败**：检查 application.yml 配置和 MySQL 是否启动。
- **端口冲突**：如 8080 被占用，可在 application.yml 修改 `server.port`。

---

## 五、前端启动

前端 Vue 项目位于 `../frontend` 目录。启动步骤如下：

### 1. 环境要求
- Node.js 16 或以上（建议 16.x/18.x）
- npm 8+ 或 yarn

### 2. 安装依赖
```shell
cd ../frontend
npm install
# 或 yarn install
```

### 3. 启动开发服务器
```shell
npm run dev
# 或 yarn dev
```
- 默认访问地址：http://localhost:3000

### 4. 配置后端代理（如需修改）
- 默认 `vite.config.js` 已将 `/api` 代理到后端 `http://localhost:8080`
- 如后端端口有变，修改 `vite.config.js` 的 proxy 配置即可

### 5. 构建生产包（可选）
```shell
npm run build
# 生成的静态文件在 dist/ 目录
```

### 6. 主要页面与功能
- 首页：商品列表、分类筛选
- 商品详情页：展示商品信息、加入购物车
- 登录/注册页：用户认证
- 购物车页：查看、修改、删除购物车商品
- 结算页：下单、查看订单
- 订单列表与详情页

### 7. 常见问题
- **端口冲突**：如 3000 被占用，可在 `package.json` 或启动命令中指定其他端口
- **接口跨域**：开发环境已通过 Vite 代理解决，无需额外配置
- **接口 404/500**：请确认后端服务已正确启动

如需更详细说明，请参考 `frontend/README.md`。

---

如有更多问题请联系开发者或在 Issues 区留言。
