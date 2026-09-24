# Windows 10 GPT: дополнительный IPv4

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните резервную копию сетевого файла или профиля.
- Запишите текущие `ip a` и `ip route` (Windows: `Get-NetIPAddress` и `Get-NetRoute`).
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую SSH/RDP-сессию открытой и проверяйте новую сессию отдельно.

> Настройка сети может прервать удалённый доступ. Если нет работающей консоли, сначала обеспечьте её доступность.

## Посмотрите текущие параметры

Откройте PowerShell от имени администратора:

```powershell
Get-NetAdapter
Get-NetIPAddress -AddressFamily IPv4 | Format-Table InterfaceAlias,IPAddress,PrefixLength,SkipAsSource
Get-NetRoute -AddressFamily IPv4 | Sort-Object RouteMetric | Format-Table InterfaceAlias,DestinationPrefix,NextHop,RouteMetric
Get-DnsClientServerAddress -AddressFamily IPv4
Get-NetIPInterface -AddressFamily IPv4 | Format-Table InterfaceAlias,Dhcp
```

Найдите `<INTERFACE>` с `<MAIN_IP>`. Сохраните вывод в файл или скриншот. Конфигурация хранится в настройках адаптера Windows, единого файла вроде Linux YAML нет. Экспортируйте текущие настройки для backup:

```powershell
Get-NetIPAddress -InterfaceAlias "<INTERFACE>" | Export-Clixml C:\ip-before.xml
Get-NetRoute -InterfaceAlias "<INTERFACE>" | Export-Clixml C:\routes-before.xml
```

## Вариант 1: графический интерфейс

1. Откройте VNC/веб-консоль и `Win+R` -> `ncpa.cpl`.
2. `Ethernet` (или ваш адаптер) -> `Properties` -> `Internet Protocol Version 4 (TCP/IPv4)` -> `Properties` -> `Advanced`.
3. В разделе **IP addresses** нажмите **Add**, введите `<ADDITIONAL_IP>` и маску, соответствующую выданному `<PREFIX>`. Сохраните окна.
4. Не удаляйте `<MAIN_IP>` и не меняйте действующий gateway. В разделе gateway новый адрес обычно не требуется.

Если основной IPv4 получен через DHCP, не переключайте адаптер на статическую адресацию и не выполняйте приведённую ниже команду: она отключит DHCP. Сначала уточните у поддержки безопасную схему для этого образа.

## Вариант 2: PowerShell

> Выполняйте команду только если `Get-NetIPInterface` показывает `Dhcp Disabled` для IPv4 на выбранном адаптере. `New-NetIPAddress` автоматически отключает DHCP, если он включён, что может убрать основной адрес и доступ.

```powershell
New-NetIPAddress -InterfaceAlias "<INTERFACE>" -IPAddress "<ADDITIONAL_IP>" -PrefixLength <PREFIX> -SkipAsSource $true
Get-NetIPAddress -InterfaceAlias "<INTERFACE>" -AddressFamily IPv4
```

`-SkipAsSource $true` помогает сохранить выбор основного IP для обычных исходящих соединений. Новый gateway **не указывайте**, если провайдер не выдал отдельную схему маршрутизации. При DHCP эту команду не применяйте.

## Проверка

```powershell
Get-NetIPAddress -InterfaceAlias "<INTERFACE>" -AddressFamily IPv4
Get-NetRoute -AddressFamily IPv4
Test-Connection <GATEWAY> -Count 3
Invoke-RestMethod https://api.ipify.org
```

С другой машины проверьте `ping <ADDITIONAL_IP>` и `Test-NetConnection <ADDITIONAL_IP> -Port 22` для SSH либо `-Port 3389` для RDP, если он доступен и сервис установлен, запущен и разрешён firewall. ICMP может быть запрещён. Исходящий IP может остаться основным.

## Откат и потеря доступа

Из VNC/веб-консоли удалите **только** добавленный адрес в `Advanced` либо:

```powershell
Remove-NetIPAddress -InterfaceAlias "<INTERFACE>" -IPAddress "<ADDITIONAL_IP>" -Confirm:$false
```

Не удаляйте `<MAIN_IP>`. Если пропал RDP/SSH, войдите через консоль, сверьте сохранённые параметры и удалите ошибочно добавленный адрес. [Диагностика](../troubleshooting.md).

