# AlmaLinux 8/9: дополнительный IPv4

## Перед началом

- Убедитесь, что доступна VNC или веб-консоль, и проверьте вход в неё.
- Сохраните резервную копию активного сетевого профиля.
- Запишите текущие `ip a` и `ip route`.
- Не удаляйте основной IP. Выполняйте действия по одному.
- Настройка внутри ОС выполняется клиентом самостоятельно. Держите текущую SSH-сессию открытой и проверяйте новую отдельно.

> Изменение сети может прервать SSH. Работайте через VNC/консоль, если не можете восстановить подключение другим способом.

## Определите интерфейс и службу

```bash
ip -br a
ip route
systemctl is-active NetworkManager
systemctl is-active networking
nmcli device status
nmcli connection show --active
```

Найдите `<INTERFACE>` с `<MAIN_IP>`. Если `NetworkManager` активен и `nmcli device status` показывает этот интерфейс как **connected** с активным профилем, выполните следующие шаги. Наличие программы `nmcli` само по себе этого не доказывает.

Если интерфейс действительно обслуживается `networking` и его блок `iface <INTERFACE> inet ...` найден в `/etc/network/interfaces` или подключённом файле, используйте [инструкцию networking](interfaces.md). Если ни один вариант не подтверждён, не меняйте сеть по этим примерам и уточните конфигурацию образа.

## Найдите и сохраните активный профиль NetworkManager

```bash
nmcli -f GENERAL.CONNECTION,GENERAL.DEVICE device show <INTERFACE>
nmcli -f connection.id,connection.uuid,ipv4.method,ipv4.addresses,ipv4.gateway connection show "<PROFILE>"
nmcli connection show "<PROFILE>" > profile-before.txt
```

Подставьте точное имя `<PROFILE>`, связанного с `<INTERFACE>`. По UUID из вывода найдите файл профиля в `/etc/NetworkManager/system-connections/` или, для некоторых образов AlmaLinux 8, в `/etc/sysconfig/network-scripts/`. Не правьте другой профиль и не угадывайте имя файла.

```bash
sudo grep -R -l -F "<PROFILE_UUID>" /etc/NetworkManager/system-connections /etc/sysconfig/network-scripts 2>/dev/null
sudo cp -a <ACTUAL_FILE> <ACTUAL_FILE>.bak
```

Если файл активного профиля не найден или он создаётся автоматически, остановитесь и выясните источник конфигурации до изменения. Сохраните `profile-before.txt` в безопасном месте: вывод профиля может содержать сетевые параметры.

## Добавьте адрес без замены основного

Проверьте, что `<ADDITIONAL_IP>` ещё не присутствует в профиле и `ip -br a`. Возьмите `<PREFIX>` из параметров услуги, не копируйте его с основного IP наугад.

```bash
sudo nmcli connection modify "<PROFILE>" +ipv4.addresses "<ADDITIONAL_IP>/<PREFIX>"
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway connection show "<PROFILE>"
```

Знак `+` добавляет адрес к существующим; не используйте `ipv4.addresses` без `+`, так как это может заменить основной адрес. Не меняйте `ipv4.method`, DNS и gateway.

## Примените и проверьте

```bash
sudo nmcli device reapply <INTERFACE>
ip -br a
ip route
curl -4 https://api.ipify.org
```

Если `reapply` сообщает, что изменение нельзя применить, не выполняйте `connection down/up` через единственную SSH-сессию. Переподключайте профиль только через VNC/консоль после сохранения backup. С другого устройства проверьте `ping <ADDITIONAL_IP>` и `ssh <USER>@<ADDITIONAL_IP>`. ICMP может быть закрыт. `curl` без выбора source IP может показать основной адрес.

## Откат и потеря SSH

Через VNC/веб-консоль удалите только добавленный адрес из профиля и перепримените его:

```bash
sudo nmcli connection modify "<PROFILE>" -ipv4.addresses "<ADDITIONAL_IP>/<PREFIX>"
sudo nmcli device reapply <INTERFACE>
ip -br a
ip route
```

Если профиль повреждён, восстановите копию `<ACTUAL_FILE>.bak` через консоль, выполните `sudo nmcli connection load <ACTUAL_FILE>` и перепримените интерфейс. Не удаляйте `<MAIN_IP>`. Подробнее: [диагностика](../troubleshooting.md).
