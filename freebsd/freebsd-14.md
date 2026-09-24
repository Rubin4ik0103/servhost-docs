# FreeBSD 14: дополнительный IPv4

## Перед началом

- Убедитесь, что VNC/веб-консоль работает, и оставьте текущую SSH-сессию открытой.
- Сохраните backup `/etc/rc.conf`; запишите вывод `ifconfig -a` и `netstat -rn -f inet`.
- Не удаляйте основной IP и меняйте настройки по одному действию.
- Настройка внутри ОС выполняется клиентом самостоятельно.

> Не перезапускайте `netif` и `routing` из единственной SSH-сессии: это может оборвать доступ.

## Узнайте интерфейс и действующий маршрут

```sh
ifconfig -a
netstat -rn -f inet
sysrc -a | grep -E '^(ifconfig_|defaultrouter)'
```

Найдите интерфейс, на котором уже находится `<MAIN_IP>`. Во FreeBSD имя может быть `vtnet0` или другим. Действующий gateway оставьте без изменений. Если сеть образа конфигурирует другое средство, найдите его конфигурацию до правки.

## Сохраните файл и добавьте адрес

```sh
sudo cp -p /etc/rc.conf /etc/rc.conf.bak
```

В `/etc/rc.conf` добавьте новую строку для **того же** интерфейса. Выберите свободный номер alias после проверки имеющихся `ifconfig_<INTERFACE>_aliasN`. Маску `<NETMASK>` получите из выданного `<PREFIX>`; не подставляйте `255.255.255.255` без подтверждения схемы адреса.

```text
ifconfig_<INTERFACE>_alias0="inet <ADDITIONAL_IP> netmask <NETMASK>"
```

Если `alias0` уже занят, используйте следующий свободный индекс. Не меняйте основную строку `ifconfig_<INTERFACE>` и `defaultrouter`. ZFS как вариант диска не меняет порядок настройки сети.

## Примените и проверьте

Из VNC/веб-консоли добавьте IP без перезапуска интерфейса:

```sh
sudo ifconfig <INTERFACE> inet <ADDITIONAL_IP> netmask <NETMASK> alias
ifconfig <INTERFACE>
netstat -rn -f inet
ping -c 3 <GATEWAY>
fetch -qo - https://api.ipify.org
```

С другой машины проверьте `ping <ADDITIONAL_IP>` и `ssh <USER>@<ADDITIONAL_IP>`. ICMP может быть закрыт. `fetch` показывает обычный исходящий IP, который может остаться основным. Для проверки сохранения настройки после перезагрузки используйте консоль и резервную копию.

## Откат и потеря SSH

Через VNC удалите добавленную строку из `/etc/rc.conf` или восстановите backup. Удалите **только** дополнительный адрес из работающей системы:

```sh
sudo ifconfig <INTERFACE> inet <ADDITIONAL_IP> -alias
ifconfig <INTERFACE>
netstat -rn -f inet
```

Если SSH пропал, проверьте адрес, маршрут, `sockstat -4 -l` и firewall через VNC. Подробнее: [диагностика](../troubleshooting.md).
