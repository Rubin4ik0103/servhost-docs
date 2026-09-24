# Диагностика дополнительного IPv4

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните резервную копию сетевого файла или профиля.
- Запишите текущие `ip a` и `ip route` (Windows: `Get-NetIPAddress` и `Get-NetRoute`).
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую SSH/RDP-сессию открытой и проверяйте новую сессию отдельно.

> Настройка сети может прервать удалённый доступ. Если нет работающей консоли, сначала обеспечьте её доступность.

Сначала проверьте четыре состояния в [общей информации](general.md). Команды выполняйте через рабочий основной IP или VNC. Заменяйте placeholders своими значениями.

## Базовая проверка Linux

```bash
ip a
ip -br a
ip route
ip rule
ip neigh
ss -lntp
systemctl status ssh
systemctl status sshd
ufw status
iptables -L -n
nft list ruleset
systemctl status networking
ping -c 3 <GATEWAY>
traceroute <ADDITIONAL_IP>
mtr <ADDITIONAL_IP>
curl -4 ifconfig.me
curl -4 https://api.ipify.org
```

Не все утилиты установлены. `traceroute` и `mtr` к дополнительному IP полезнее запускать **с другой сети**. `systemctl status ssh` обычно используется в Debian/Ubuntu, `sshd` часто в RHEL-подобных ОС. Не меняйте firewall вслепую.

| Симптом | Что проверить и сделать |
| --- | --- |
| IP есть в панели, но нет в `ip a` | Адрес назначен услуге, но не добавлен в ОС. Выберите фактический backend на странице ОС, добавьте адрес и проверьте `ip -br a`. |
| IP есть в `ip a`, но нет ping | Проверьте `<PREFIX>`, `ip route`, состояние интерфейса, ICMP/firewall и проверку из другой сети. ICMP может быть закрыт. |
| Ping работает, SSH нет | `ss -lntp`, `systemctl status ssh` или `sshd`, firewall и порт. Проверьте `sshd_config`: `ListenAddress` может ограничивать основной IP. |
| SSH доступен только по основному IP | Проверьте привязку sshd, firewall для нового адреса, маршрут ответов `ip route get <CLIENT_IP> from <ADDITIONAL_IP>` и внешнюю проверку порта. |
| После изменения сети пропал SSH | Через VNC восстановите backup файла `/etc/network/interfaces` или подключённого файла; проверьте имя интерфейса, основной IP и default route. |
| Неправильный gateway | Сверьте старый default route с `ip route`. Для дополнительного входящего IP обычно не нужен второй gateway. Верните прежний маршрут из backup. |
| Неправильная маска/prefix | Сверьте `<PREFIX>` с параметрами услуги или уточните у поддержки. Не копируйте префикс основного адреса наугад. |
| Duplicate IP | Проверьте, что адрес не назначен дважды в профилях или на другом интерфейсе; `ip neigh` и сообщения системы. Не используйте конфликтующий адрес до выяснения. |
| Interface DOWN | `ip -br link`, `systemctl status networking`; выясните причину до переподнятия, которое может прервать SSH. |
| Firewall | Сверьте `ufw status`, `iptables -L -n`, `nft list ruleset` и Windows Firewall. Разрешите только нужный сервис и адрес. |
| `sshd` не слушает | `ss -lntp`, статус службы, `ListenAddress` и журнал службы. Не считайте ping подтверждением SSH. |
| Reverse path filtering | При асимметричной маршрутизации проверьте `sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.<INTERFACE>.rp_filter`; сначала разберитесь в маршрутах, не отключайте проверку без причины. |
| Routing table | Сравните `ip route`, `ip rule` и `ip route get <CLIENT_IP> from <ADDITIONAL_IP>`; неверный ответный маршрут может ломать входящие соединения. |
| ARP / neighbor | Сверьте `ip neigh` и состояние интерфейса; если адрес и маршрут верны, соберите результаты и обратитесь в поддержку для проверки сетевой стороны. |
| После Docker/VPN/3x-ui/Amnezia/Xray изменились маршруты | Сравните `ip route`, `ip rule`, firewall до и после запуска приложения. Проверьте правила приложения и route ответа; не удаляйте основной маршрут вслепую. |
| После переустановки ОС IP исчез | Переустановка удаляет данные и локальную конфигурацию. Настройте дополнительный IP заново по странице новой ОС. |
| VNC работает, SSH нет | Проверьте адрес на интерфейсе, службу SSH, порт, firewall, `ListenAddress` и route ответа. Это не доказывает проблему с назначением адреса в панели. |

На Windows используйте `Get-NetIPAddress`, `Get-NetRoute`, `Get-NetAdapter`, `Test-NetConnection <ADDITIONAL_IP> -Port 22` (или нужный порт) и правила Windows Firewall. Если локальная конфигурация верна, а внешнее соединение не проходит, передайте в поддержку IP услуги, ОС, снимки адресов и маршрутов, время проверки, исходную сеть и результаты с обеих сторон. Доступность из всех сетей и стран не гарантируется; вывод о причине делается после диагностики. [FAQ](faq.md).


## FreeBSD

Проверьте `ifconfig -a`, `netstat -rn -f inet`, записи `ifconfig_<INTERFACE>_aliasN` в `/etc/rc.conf`, `sockstat -4 -l` и действующий firewall (`pfctl -sr`, если используется PF). Проверяйте входящий ping и SSH с другого устройства. При потере SSH войдите через VNC и восстановите `/etc/rc.conf.bak` по [инструкции FreeBSD](freebsd/freebsd-13.md). Не применяйте к FreeBSD команды `ip`, `systemctl` или `/etc/network/interfaces`.
