# Справочные материалы

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните резервную копию сетевого файла или профиля.
- Запишите текущие `ip a` и `ip route` (Windows: `Get-NetIPAddress` и `Get-NetRoute`).
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую SSH/RDP-сессию открытой и проверяйте новую сессию отдельно.

> Настройка сети может прервать удалённый доступ. Если нет работающей консоли, сначала обеспечьте её доступность.

Документация составлена с опорой на официальные руководства. Проверяйте применимость команд к установленной версии и собственному образу.

- [Debian Handbook: network configuration](https://www.debian.org/doc/manuals/debian-handbook/sect.network-config.en.html)
- [Debian Reference: network setup](https://www.debian.org/doc/manuals/debian-reference/ch05.en.html)
- [Microsoft New-NetIPAddress](https://learn.microsoft.com/en-us/powershell/module/nettcpip/new-netipaddress)
- [Microsoft Get-NetIPAddress](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-netipaddress)

