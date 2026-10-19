# penTools

Коллекция инструментов для пентеста. Каждый проект подключается как отдельный Git-сабмодуль.

## Проекты

- [bhCollector](https://github.com/ushst/bhCollector) — модуль NetExec со встроенным legacy-коллектором BloodHound.

## Клонирование

```bash
git clone --recurse-submodules https://github.com/ushst/penTools.git
```

Если репозиторий уже склонирован без сабмодулей:

```bash
git submodule update --init --recursive
```

## Обновление проектов

```bash
git submodule update --remote --merge
```

После обновления нужно зафиксировать новые версии сабмодулей обычным коммитом в `penTools`.
