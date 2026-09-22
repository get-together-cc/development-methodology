# 安全审计清单（交付前逐项检查）

> 可勾选清单，辅助多模型审核提示词。不是替代审核，是让审核聚焦在容易漏的检查项上。

## 通用检查

- [ ] 所有用户输入已按场景 sanitize（`sanitize_text_field` / `intval` / `wp_kses_post` / 参数强转）
- [ ] 期望为字符串的数组参数做了 `(string)` 强转，防 TypeError 500
- [ ] Nonce 双检：`wp_nonce_field()` + `check_admin_referer()` 或 `check_ajax_referer()`
- [ ] 每个 handler/API endpoint 入口有权检查（`current_user_can()`）
- [ ] SQL 用 prepare() 而非字符串拼接
- [ ] AJAX handler 加了 `check_ajax_referer()` + `die()` / `wp_die()`
- [ ] 表单按钮显式 `value="1"` + 后端用 `isset()` 而非 `empty()`（按钮 value 默认空串陷阱）
- [ ] 文件上传验证了类型/大小/MIME，文件名已 sanitize
- [ ] PHP 8.0+ 兼容：无 deprecated 语法
- [ ] 敏感路径已加 deny 访问保护

## WordPress 特有

- [ ] CPT capabilities 未滥用 `manage_options` 等共享 cap
- [ ] 自定义 cron 任务：安装时注册、卸载时 `wp_clear_scheduled_hook()` 清除
- [ ] options 卸载时清理干净（不残留）
- [ ] 跨站 ACF 同步不直接传 image ID（各站 media ID 不兼容）
- [ ] 序列化数据用安全脚本更新，不直接字符串 REPLACE

## Go 微服务

- [ ] HTTP handler 检查方法：`r.Method != http.MethodPost`
- [ ] 无硬编码凭据/密钥——走环境变量或配置文件
- [ ] systemd WritePaths 目录预先 `mkdir`
- [ ] 无残留的 heredoc 部署脚本（改用 base64 管道或文件模式）
- [ ] 考虑环境代理劫持（`Proxy: nil` 检查）

## 凭据泄露检查

- [ ] 快照/提交前检查无 `.pem` / `.key` / `*login*` / `.env` / `wp-config`（类型黑名单）
- [ ] 代码内无 `api_key` / `password` / `secret` / `BEGIN PRIVATE` 字面量
- [ ] 敏感文件不走云端审核（→ 本地模型审）