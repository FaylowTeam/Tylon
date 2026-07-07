## Agent Guidance

- If the `acton` Codex skill is available in this environment, use it for Acton CLI, Tolk, wrappers, tests, scripts, deployment, and `Acton.toml` tasks.
- If the `acton` skill is not available, continue without it. Do not block on installation and do not assume network access is available.
- Treat `contracts/types.tolk` and `contracts/Empty.tolk` as the source of truth for storage, messages, and ABI-facing behavior.
- Keep `wrappers/Empty.gen.tolk`, `tests/contract.test.tolk`, and `scripts/deploy.tolk` aligned with contract changes.
- When ABI changes are involved, prefer regenerating the wrapper with `acton wrapper Empty` over hand-editing the wrapper file.
- Prefer this validation loop when feasible: `acton build`, `acton test`, `acton run deploy-emulation`.
- Before proposing broadcast deployment changes, verify the script in emulation first.
- When command syntax or flags are unclear, verify them with `acton --help` or `acton <command> --help`.

## Работа с Git и GitHub
### Коммиты
После выполнения задачи делай коммит с сообщением формата `[AI] ОПИСАНИЕ ИЗМЕНЕНИЙ`.

### Ветки Git
В репозитории есть две основные ветки. `main` - это основной стабильный релиз. Не используется в основной разработке. `dev` - основная эспериментальная ветка разработки, куда заливаются все изменения. При разработке обязательно нужно создавать отдельную ветку из `dev` по шаблону `префикс/название_постфикс`. Например, в ветке `bugfix/tests_ai`, где `bugfix/` означает, что в этой ветке фиксится баг, `tests` - это название ветки, показывающее, что работа ведется над автотестами и постфикс `_ai` говорит, что в ветке работает AI.

### Issues
При создании Issues через AI обязательно добавляй маркер `[AI]` в заголовок задачи. Формат: `[AI] Описаниe задачи`.

### Pull Requests
После выполнения задачи необходимо создавать Pull Request с созданными изменениями. Оформляй правильно и подробно, как это делают в больших корпах, обязательно добавляй в название маркер [AI]. Pull request делай в ветку `dev`.