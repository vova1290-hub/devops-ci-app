# devops-ci-app

Учебное Flask-приложение для практической работы № 10.
CI: Pylint → pytest → Docker build → smoke-test → GHCR.
Публикация образа выполняется только из `main`. В Pull Request выполняются проверки без push.

- health endpoint: `/health`;
- образ: `ghcr.io/vova1290-hub/devops-ci-app`;
- основной workflow: `.github/workflows/ci.yml`.

Автор: Беломытцев Владимир Алексеевич, ЭФБО-07-24.
