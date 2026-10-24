# Chevereto-Free → Chevereto 4.5.7 升级记录

- 站点：`image.knebulae.com`
- 服务器：Windows Server + 宝塔面板（BtSoft）+ nginx
- 网站目录：`C:\wwwroot\image.knebulae.com`
- 记录日期：2026-10-23 ～ 2026-10-24
- 当前状态：**数据库迁移尚未完成**（见文末“当前进度与待办”）

---

## 1. 背景与目标

| 项目 | 升级前 | 升级目标 |
| --- | --- | --- |
| 程序 | Chevereto-Free 1.6.2（仅支持 PHP 7.4） | Chevereto 4.5.7 Free 版（AGPLv3） |
| PHP | 7.4 | 宝塔 PHP 8.2（已确认为 8.2.35） |
| 数据库 | 4 张表为 MyISAM | 全部 InnoDB |

升级原因：2026-10-21 PHP 发布紧急安全更新，修复 CVE-2026-109618、CVE-2026-109619（未授权远程代码执行，CVSS v4.0 9.2），修复版本为 8.5.12 / 8.4.27 / 8.3.36 / 8.2.35。PHP 7.4 无官方二进制修复。Chevereto 4.5 官方只支持 PHP 8.2，因此选用 8.2.35。

注意：PHP 8.2 于 2026-12-31 停止安全支持，届时需再评估（等待 Chevereto 支持 8.3+，或迁移到 Docker / Linux）。

---

## 2. 项目文件改动清单（重点）

### 2.1 Chevereto 源码补丁（Windows 兼容，升级 Chevereto 时会被覆盖，需重新打）

#### 补丁 A：`app\src\Config\HostConfig.php`

**问题**：`env.php` 中 `CHEVERETO_HOSTNAME_PATH` 设为 `'/'`，但运行时读到的值是 `\/`（Windows 路径分隔符被拼进去）。导致访问首页 `/` 时报：

```
LogicException: Invalid PHP front controller setup. Review your web server configuration.
```

诊断日志：`root='\\/' uri='/' raw='/' script='/index.php'`

**修改**：构造函数内增加规范化逻辑。

修改前：

```php
    public function __construct(
        private string $hostnamePath = '/',
        private string $hostname = 'localhost',
        private bool $isHttps = false,
    ) {
    }
```

修改后：

```php
    public function __construct(
        private string $hostnamePath = '/',
        private string $hostname = 'localhost',
        private bool $isHttps = false,
    ) {
        // Windows 补丁：把反斜杠统一成 /，并规范为 "/" 或 "/子目录/"
        $path = preg_replace('#/+#', '/', str_replace('\\', '/', $this->hostnamePath));
        $path = trim($path, '/');
        $this->hostnamePath = $path === '' ? '/' : '/' . $path . '/';
    }
```

结果：首页路由恢复正常。

#### 补丁 B：`app\src\Legacy\functions.php` 第 1147 行

**问题**：页面能渲染但无样式，浏览器报 `Not allowed to load local resource: file:///C:/wwwroot/image.knebulae.com/content/...`。原因是 `PATH_PUBLIC` 在 Windows 上为 `C:\wwwroot\image.knebulae.com/`（反斜杠），而资源磁盘路径为正斜杠形式，路径→URL 的替换匹配失败，磁盘路径被原样输出。

修改前：

```php
    define('PATH_PUBLIC', dirname(__DIR__, 3) . '/');
```

修改后：

```php
    define('PATH_PUBLIC', str_replace('\\', '/', dirname(__DIR__, 3)) . '/'); // Windows 补丁
```

结果：CSS / JS / 图片链接恢复为 `https://image.knebulae.com/...`，页面显示正常。

#### 临时诊断修改（已要求还原，需确认）

- `app\src\Legacy\G\Handler.php`：曾把 `throw new LogicException('Invalid PHP front controller setup...')` 改为输出 `root / uri / raw / script / path` 的诊断版本，用于定位补丁 A 的问题。**应已用备份还原。**
- `app\legacy\load\register-handlers.php`：曾建议在空白页时临时改为 `$doDebug = true;`。如果改过，**应已还原。**

### 2.2 配置文件 `app\env.php`

由旧的 `app\settings.php`（已改名为 `settings.php.bak`）改写而来。关键项：

```php
<?php
return [
    'CHEVERETO_DB_HOST' => 'localhost',
    'CHEVERETO_DB_PORT' => '3306',
    'CHEVERETO_DB_NAME' => '（库名）',
    'CHEVERETO_DB_USER' => '（用户）',
    'CHEVERETO_DB_PASS' => '（密码）',
    'CHEVERETO_DB_TABLE_PREFIX' => 'chv_',

    'CHEVERETO_HOSTNAME' => 'image.knebulae.com',
    'CHEVERETO_HOSTNAME_PATH' => '/',
    'CHEVERETO_HTTPS' => '1',

    'CHEVERETO_IMAGE_LIBRARY' => 'gd',
    'CHEVERETO_SESSION_SAVE_PATH' => '（可写的会话目录，使用正斜杠）',
    'CHEVERETO_ERROR_LOG' => '（日志文件路径）',
];
```

排查期间曾建议临时加入 `'CHEVERETO_DEBUG_LEVEL' => '3'`，**排查结束后须改回 `'1'` 或删除该行。**

### 2.3 nginx 站点配置（宝塔 vhost）

已应用的改动：

```nginx
index index.php;

location / {
    try_files $uri $uri/ /index.php$is_args$query_string;
}
```

建议加入宝塔“伪静态”（`rewrite/image.knebulae.com/*.conf`）的 Chevereto 4 访问限制（是否已应用待确认）：

```nginx
location ^~ /app/ {
    return 404;
}
location ^~ /importing/ {
    return 404;
}
location ^~ /content/ {
    location ~* \.(php[345]?|phtml|html?)$ {
        return 403;
    }
}
location ~* (composer\.json|composer\.lock|\.gitignore)$ {
    return 404;
}
location ~* \.(ttf|ttc|otf|eot|woff|woff2|css|js)$ {
    add_header Access-Control-Allow-Origin "*";
}
```

其他建议（待确认）：

- `location ^~ /images/` 中加入 `log_not_found off;` 和 `error_page 404 /content/images/system/default/404.gif;`
- `ssl_protocols` 改为 `TLSv1.2 TLSv1.3`
- 排查时如果加过 `allow 你的IP; deny all;`，**上线前须删除**

说明：宝塔的 `php/82.conf` 没有 `internal;`，直接访问 `/index.php` 会由 Chevereto 返回错误页，不影响使用。该文件由宝塔管理，不手动修改。

### 2.4 PHP（宝塔 PHP 8.2）

- 版本已确认为 8.2.35（含漏洞修复）
- 已确认未开启“防跨站攻击”（open_basedir），会话目录可写
- PHP 7.4 期间建议的临时缓解：`session.upload_progress.enabled = 0`、`post_max_size = 32M`

---

## 3. 数据库改动记录

### 3.1 前置条件：MyISAM → InnoDB

首次执行 `database-migrate` 时 Chevereto 提示必须转换存储引擎：

```sql
ALTER TABLE chv_albums ENGINE = InnoDB;
ALTER TABLE chv_images ENGINE = InnoDB;
ALTER TABLE chv_redirects ENGINE = InnoDB;
ALTER TABLE chv_users ENGINE = InnoDB;
```

### 3.2 迁移过程中发现的问题

1. **第一次迁移中途失败**：执行到写入 `chv_variables` 之后失败，未写入版本 4.5.7，维护模式停在 `1`。此时版本号停在 `1.6.2`，导致：
   - 登录时 Chevereto 走 3.14 之前的旧登录流程，报 `Key KEEP_LOGIN not found`
   - 前台显示“网站正在维护中”
2. **版本号存放位置**：V4 把 `chevereto_version_installed` 从 `chv_settings` 迁到 `chv_variables`，`chv_settings` 中残留的 `1.6.2` 属正常现象。判断迁移是否完成要看 `chv_variables`。
3. **在半迁移状态上重试导致混乱**：日志中只给出“剩余部分”SQL。后来数据库恢复到迁移前，执行这段“剩余 SQL”时报 `#1054 Unknown column 'album_cover_id'`。
4. **从干净的 1.6.2 状态重新运行**，拿到完整迁移 SQL，定位到**脚本自身冲突**（Chevereto 升级路径缺陷）：
   - 脚本先以最新结构 `CREATE TABLE` 三张 tags 表（已含主键和自动命名为 `*_ibfk_N` 的外键）；
   - 随后又执行 7 条 `ADD CONSTRAINT chv_tags_*_ibfk_N`（外键重名）；
   - 以及 3 条 `DROP INDEX tag_*_UNIQUE, ADD PRIMARY KEY`（索引不存在、主键已存在）。

### 3.3 修正方案：`C:\temp\migrate_fixed.sql`

取 `chevereto-error.log` 中完整迁移 SQL（两条 `---` 之间），**删除以下 10 条语句**后保存为 UTF-8：

```sql
ALTER TABLE `chv_tags_albums` ADD CONSTRAINT `chv_tags_albums_ibfk_1` ...;
ALTER TABLE `chv_tags_albums` ADD CONSTRAINT `chv_tags_albums_ibfk_2` ...;
ALTER TABLE `chv_tags_albums` ADD CONSTRAINT `chv_tags_albums_ibfk_3` ...;
ALTER TABLE `chv_tags_users`  ADD CONSTRAINT `chv_tags_users_ibfk_1` ...;
ALTER TABLE `chv_tags_users`  ADD CONSTRAINT `chv_tags_users_ibfk_2` ...;
ALTER TABLE `chv_tags_files`  ADD CONSTRAINT `chv_tags_files_ibfk_1` ...;
ALTER TABLE `chv_tags_files`  ADD CONSTRAINT `chv_tags_files_ibfk_2` ...;
ALTER TABLE `chv_tags_albums` DROP INDEX tag_album_UNIQUE, ADD PRIMARY KEY (...);
ALTER TABLE `chv_tags_files`  DROP INDEX tag_file_UNIQUE,  ADD PRIMARY KEY (...);
ALTER TABLE `chv_tags_users`  DROP INDEX tag_user_UNIQUE,  ADD PRIMARY KEY (...);
```

保留夹在中间的 `ALTER TABLE chv_images ADD image_delete_hash VARCHAR(255) NULL;`。

执行方式（mysql 命令行，遇错即停并显示行号）：

```
mysql -h 127.0.0.1 -u <CHEVERETO_DB_USER> -p --default-character-set=utf8mb4 <CHEVERETO_DB_NAME> < C:\temp\migrate_fixed.sql
```

出错处理规则：

| 错误码 | 处理 |
| --- | --- |
| 1091 / 1060 / 1061 / 1826 | 改动已存在：删除文件第 1 行至出错行，重新执行 |
| 1833 且为 `chv_images` / `chv_albums` / `chv_users` 的 `MODIFY ..._id` | 脚本前面已修改过：同上处理 |
| 其他 | 停止并记录错误 |

---

## 4. 备份清单

| 文件 | 内容 | 用途 |
| --- | --- | --- |
| 升级前备份（阶段三） | Chevereto-Free 1.6.2 原始数据库 + 网站文件 | 完整回滚基线 |
| `D:\backup\chevereto_halfway.sql` | 第一次迁移失败后的半迁移状态 | 排查参考 |
| `D:\backup\chevereto_clean_innodb.sql` | 干净的 1.6.2 数据，已转 InnoDB | **重试迁移的起点** |
| `HostConfig.php`、`functions.php`、`Handler.php` 原文件备份 | 打补丁前的源码 | 还原或对照 |

---

## 5. 常用诊断命令

```
:: 数据库迁移（完整输出含错误）
cd /d C:\wwwroot\image.knebulae.com
C:\BtSoft\php\82\php.exe -c C:\BtSoft\php\82\php.ini app\bin\cli -C database-migrate > C:\temp\migrate.txt 2>&1
echo %ERRORLEVEL%

:: 首页状态码
curl.exe -sS -o NUL -w "%{http_code}\n" https://image.knebulae.com/
```

```sql
-- 迁移是否完成
SELECT variable_value FROM chv_variables WHERE variable_name = 'chevereto_version_installed';
SELECT setting_value FROM chv_settings WHERE setting_name = 'maintenance';

-- 非 InnoDB 的表
SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE() AND ENGINE <> 'InnoDB';
```

---

## 6. 当前进度与待办

当前卡在：执行 `migrate_fixed.sql` 时 mysql 报 `ERROR 1045 Access denied for user 'root'@'localhost'`，是账户密码问题。改用 `env.php` 中的 Chevereto 数据库账户，或在宝塔“数据库 → root 密码”中查看。

待办：

- [ ] 把数据库恢复到 `chevereto_clean_innodb.sql`
- [ ] 用正确账户执行 `migrate_fixed.sql`，按规则处理报错
- [ ] 确认 `chv_variables` 版本为 `4.5.7`，`maintenance` 为 `0`
- [ ] 再跑一次 `database-migrate`，确认无新 SQL、无新错误
- [ ] 重启宝塔 PHP 8.2，清除浏览器 cookie，测试登录（`KEEP_LOGIN` 报错应消失）
- [ ] 确认 `Handler.php`、`register-handlers.php` 已还原，`CHEVERETO_DEBUG_LEVEL` 已改回 `1`
- [ ] 删除 nginx 临时 allow/deny；确认伪静态访问限制已生效
- [ ] 按验证清单检查：上传、缩略图、旧图直链、后台设置、`/app/` 返回 404
- [ ] 设置计划任务：`C:\BtSoft\php\82\php.exe app\bin\cli -C cron`（每分钟）
- [ ] 向 chevereto/chevereto 报告两个问题：Windows 路径分隔符、Free 1.6.2 → 4.5.7 迁移脚本外键/主键重复
- [ ] 每次升级 Chevereto 后重新应用补丁 A、B
