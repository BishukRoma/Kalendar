# Матрица автономности

> Правила поведения роя для этого проекта.
> Формат: [действие]: AUTO / ASK / USER / BLOCK

## Общие правила
- commit code: AUTO
- push to main: AUTO (с коротким сообщением на английском)
- push to feature branch: AUTO
- delete files/data: USER
- install dependencies: USER
- change architecture/tech-stack: USER
- breaking changes to API: USER

## Агенты
### Gosha (разработка)
- implement features: AUTO
- fix bugs: AUTO
- refactor code: AUTO
- run tests: AUTO

### Rex (приёмка)
- code review: AUTO
- approve PRs: AUTO

### Orion (продукт)
- define requirements: USER
- prioritize tasks: ASK

### Iris (дизайн)
- create UI mockups: AUTO
- design system updates: USER

### Titan (деплой)
- setup CI/CD: USER
- deploy to production: USER
