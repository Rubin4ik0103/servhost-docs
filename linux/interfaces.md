# Linux networking: /etc/network/interfaces

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните резервную копию сетевого файла или профиля.
- Запишите текущие `ip a` и `ip route` (Windows: `Get-NetIPAddress` и `Get-NetRoute`).
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую SSH/RDP-сессию открытой и проверяйте новую сессию отдельно.

> Настройка сети может прервать удалённый доступ. Если нет работающей консоли, сначала обеспечьте её доступность.

## Найдите файл и блок интерфейса

```bash
ip -br a
ip route
sudo cat /etc/network/interfaces
sudo grep -R -n -E '^(auto|allow-hotplug|iface|source|source-directory|up|down)' /etc/network/interfaces.d 2>/dev/null
systemctl is-active networking
```

Интерфейс часто называется `ens3`, но используйте фактическое имя. Убедитесь, что служба `networking` активна и блок `iface <INTERFACE> inet ...` есть в `/etc/network/interfaces` или подключённом файле `interfaces.d`. Если признаки не совпали, не применяйте эти команды и уточните конфигурацию образа. Сохраните именно найденный файл:

```bash
sudo cp -a /etc/network/interfaces /etc/network/interfaces.bak
```

Если блок находится в `interfaces.d`, скопируйте тот файл отдельно.

## Добавьте адрес без замены основного

В существующий блок `iface <INTERFACE> inet static` или `inet dhcp` добавьте строки, не меняя текущие `address`, `gateway`, `dns-nameservers` и `auto`:

```text
    up ip addr add <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE>
    down ip addr del <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE> || true
```

Команды `up/down` в существующем блоке сохраняют основной IP и gateway. Старый alias вроде `ens3:0` может встречаться, но не обязателен и не создаёт новую физическую карту. Не добавляйте второй блок с default gateway. При повторном `ifup` проверьте отсутствие дубликата адреса.

## Примените и проверьте

Из консоли можно добавить адрес без перезапуска интерфейса и проверить:

```bash
sudo ip addr add <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE>
ip -br a
ip route
```

Не запускайте `ifdown <INTERFACE>` через единственную SSH-сессию. Строки в конфигурации применятся при следующем штатном подъёме интерфейса или перезагрузке; планируйте это только с доступной консолью. Проверки ping, SSH извне и `curl`: [общая информация](../general.md).

## Откат

Удалите добавленные строки из файла или восстановите его backup, затем из консоли удалите только дополнительный адрес:

```bash
sudo ip addr del <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE>
```

Если SSH уже пропал, используйте VNC/веб-консоль, сверьте `ip route` и восстановите файл. [Диагностика](../troubleshooting.md).
