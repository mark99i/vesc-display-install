# Дисплей для контроллеров VESC
**Русская версия** / [English version](https://github.com/mark99i/vesc-display-install/blob/main/README.md)

## Скриншоты
<img src='https://github.com/mark99i/vesc-display-install/raw/main/main_window.png' width=580>
<table>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/indicators.PNG' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info.PNG' width=270></td></tr>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info_chart_1.jpg' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info_chart_2.jpg' width=270></td></tr>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/vesc_uart_status_info.PNG' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/settings.PNG' width=270></td></tr>
</table>

## Видео
https://youtu.be/2bIfX92aeMQ

## Основные функции
- Два режима интерфейса: экспертный (как на скриншотах) и упрощенный
  - В экспертном режиме доступны все текущие показатели по контроллерам, график скорости и мощности
  - В упрощенном интерфейсе показано отображение текущей скорости, а также сведений о сессии и данных с последнего старта при остановке. Для настройки доступны 3 параметра
- Быстрое отображение текущих данных тока и скорости
- Сохранение суммарного пробера
- Подсчет параметров сессии (средняя скорость, максимальная, затраченные ватты, и т.д.)
- Сохранение истории сессий, построение графиков скорости и мощности в течение сессии
- Несколько индикаторов энергоэффективности (таких, как Ватт/Км, моментальный Ватт/Км)
- Режим speedlogic с максимальной частотой обновления для измерения разгона 0-40, 0-50, 0-60, 0-70
- Полностью поддерживает (и разрабатывался под) двухконтроллерные конфигурации VESC по CAN

## Архитектурные особенности
- Полностью написан на Python
- Поддержка скинов интерфейса для разных разрешений экрана (пока что нарисованы только 640x480 и 480х320)
- Работает на любом устройстве с Python 3.7 и выше
- Производительность достигает 16-18 обновлений в секунду на старых и слабых устройствах, таких как Raspberry Pi Zero W
- VESC-UART может предоставлять данные с контроллера сторонним сервисам в формате JSON через API

## Детали архитекруты
Проект состоит из двух частей: `vesc-uart` и `vesc-display`

Первая часть,`vesc-uart`, соединяется с serial интерфейсом, который подключен к ESC.
Запускает HTTP сервер, выполняет запросы на ESC и возвращает данные в виде JSON структуры.
`vesc-uart` поддерживает tcp транспорт, для подключения к tcp server vesctool. ( https://github.com/mark99i/vesc-uart ).

Вторая часть, `vesc-display`, получает данные по API и отображает их в интерфейсе, основанным на PyQt5 ( https://github.com/mark99i/vesc-display ).

При необходимости, части могут находиться на разных устройствах (и в разных местах), потребуется только IP-связность.

## Установка службы pigpio на RaspberryPi
_Примечание: вы можете пропустить этот раздел, если не будете использовать gpio пины для подключения к ESC._

Подключитесь к устройству по SSH и склонируйте проект pigpio:
```
# раскомментируйте, если вы находитесь в сессии от имени пользователя, а не root
# sudo -s 
cd /root
wget https://github.com/joan2937/pigpio/archive/master.zip
unzip master.zip
cd pigpio-master
make
make install
```

Создайте systemd сервис и запустите.
```
cat > /etc/systemd/system/pigpiod.service << EOF
[Unit]
Description=PIGPIO Service
After=network.target

[Service]
Type=simple
ExecStart=/root/pigpio-master/pigpiod -g -l
StandardOutput=syslog
StandardError=syslog

KillSignal=SIGINT
KillMode=process
TimeoutStopSec=10
TimeoutStartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
service pigpiod start
service pigpiod status
systemctl enable pigpiod
```
Установка службы ```pigpiod``` завершена.

## Установка `vesc-uart` и `vesc-display`
Подключитесь к устройству по SSH, склонируйте проекты и установите необходимые пакеты:
```
# раскомментируйте, если вы находитесь в сессии от имени пользователя, а не root
# sudo -s 
cd /home/$SUDO_USER/Desktop/
git clone https://github.com/mark99i/vesc-uart
git clone https://github.com/mark99i/vesc-display
chown -R $SUDO_USER:$SUDO_USER /home/$SUDO_USER/Desktop/
apt install python3 python3-pip python-pyqt5 python3-pyqt5.qtchart -y
python3 -m pip install PySerial
python3 -m pip install ujson
python3 -m pip install screeninfo

python3 -c "import serial.tools.list_ports; print ([port.name for port in serial.tools.list_ports.comports()])"
```
В результате последней команды должен быть serial интерфейс, подключенный к ESC.

Создайте systemd сервис и запустите.
В результате выполнения команды `service vesc-uart status`, статус должен быть `active` и последняя строка `starting server`.
```
cat > /etc/systemd/system/vesc-uart.service << EOF
[Unit]
Description=VESC-UART Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 -u /home/$SUDO_USER/Desktop/vesc-uart/main.py
StandardOutput=syslog
StandardError=syslog

KillSignal=SIGINT
KillMode=process
TimeoutStopSec=10
TimeoutStartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
service vesc-uart start
service vesc-uart status
systemctl enable vesc-uart
```
Установка `vesc-uart` завершена.

Для автоматического запуска GUI части, скорее всего потребуется найти инструкцию по установленной у вас оболочке.
На Raspberry Pi Zero W вы можете сделать так:
```
mkdir /home/$SUDO_USER/.config/autostart
cat > /home/$SUDO_USER/.config/autostart/vesc-display.desktop << EOF
[Desktop Entry]
Encoding=UTF-8
Name=vdisplay
Comment=vesc-display
Exec=/usr/bin/python3 /home/$SUDO_USER/Desktop/vesc-display/main.py
EOF
chown -R $SUDO_USER:$SUDO_USER /home/$SUDO_USER/.config/autostart
```
Готово!

### Настройка подключения к VESC
Чтобы создать файл конфигурации запустите `python3 vesc-display/main.py` и закройте.
Файл конфигурации будет находиться по пути `vesc-display/configs/config.json`, для редактирования используйте редактор `nano`.

Подключение к VESC может осуществляться тремя способами: 
- через физический порт serial/uart или usb-serial переходник (схема `port://`)
- через tcp, с промежуточным устройством VescTool (схема `tcp://`, см Запуск в тестовом режиме через TCP)
- через gpio пины, используя software serial библиотеки pigpio (схема `pigpio://`)

Таким образом значение опции `serial_vesc_path` может иметь вид:
- `port://{path_to_device}?speed={speed}` (например: `port:///dev/ttyUSB0?speed=115200`)
- `tcp://{ip_address}:{port}` (например: `tcp://192.168.1.10:65102`)
- `pigpio://?tx={tx_pin}&rx={rx_pin}&speed={speed}` (например: `pigpio://?tx=21&rx=20&speed=57600`)

В случае использования pigpio внимательно изучите распиновку вашей платы, чтобы ввести правильные номера пинов gpio для подключения.
Установка занятых (например дисплеем или spi модулем sd карты) пинов может привести к неработоспособности устройства.

По умолчанию значением опции является `port:///dev/ttyUSB0?speed=115200`, то есть использование внешнего usb-serial переходника и скорость 115200.

## Первоначальная настройка
После первого запуска, если вы используете DualESC, у вас будет информация только об одном ESC, также не будет корректно отображаться скорость и некоторые другие параметры.
Убедитесь, что на локально подключенном ESC правильно заданы опции `MotorCFG` -> `AdditionalInfo`: `MotorPoles`, `WheelDiameter`, `BatteryCells`, `BatteryCapacity`.

Затем загрузите эти настройки из ESC через опцию `get configuration from vesc` в настройках.
Одноименные опции в настройках и `esc_b_id` должны синхронизироваться.

Если у вас используется неподдерживаемая VESC FW, то опция `get configuration from vesc` работать не будет, параметры нужно будет установить вручную.

## Запуск в тестовом режиме через TCP
Если ваш контроллер имеет Bluetooth модуль, благодаря поддержке tcp транспорта `vesc-uart`, вы можете проверить функциональность без подключения к serial порту ESC.

Для этого, в подключенном к ESC VescTool запустите TCP Server.

Отредактируйте файл `vesc-display/configs/config.json` (появится после первого запуска) и смените опцию `serial_vesc_path` на `tcp://{ip}:{port}`
Пример: `tcp://192.168.1.20:65102`.

Если вы все сделали правильно, информация с контроллера начнет обновляться (немного медленнее, чем при подключении через serial).

## Описание конфигурации/настроек
Вся конфигурация хранится в файле `vesc-display/configs/config.json`, который создается при первом запуске `vesc-display`.
Все опции, кроме текстовых `serial_vesc_api` и `serial_vesc_path`, могут быть изменены через встроенный в интерфейсе редактор.
Файл конфигурации также содержит некоторые другие опции, изменение которых не рекомендуется без изучения исходного кода.

Опции и их описания:

| Опция | Описание | 
| ------ | ------ |
| delay_update_ms | Пауза в миллисекундах между обновлениями в основном потоке получения информации. Можете увеличить, если вы хотите снизить нагрузку на устройство или ESC за счет уменьшения количества запросов |
| delay_chart_update_ms | Пауза в миллисекундах между обновлениями графика в GUI |
| chart_points | Количество точек графика. Чем больше точек - тем больший интервал времени отображается на графике |
| chart_pcurrent_insteadof_power | Отображать на графике вместо мощности (W) значения суммарного фазного тока (PC) |
| nsec_calc_count | Количество состояний, хранимых функцией подсчета значений NSec, выставляется исходя из UpdatesPerSeconds |
| use_gui_lite | Использовать упрощенный интерфейс |
| mtemp_insteadof_load | Отображать температуру мотора (MT) вместо загрузки контроллера (L). Установите 1, если у вас подключен датчик температуры мотора |
| speed_as_integer | Отображать скорость целым числом (без дробной части) |
| write_session | Записывать историю поездок |
| write_session_track | Записывать точки скорости и мощности во время поездки для построения графиков |
| session_track_average_sec | Количество секунд, за которые берется среднее значение скорости и мощности. При коротких поездках искользуйте 5-8, для длинных увеличивайте |
| hw_controller_current_limit | Фазный ток, объявленный вашим производителем ESC как максимальный. Это влияет только на расчет нагрузки (L) в статистике ESC |
| hw_controller_voltage_offset_mv | Смещение измерений напряжения (если ESC измеряет неточно) |
| switch_a_b_esc | Эта опция поменяет местами отображение ESC в главном окне |
| esc_b_id | При использовании режима двухконтроллерной конфигурации потребуется установить ID второго ESC |
| write_logs | Если эта опция включена, приложение будет записывать статистику при ее обновлении. Она будут храниться в папке `журналы` с именем `session_$date_$time.log`. Логи могут быть воспроизведены в будущем с помощью `vesc-display/main.py <путь к файлу журнала>`. Пожалуйста, обратите внимание, что размер журнала довольно большой, убедитесь, что у вас не закончится место на диске |
| motor_magnets | Количество магнитов в моторе. Требуется для преобразования ERPM в RPM. Влияет на расчет текущей скорости и расстояния пробега |
| wheel_diameter | Наружный диаметр колеса в мм. Влияет на расчет текущей скорости и расстояния пробега |
| battery_cells | Количество последовательных блоков вашей батареи. Влияет на определение текущего уровня заряда и работу функции отслеживания заряда |
| battery_mah | Заявленная или протестированная емкость аккумулятора в миллиамперах в час. Влияет на работу функции отслеживания заряда |
| serial_vesc_api | Адрес API сервиса `vesc-uart`. По умолчанию предполагается, что он расположен на том же устройстве |
| serial_vesc_path | см. Настройка подключения к VESC (выше) |
| service_enable_debug | Опция для включения отладки службы `vesc-uart`. Если этот параметр включен, журнал запросов будет выводиться в syslog (stdout) |
| service_rcv_timeout_ms | Опция максимального времени ожидания ответа `vesc-uart` от ESC. Он может быть увеличен при использовании режима TCP и низкой скорости сети |

Изменения параметров `serial_vesc_api`, `serial_vesc_path`, `service_enable_debug` и `service_rcv_timeout_ms` требуют перезапуска соединения с ESC на панели "status vesc-uart service" (кнопка `UART` наверху).
Остальные параметры применяются сразу после нажатия кнопки `ОК` в редакторе.

Стандартные значения параметров можно найти в файле `vesc-display/config.py`