### General

- **Enabled**
   勾选表示启用这个 Job，如果取消勾选，Job 不会被触发。
- **描述**
   可以填写对这个 Job 的说明、用途或备注信息。

------

### GitLab Connection

- **gitlab-lxm**
   指定这个 Job 使用哪个 GitLab 连接，通常在 **Jenkins 系统配置**中提前配置了 GitLab 访问令牌。
- **Use alternative credential**
   勾选后可以为这个 Job 指定单独的 GitLab 凭据，而不使用全局默认凭据。
- **Permission to Copy Artifact**
   是否允许这个 Job 复制其他 Job 的构建产物。
- **Throttle builds**
   限制并发构建的数量。例如：防止某个 Job 同时被触发多次导致系统资源过载。
- **丢弃旧的构建（Discard old builds）**
   勾选表示自动删除旧的构建记录，避免占用过多磁盘。

------

### 策略（Log Rotation）

- **保持构建的天数**
   如果填写，Jenkins 会自动删除超过指定天数的构建记录。
- **保持构建的最大个数**
   如果填写，Jenkins 会自动删除超过指定数量的构建记录，只保留最新的几次。

------

### 高级

通常可以配置：

- **参数化构建**（Build with parameters）
- **触发器**（Triggers，例如定时触发、GitLab webhook）
- **构建环境**（Build Environment，例如清理工作空间、注入环境变量）
- **构建步骤**（Build Steps，例如执行 Shell、调用 Maven/Gradle）
- **构建后操作**（Post-build Actions，例如归档构建产物、触发其他 Job、发送通知）