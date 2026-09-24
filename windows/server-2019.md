# Windows Server 2019: дополнительный IPv4

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните текущие параметры адаптера в файл или сделайте скриншоты.
- Запишите вывод `Get-NetIPAddress`, `Get-NetRoute` и состояние DHCP.
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую RDP-сессию открытой и проверяйте новую отдельно.

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

Найдите `<INTERFACE>` с `<MAIN_IP>`. Сохраните вывод в файл или скриншот. Конфигурация хранится в настройках адаптера Windows, отдельного сетевого файла для редактирования нет. Экспортируйте текущие настройки для backup:

```powershell
Get-NetIPAddress -InterfaceAlias "<INTERFACE>" | Export-Clixml C:\ip-before.xml
Get-NetRoute -InterfaceAlias "<INTERFACE>" | Export-Clixml C:\routes-before.xml
```

## Вариант 1: графический интерфейс

1. Откройте VNC/веб-консоль и `Win+R` -> `ncpa.cpl`.
2. `Ethernet` (или ваш адаптер) -> `Properties` -> `Internet Protocol Version 4 (TCP/IPv4)` -> `Properties` -> `Advanced`.
3. В разделе **IP addresses** нажмите **Add**, введите `<ADDITIONAL_IP>` и маску, соответствующую выданному `<PREFIX>`. Сохраните окна.
4. Не удаляйте `<MAIN_IP>` и не меняйте действующий gateway. В разделе gateway новый адрес обычно не требуется.

До изменения проверьте `Get-NetIPInterface`. Если основной IPv4 получен через DHCP, не переключайте адаптер на статическую адресацию ни в этом окне, ни через PowerShell: можно потерять основной адрес и доступ. Сначала уточните у поддержки безопасную схему для этого образа.

## Вариант 2: PowerShell

> Выполняйте команду только если `Get-NetIPInterface` показывает `Dhcp Disabled` для IPv4 на выбранном адаптере. `New-NetIPAddress` автоматически отключает DHCP, если он включён, что может убрать основной адрес и доступ.

```powershell
New-NetIPAddress -InterfaceAlias "<INTERFACE>" -IPAddress "<ADDITIONAL_IP>" -PrefixLength <PREFIX> -SkipAsSource $true
Get-NetIPAddress -InterfaceAlias "<INTERFACE>" -AddressFamily IPv4
```

`-SkipAsSource $true` помогает сохранить выбор основного IP для обычных исходящих соединений. Новый gateway **не указывайте**, если провайдер не выдал отдельную схему маршрутизации. При DHCP эту команду не применяйте.

## Проверка

```powershell
Get-NetIPAddress -InterfaceAlias "<INTERFACE>" -AddressFamily IPv4 | Format-Table IPAddress,PrefixLength,AddressState,SkipAsSource
Get-NetRoute -AddressFamily IPv4
Test-Connection <GATEWAY> -Count 3
Invoke-RestMethod https://api.ipify.org
```

С другой машины проверьте `ping <ADDITIONAL_IP>` и `Test-NetConnection <ADDITIONAL_IP> -Port 22` для SSH либо `-Port 3389` для RDP, если сервис установлен, запущен и разрешён firewall. ICMP может быть запрещён. Исходящий IP может остаться основным.

## Откат и потеря доступа

Из VNC/веб-консоли удалите **только** добавленный адрес в `Advanced` либо:

```powershell
Remove-NetIPAddress -InterfaceAlias "<INTERFACE>" -IPAddress "<ADDITIONAL_IP>" -Confirm:$false
```

Не удаляйте `<MAIN_IP>`. Если пропал RDP/SSH, войдите через консоль, сверьте сохранённые параметры и удалите ошибочно добавленный адрес. [Диагностика](../troubleshooting.md).

