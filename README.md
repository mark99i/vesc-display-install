# OpenSource Display for VESC Controllers
[Русская версия](https://github.com/mark99i/vesc-display-install/blob/main/README.ru.md) / **English version**

## Screenshots
<img src='https://github.com/mark99i/vesc-display-install/raw/main/main_window.png' width=580>
<table>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/indicators.PNG' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info.PNG' width=270></td></tr>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info_chart_1.jpg' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info_chart_2.jpg' width=270></td></tr>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/vesc_uart_status_info.PNG' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/settings.PNG' width=270></td></tr>
</table>

## Video demonstration
https://youtu.be/2bIfX92aeMQ

## Usability features
- Two interface modes: expert (as in the screenshots) and simplified
  - In expert mode, all current indicators for controllers, a graph of speed and power are available
  - The simplified interface displays the current speed, as well as information about the session and data from the last start when stopping. There are 3 parameters available for configuration
- Fast display of current speed and current data
- Saves the total distance (odometer)
- Calculation of the parameters of the session (average speed, maximum, watts, etc.)
- Saving session history, plotting speed and power during the session
- Has several indicators of consumption efficiency (eg. WhKm, WhKmH)
- Speedlogic mode with a maximum refresh rate for measuring acceleration 0-40, 0-50, 0-60, 0-70
- Fully supports (and designed for) Dual-ESC mode

## Architectural features
- Written entirely in Python
- Layout support for different screens (currently only 640x480 and 480x320 are drawn)
- Works on any device with Python 3.7 and higher
- The performance reaches 16-18 information updates per second on an old and slow device, such as the Raspberry Pi Zero W
- VESC-UART part can provide data to third-party services in Json via API

## Architecture details
The project consists of two parts: `vesc-uart` and `vesc-display`

The first part,`vesc-uart`, holds the connection to the serial interface connected to the ESC. 
Starts an HTTP server for receiving requests, receives data from ESC when requests are received and returns it as a JSON structure.
`vesc-uart` also supports tcp transport, for connecting to tcp server VescTool. ( https://github.com/mark99i/vesc-uart )

The second part, `vesc-display`, receives data via the API and displays it in the PyQt5-based user interface. ( https://github.com/mark99i/vesc-display )

If necessary, the parts can be located on different devices (and in different places), only ip connectivity is required

## Installing the pigpio service on RaspberryPi
_Note: You can skip this section if you will not use gpio pins to connect to the ESC._

Connect to the device via SSH and clone the pigpio project:
```
# uncomment if you are in the session as a user and not as root
# sudo -s 
cd /root
wget https://github.com/joan2937/pigpio/archive/master.zip
unzip master.zip
cd pigpio-master
make
make install
```

Create a systemd service and run it.
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
The installation of `pigpiod` is finished.

## Installation `vesc-uart` and `vesc-display`
Connect via SSH to the device, clone the project and install the necessary packages
```
sudo -s
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
Check that the serial interface (eg. /dev/ttyUSB0) connected to ESC is available as a result of last command.

Create a systemd service and run it.
As a result of executing the `service vesc-uart status` command, the status should be `active`, and the last line of the log is `starting server`
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
The installation of `vesc-uart` is finished.

To automatically start the GUI at system startup, you need to refer to the manual for the GUI environment you have installed. 
In my case, it was a Raspberry Pi Zero W
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
Enjoy!

## Configuring the connection to VESC
To create a configuration file, run `python3 vesc-display/main.py` and close it.
The configuration file will be located at the path `vesc-display/configs/config.json`, use the `nano` editor for editing.

There are three ways to connect to VESC: 
- the physical port serial/uart or usb-serial adapter (scheme `port://`) 
- tcp, with the intermediate device VescTool (scheme `tcp://`, see Run in test mode, using TCP) 
- using gpio pins using software serial library pigpio (scheme `pigpio://`)

Thus, the value of the `serial_vesc_path` option can have the form:
- `port://{path_to_device}?speed={speed}` (for example: `port:///dev/ttyUSB0?speed=115200`)
- `tcp://{ip_address}:{port}` (for example: `tcp://192.168.1.10:65102`)
- `pigpio://?tx={tx_pin}&rx={rx_pin}&speed={speed}` (for example: `pigpio://?tx=21&rx=20&speed=57600`)

If you are using pigpio, carefully study the pinout of your board to enter the correct gpio pin numbers for connection.
Installing pins that are occupied (for example, by the display or the spi module of the sd card) may cause the device to malfunction.

By default, the option value is `port:///dev/ttyUSB0?speed=115200`, that is, using an external usb-serial adapter and the speed is 115200.

## Setting up parameters
After the first launch, if you use DualESC, you will have information about only one ESC, and the speed and some other parameters will not be displayed correctly.
Make sure that the options `MotorCFG` -> `AdditionalInfo` are correctly set on the locally connected ESC: `MotorPoles`, `WheelDiameter`, `BatteryCells`, `BatteryCapacity`.

Then load these settings from ESC via the `get configuration from vesc` option in the settings.
The options of the same name in the settings and `esc_b_id` should be synchronized.

If you use an unsupported VESC FW, the `get configuration from vesc` option will not work, the parameters will need to be set manually.

## Run in test mode, using TCP
If your ESC has a Bluetooth module, thanks to the support of the `vesc-uart` tcp transport, you can test the functionality without connecting to the ESC serial port.

To do this, start the TCP Server in the VescTool connected to the ESC.

Edit the `vesc-display/configs/config.json` file (it will appear after the first launch) and change the `serial_vesc_path` option to `tcp://{ip}:{port}`
Example: `tcp://192.168.1.20:65102`.

If you did everything correctly, the information from the ESC will start updating (a little slower than when connecting the UART).

## Configuration description
The entire configuration is stored in the `vesc-display/configs/config.json` file that appears after the first launch of `vesc-display`.
All options, except for text `serial_vesc_api` and `serial_vesc_path`, can be changed using the built-in GUI editor. Config also contain other service options, changing which is not recommended without studying the source code


Options and their descriptions:

| Option | Description | 
| ------ | ------ |
| delay_update_ms | Pause in milliseconds between updates in the main thread of receiving information. It may be necessary if you want to reduce the load on the device or ESC by reducing the number of requests |
| delay_chart_update_ms | Pause in milliseconds between graph updates in the GUI |
| chart_points | The number of points on the graph. The more points there are, the longer the time interval is displayed on the graph |
| chart_pcurrent_insteadof_power | Display the values of the total phase current (PC) on the graph instead of the power (W) |
| nsec_calc_count | The number of states stored by the NSec calculation function is set based on UpdatesPerSeconds |
| use_gui_lite | Use a simplified interface |
| mtemp_insteadof_load | Display the motor temperature (MT) instead of loading the controller (L). Set 1 if you have a motor temperature sensor connected |
| speed_as_integer | Display the speed as an integer (without the fractional part) |
| write_session | Record your trip history |
| write_session_track | Record speed and power points during the trip for generate charts |
| session_track_average_sec | The number of seconds for which the average value of speed and power is taken. For short trips, slide 5-8, for long ones, increase |
| hw_controller_current_limit | The current parameter that is declared by your ESC manufacturer as the maximum. It only affects the calculation of the Load (L) point in the ESC statistics |
| hw_controller_voltage_offset_mv | Offset of voltage measurements (if the ESC measures inaccurately) |
| switch_a_b_esc | This option will reverse the display of ESC in the main window |
| esc_b_id | When using Dual-ESC mode, it will require setting the ID of the second ESC |
| write_logs | When this option is enabled, the application will record statistics when they are updated. They will be stored in the `logs` folder named `session_$date_$time.log`. They can be reproduced in the future via `vesc-display/main.py <path to log file>`. Please note that the log size is quite large, make sure that you don't run out of disk memory |
| motor_magnets | The number of magnets in your motor. Required to convert ERPM to RPM. Affects the calculation of the current speed and mileage distances |
| wheel_diameter | The outer diameter of the wheel in mm. Affects the calculation of the current speed and mileage distances |
| battery_cells | The number of consecutive blocks of your battery. Affects the determination of the current charge level and operation of the `battery_tracking` function |
| battery_mah | The declared or tested battery capacity in milliamps per hour. Affects on operation of the `battery_tracking` function |
| serial_vesc_api | The API address of the `vesc-uart` service. By default, it is assumed that it is located on the same device |
| serial_vesc_path | see Configuring the connection to VESC (above) |
| serial_speed | The speed of the serial port, when using the physical interface to connect to the ESC |
| service_enable_debug | Option to enable debugging of the `vesc-uart` service. By default, when enabled, the log will be output to syslog |
| service_rcv_timeout_ms | The option of the maximum waiting time for the response of the `vesc-uart` service from ESC. It can be increased when using TCP mode and a bad network |

Changes `serial_vesc_api`, `serial_vesc_path`, `service_enable_debug` and `service_rcv_timeout_ms` options require restarting the connection with ESC in the `service vesc-uart status` panel (UART button at the top).
The remaining options are applied immediately after clicking OK in the editor.

The standard values for the options can be found in the file `vesc-display/config.py`
