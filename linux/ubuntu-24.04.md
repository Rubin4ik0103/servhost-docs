# Ubuntu 24.04: дополнительный IPv4

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните резервную копию сетевого файла или профиля.
- Запишите текущие `ip a` и `ip route` (Windows: `Get-NetIPAddress` и `Get-NetRoute`).
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую SSH/RDP-сессию открытой и проверяйте новую сессию отдельно.

> Настройка сети может прервать удалённый доступ. Если нет работающей консоли, сначала обеспечьте её доступность.

## Проверьте интерфейс и действующую конфигурацию

```bash
ip -br a
ip route
systemctl is-active networking
sudo cat /etc/network/interfaces
sudo grep -R -n -E '^(auto|allow-hotplug|iface|source|source-directory)' /etc/network/interfaces.d 2>/dev/null
```

Найдите интерфейс с `<MAIN_IP>` и действующий default gateway. Проверьте, что на этом образе используется `networking`: найдите его блок `iface <INTERFACE> inet ...` в `/etc/network/interfaces` или подключённом файле. Имя может быть `ens3`, но берите фактическое. Если служба неактивна или блок не найден, не меняйте сеть по этой инструкции: уточните конфигурацию конкретного образа.

## Добавьте дополнительный IP

Сохраните backup файла, в котором находится блок интерфейса. Пример для основного файла:

```bash
sudo cp -a /etc/network/interfaces /etc/network/interfaces.bak
```

Если блок находится в `/etc/network/interfaces.d/<ACTUAL_FILE>`, сделайте backup именно этого файла. В существующий блок интерфейса добавьте:

```text
    up ip addr add <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE>
    down ip addr del <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE> || true
```

Сохраните основной IP, текущий gateway и другие строки без изменений. `<PREFIX>` возьмите из параметров услуги или уточните в поддержке. Подробный разбор файла и отката: [настройка networking](interfaces.md).

## Примените и проверьте

Через VNC/консоль добавьте адрес к работающему интерфейсу без его перезапуска:

```bash
sudo ip addr add <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE>
ip -br a
ip route
ping -c 3 <GATEWAY>
curl -4 https://api.ipify.org
```

Держите старую SSH-сессию открытой. С другого устройства проверьте `ping <ADDITIONAL_IP>` и `ssh <USER>@<ADDITIONAL_IP>`. ICMP может быть закрыт. Проверка `curl` показывает исходящий адрес по текущему маршруту: он может остаться основным. Не используйте `ifdown` по единственному SSH-подключению. Сохранённые строки применятся при следующем штатном поднятии интерфейса.

## Откат при потере SSH

Войдите через VNC/веб-консоль, восстановите backup конфигурационного файла и удалите только дополнительный адрес, если он уже добавлен:

```bash
sudo ip addr del <ADDITIONAL_IP>/<PREFIX> dev <INTERFACE>
ip -br a
ip route
```

Основной IP не удаляйте. Проверьте [диагностику](../troubleshooting.md).
