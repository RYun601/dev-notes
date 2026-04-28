# 适用于 golang 的AGENTS.md

## 文档与日志要求
- `AGENTS.md`：仅用于开发规范/协作规则，不记录过程日志
- 过程日志：统一写入 `RELEASE.md`。每次开发/部署/测试后必须更新（如果只是调整文档，不涉及代码则不用），记录：
  - 变更摘要
  - 影响范围
  - 冒烟测试与结果（含命令与关键输出）
  - 未完成事项或风险
- view/frontend/下的每次开发必须单独更新 view/frontend/AGENTS.md，记录：
  - 变更摘要
  - 影响范围
  - 接了后端api，要补充详细的后端接口文档
  - 冒烟测试与结果（含命令与关键输出）
  - 未完成事项或风险
- view/admin-frontend/下的每次开发必须单独更新 view/admin-frontend/AGENTS.md，记录：
  - 变更摘要
  - 影响范围
  - 接了后端api，要补充详细的后端接口文档
  - 冒烟测试与结果（含命令与关键输出）
  - 未完成事项或风险

- `README.md`：记录运维层说明与更新日志（发布/升级/回滚/环境变更）
- 如果有定时任务需要执行，应该在 `README.md` 中注明调用方式和频率

## 提交前检查命令（必须执行并在 RELEASE.md 留痕）
- Model 字段注释扫描（结果必须为空）：
  - `rg --line-number -P 'gorm:"(?![^"]*comment:)[^"]*"' backend/internal/model | rg -v 'gorm:"-"'`
- Go 小数字段浮点类型扫描（结果必须为空或经评审确认为非业务小数）：
  - `rg --line-number '\bfloat32\b|\bfloat64\b' backend/internal/model backend/internal/service backend/internal/repository`
- PostgreSQL 缺失表注释检查（结果必须 0 行）
- PostgreSQL 缺失字段注释检查（结果必须 0 行）
- 过程日志要求：
  - `RELEASE.md` 必须记录以上命令与关键输出（含 0 行或通过结论）

### 缺失注释检查 SQL（PostgreSQL）
- 缺失表注释检查：
  ```sql
  SELECT n.nspname, c.relname
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname = 'public'
    AND COALESCE(BTRIM(obj_description(c.oid, 'pg_class')), '') = '';
  ```
- 缺失字段注释检查：
  ```sql
  SELECT n.nspname, c.relname, a.attname
  FROM pg_attribute a
  JOIN pg_class c ON c.oid = a.attrelid
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname = 'public'
    AND a.attnum > 0
    AND NOT a.attisdropped
    AND COALESCE(BTRIM(col_description(a.attrelid, a.attnum)), '') = '';
  ```
