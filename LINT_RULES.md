# Bộ rule lint / format — spec để sinh config sau

Mục tiêu: mỗi ngôn ngữ **một linter + một formatter**, dùng preset chuẩn của cộng đồng, chỉ override vài chỗ. Khi sinh config từ file này: giữ nguyên preset, chỉ thêm các override ghi dưới đây.

## Quy ước chung (mọi ngôn ngữ)

| Mục | Giá trị |
|---|---|
| Thụt lề | 2 space (Ruby, JS/TS, YAML, JSON, HTML, CSS); 4 space (Python); tab (Makefile, Go) |
| Độ dài dòng | 100 (Python, Ruby); 100 cho Prettier |
| Kết thúc dòng | LF, có newline cuối file, xóa space thừa cuối dòng |
| Encoding | UTF-8 |
| Quote | Ruby/JS: single quote; Python: double quote (theo Ruff format) |

File chung: `.editorconfig` ở root, mã hóa đúng bảng trên.

## Ruby — RuboCop

- Gem: `rubocop`, `rubocop-performance`; thêm `rubocop-rails` nếu là Rails, `rubocop-rspec` nếu dùng RSpec.
- File: `.rubocop.yml`. Nạp plugin bằng `plugins:` (RuboCop ≥ 1.72), không dùng `require:`.
- `AllCops`: `NewCops: enable`, `TargetRubyVersion` theo `.ruby-version`, exclude `bin/`, `db/schema.rb`, `vendor/`, `node_modules/`, `tmp/`.
- Override:
  - `Layout/LineLength: Max: 100`
  - `Style/Documentation: Enabled: false`
  - `Style/FrozenStringLiteralComment: EnforcedStyle: always`
  - `Metrics/MethodLength: Max: 15`, `Metrics/AbcSize: Max: 20`
  - `Metrics/BlockLength`: exclude `spec/**/*`, `config/routes.rb`
- Repo cũ: chạy `rubocop --auto-gen-config` để sinh `.rubocop_todo.yml`, rồi giảm dần.
- Lệnh: `bundle exec rubocop`, sửa tự động `bundle exec rubocop -a`.

## JavaScript / TypeScript — ESLint + Prettier

- ESLint 9, **flat config** `eslint.config.js` (không dùng `.eslintrc*`).
- Preset: `@eslint/js` recommended; TS thì thêm `typescript-eslint` recommended (type-checked nếu có `tsconfig`); cuối cùng `eslint-config-prettier` để tắt rule đụng Prettier.
- Framework (chỉ thêm khi dùng): `eslint-plugin-react` + `eslint-plugin-react-hooks`; Vue: `eslint-plugin-vue`.
- Override:
  - `no-unused-vars` (hoặc bản TS): error, bỏ qua biến bắt đầu bằng `_`
  - `eqeqeq: error`, `no-console: warn` (trừ script/CLI)
  - `prefer-const: error`
- Prettier: `.prettierrc.json` → `{ "singleQuote": true, "semi": true, "printWidth": 100, "trailingComma": "all" }`; `.prettierignore` theo `.gitignore`.
- Lệnh: `npx eslint .`, `npx prettier --check .` (sửa: `--fix` / `--write`).

## Python — Ruff (lint + format)

- Một tool `ruff` thay cho flake8 + isort + black + pyupgrade.
- File: `pyproject.toml` mục `[tool.ruff]` (hoặc `ruff.toml` nếu project không có pyproject).
- `line-length = 100`, `target-version` theo Python đang dùng.
- `[tool.ruff.lint] select = ["E", "W", "F", "I", "B", "UP", "SIM", "N", "S"]`
  - `S` (bandit) quan trọng cho code edge/Modbus: bắt `eval`, `subprocess` shell=True, bind `0.0.0.0`...
  - `per-file-ignores`: `tests/**` bỏ `S101` (assert).
- Type check (tùy chọn, bật sau): `mypy` hoặc `pyright` ở mức basic.
- Lệnh: `ruff check .`, `ruff format --check .` (sửa: `ruff check --fix .`, `ruff format .`).

## Các file khác

| Loại | Tool | File config | Ghi chú |
|---|---|---|---|
| Shell | `shellcheck` + `shfmt` | `.shellcheckrc` | `shfmt -i 2 -ci` |
| YAML | `yamllint` | `.yamllint.yml` | extends `default`, `line-length: max 120`, tắt `document-start` |
| Dockerfile | `hadolint` | `.hadolint.yaml` | bật `DL3008` (pin version apt) ở mức warning |
| Markdown | `markdownlint-cli2` | `.markdownlint.jsonc` | tắt `MD013` (line length), `MD033` (HTML inline) |
| CSS | `stylelint` | `stylelint.config.js` | extends `stylelint-config-standard` (chỉ khi CSS nhiều lên) |

## Chạy tự động

- **pre-commit** (`.pre-commit-config.yaml`) ở root, gom hook: ruff, ruff-format, shellcheck, yamllint, hadolint, markdownlint, end-of-file-fixer, trailing-whitespace. Ruby/JS chạy qua hook `local` gọi `bundle exec rubocop` / `npx eslint` để dùng đúng version trong project.
- **CI**: một job `lint` chạy `pre-commit run --all-files`; fail là chặn merge.
- **Editor**: khuyến nghị extension Ruff, ESLint, Prettier, RuboCop (Ruby LSP), EditorConfig; bật format-on-save.

## Thứ tự làm khi vibe code

1. `.editorconfig` + `.pre-commit-config.yaml` (hook chung).
2. Ruff cho `modbus-lab` và `demo-a04` (phần lớn code là Python) — chạy `--fix`, commit riêng.
3. yamllint, hadolint, shellcheck cho file hạ tầng hiện có.
4. ESLint + Prettier khi phần JS lớn hơn vài file.
5. RuboCop khi có project Ruby.
