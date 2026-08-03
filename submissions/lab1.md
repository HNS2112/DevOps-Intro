# Lab 1 — DevOps Foundations: Fork, Sign, and Open Your First PR

**Author:** Elvira ([@HNS2112](https://github.com/HNS2112))
**Date:** 3 August 2026
**Environment:** macOS 26.3.1 (Apple Silicon), Git 2.55.0, Go 1.26.5

---

## Task 1 — SSH Commit Signing & First Signed Commit

### 1.2 QuickNotes running locally

```console
$ curl -s http://localhost:8080/health | python3 -m json.tool
{
    "notes": 4,
    "status": "ok"
}

$ curl -s http://localhost:8080/notes | python3 -m json.tool
[
    {
        "id": 3,
        "title": "DevOps mantra",
        "body": "If it hurts, do it more often.",
        "created_at": "2026-01-15T10:10:00Z"
    },
    {
        "id": 4,
        "title": "Endpoint cheat-sheet",
        "body": "GET /notes  GET /notes/{id}  POST /notes  DELETE /notes/{id}  GET /health  GET /metrics",
        "created_at": "2026-01-15T10:15:00Z"
    },
    {
        "id": 1,
        "title": "Welcome to QuickNotes",
        "body": "This is the project you'll containerize, deploy, monitor, and harden across all 10 labs.",
        "created_at": "2026-01-15T10:00:00Z"
    },
    {
        "id": 2,
        "title": "Read app/main.go first",
        "body": "Start by understanding the entry point \u2014 env vars, signal handling, graceful shutdown.",
        "created_at": "2026-01-15T10:05:00Z"
    }
]

$ curl -s -X POST http://localhost:8080/notes -H 'Content-Type: application/json' \
    -d '{"title":"hello","body":"first POST"}' | python3 -m json.tool
{
    "id": 5,
    "title": "hello",
    "body": "first POST",
    "created_at": "2026-08-03T19:28:02.543864Z"
}

$ curl -s http://localhost:8080/notes | python3 -c 'import sys,json;print(len(json.load(sys.stdin)))'
5
```

`/notes` возвращает 4 seed-заметки при первом запросе и 5 после `POST /notes` —
новой записи присвоен `id: 5`.

Два наблюдения по ходу запуска:

1. **Сид загружается относительно рабочей директории.** В `main.go` путь берётся
   как `envOrDefault("SEED_PATH", "seed.json")`, поэтому запуск собранного
   бинарника из корня репозитория даёт пустое хранилище без единой ошибки —
   `ensureSeeded` молча не находит файл. Корректный запуск — из `app/`, как и
   указано в методичке. Это хороший пример того, почему в контейнере рабочая
   директория задаётся явно через `WORKDIR`, а не подразумевается.
2. **Порядок выдачи `GET /notes` не детерминирован** — заметки вернулись как
   3, 4, 1, 2. Хранилище итерируется по Go-мапе, а рандомизация порядка обхода
   там сделана намеренно, чтобы клиенты не полагались на неявные гарантии.
   Для API это означает: сортировку нужно задавать явно, иначе любой тест на
   порядок элементов будет флаки.

### 1.4 Signed commit verification

```console
$ git log --show-signature -1
commit 70e9954b31d187134b9fab795739c21e05143f3a
Good "git" signature for 239804565+HNS2112@users.noreply.github.com with ED25519 key SHA256:w1hbjtty/1UU7w1K0zNjnqtEwR/cfaBaCw8NLHuSF0o
Author: Elvira <239804565+HNS2112@users.noreply.github.com>
Date:   Mon Aug 3 22:31:30 2026 +0300

    docs(lab1): start submission
    
    Signed-off-by: Elvira <239804565+HNS2112@users.noreply.github.com>
```

Строка `Good "git" signature for ...` подтверждает: коммит подписан приватным
ключом, публичная часть которого зарегистрирована в моём `allowed_signers` /
на стороне GitHub как **Signing Key**.

### 1.5 Verified badge

![Verified badge](../evidence/screenshots/verified-badge.png)
<!-- Скриншот: страница коммита на GitHub, зелёный значок Verified -->

### Почему подписанные коммиты имеют значение

В марте 2024 года в `xz-utils` (CVE-2024-3094) обнаружили бэкдор, дававший
удалённое выполнение кода через `sshd` на дистрибутивах, линкующих `liblzma`.
Ключевая деталь: злонамеренный код отсутствовал в git-репозитории — он жил в
тестовых фикстурах внутри release-тарболов, собираемых мейнтейнером вручную.
Именно расхождение между тем, что видно в истории, и тем, что реально
поставляется, позволило атаке прожить месяцы.

Честная оценка: криптоподпись коммитов **не остановила бы** эту конкретную
атаку — «Jia Tan» два года выстраивал репутацию и обладал легитимными правами
мейнтейнера, его коммиты подписались бы корректно. Ценность подписи в другом:
она делает каждое изменение неотрицаемо атрибутируемым и превращает
supply-chain-инцидент из «кто-то что-то подменил» в конкретный аудируемый
след. Без подписи достаточно подделать `user.email` в `git config`, чтобы
коммит выглядел авторства любого разработчика в проекте — а это как раз тот
слой доверия, на котором держатся code review и blame-driven расследование.

---

## Task 2 — Pull Request Template & First PR

### 2.1 Template

Файл `.github/pull_request_template.md` добавлен в **default branch** (`main`)
до открытия PR — GitHub читает темплейты только оттуда. Путь и регистр важны:
`pull_request_template.md` (единственное число) либо `PULL_REQUEST_TEMPLATE.md`.

```markdown
## Goal
## Changes
## Testing
## Checklist
- [ ] Title is a clear sentence (≤ 70 chars)
- [ ] Commits are signed (`git log --show-signature`)
- [ ] `submissions/labN.md` updated
```

### 2.2 PR

**PR URL:** https://github.com/inno-devops-labs/DevOps-Intro/pull/1487

![PR auto-populated](../evidence/screenshots/pr-template.png)
<!-- Скриншот: форма создания PR с уже подставленными секциями -->

Все коммиты в PR имеют badge **Verified**, чекбоксы отмечены.

---

## Task 3 — GitHub Community

**Starred:** `inno-devops-labs/DevOps-Intro`, `simple-container-com/api`
**Following:** @Cre-eD, @Naghme98, @pierrepicaud + @G-Akleh, @Dnau15, @Ephy01

Звезда — это не только закладка: суммарный star count де-факто работает
сигналом доверия при выборе зависимости, а публичный список звёзд в профиле
показывает технический профиль разработчика точнее, чем строчка в резюме.
Подписки на людей превращают ленту в пассивный канал обучения — видно, какие
инструменты реально применяют коллеги, и это тот же граф связей, который
позже даёт ревьюеров, соавторов и рекомендации при найме.

---

## Bonus Task — Branch Protection & Required Signed Commits

### B.1 Rules

![Branch protection](../evidence/screenshots/branch-protection.png)

Включено на `main` моего форка:
- Require signed commits
- Require a pull request before merging
- Require linear history
- Include administrators (иначе правило обходится собственным владельцем)

### B.2 Rejection

```console
$ git -c commit.gpgsign=false commit --no-gpg-sign -s --allow-empty \
    -m "test: unsigned commit (should fail)"
$ git push origin main
remote: error: GH006: Protected branch update failed for refs/heads/main.        
remote: 
remote: - Commits must have verified signatures.        
remote:   Found 1 violation:        
remote: 
remote:   29421653cbdb10f2f9ed4b97ce52a4e28fd5fab0        
remote: 
remote: - Changes must be made through a pull request.        
To github.com:HNS2112/DevOps-Intro.git
 ! [remote rejected] main -> main (protected branch hook declined)
error: failed to push some refs to 'github.com:HNS2112/DevOps-Intro.git'
```

> Примечание: в методичке приведено `git commit -S=false` — такой синтаксис Git
> не принимает (`-S` не берёт значение через `=`). Рабочий вариант —
> `--no-gpg-sign` либо `git -c commit.gpgsign=false commit`.

### B.3 Reflection — Knight Capital

1 августа 2012 года Knight Capital потеряла ~$440 млн примерно за 45 минут:
релиз раскатили вручную на восемь серверов, на одном код не обновился, и там
переиспользованный feature-флаг реактивировал восьмилетний мёртвый код
Power Peg, начавший слать ошибочные ордера.

Прямо branch protection и обязательная подпись эту аварию не предотвратили бы —
корневые причины лежали в ручном деплое без верификации, отсутствии
контроля идентичности артефакта на всех нодах и в переиспользовании флага под
новую семантику. Но косвенный эффект существенный: обязательный PR перед
мержем в prod-ветку заставил бы вторую пару глаз увидеть повторное
использование флага, требование linear history + подписи дало бы однозначный
ответ «какой именно коммит сейчас на каждом сервере» — а команда потратила
значительную часть тех 45 минут именно на выяснение, что развёрнуто и откуда.
Защита ветки не заменяет автоматизированный деплой, но делает состояние
системы верифицируемым, а это предусловие быстрого отката.

---

## Summary

| Task | Status |
|------|--------|
| Task 1 — SSH signing + QuickNotes | ✅ |
| Task 2 — PR template + first PR | ✅ |
| Task 3 — Community engagement | ✅ |
| Bonus — Branch protection | ✅ |
