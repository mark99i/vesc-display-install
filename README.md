# OpenSource Display for VESC Controllers
## Screenshots
<img src='https://github.com/mark99i/vesc-display-install/raw/main/main_window.png' width=580>
<table>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/indicators.PNG' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/session_info.PNG' width=270></td></tr>
  <tr><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/vesc_uart_status_info.PNG' width=270></td><td><img src='https://github.com/mark99i/vesc-display-install/raw/main/settings.PNG' width=270></td></tr>
</table>

## Russian README.md: 
https://github.com/mark99i/vesc-display-install/blob/main/README.ru.md

## Video demonstration
https://youtu.be/2bIfX92aeMQ

## Usability features:
- Customizable interface
- Fast display of current speed and current data
- Displaying a graph by speed and power
- Saves the total distance (odometer)
- Has several indicators of consumption efficiency (eg. WhKm, WhKmH)
- Fully supports (and designed for) Dual-ESC mode

## Architectural features:
- Written entirely in Python
- Layout support for different screens (currently only 640x480 are drawn)
- Works on any device with Python 3.7 and higher
- The performance reaches 12 information updates per second on an old and slow device, such as the Raspberry Pi Zero W
- VESC-UART part can provide data to third-party services in Json via API

## Architecture details
The project consists of two parts: `vesc-uart` and `vesc-display`

The first part,`vesc-uart`, holds the connection to the serial interface connected to the ESC. 
Starts an HTTP server for receiving requests, receives data from ESC when requests are received and returns it as a JSON structure.
`vesc-uart` also supports tcp transport, for connecting to tcp server VescTool. ( https://github.com/mark99i/vesc-uart )

The second part, `vesc-display`, receives data via the API and displays it in the PyQt5-based user interface. ( https://github.com/mark99i/vesc-display )

If necessary, the parts can be located on different devices (and in different places), only ip connectivity is required

## Installation
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
As a result of executing the `systemd vesc-uart command`, the status should be `active`, and the last line of the log is `starting server`
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

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
service vesc-uart start
service vesc-uart status
systemctl enable vesc-uart
```
The installation of 'vesc-uart` is finished.

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

## Setting up parameters
After the first launch, if you use Dual-ESC, you will have information about only one ESC. To configure the display of data from the second one, first determine its ESC-ID.

The ID of the locally connected ESC can be viewed by clicking on the `UART` label at the top, item `local controller id`.
The ID of the connected CAN ESC can be viewed in VescTool. After the ESC ID is known, go to the settings (the icon in the corner on the top left) and set the `esc_b_id` item

After the correct configuration, you should see the parameter updates from both ESC.

For the correct display of the speed, the operation of the "battery tracking" function, it is necessary to set the correct parameters of the `battery_mah`, `battery_cells`, `motor_magnets`, and `wheel_diameter` options. 
If these options are set correctly in ESC, you can get these parameters from the ESC `get battery and motor from vesc` button in settings (supports only fw version 5.02).

## Running in test mode via TCP
If your ESC has a Bluetooth module, thanks to the support of the `vesc-uart` tcp transport, you can test the functionality without connecting to the ESC serial port.

To do this, start the TCP Server in the VescTool connected to the ESC.

Edit the `vesc-display/configs/config.json` file (will appear after the first launch) and change the `serial_port` option to `IP:PORT` in the format `iii.iii.iii.iii:pppppp`, where i is part of the tcp ip address of the VescTool server, and p is the port. 
For example: `192.168.001.020:65102`. The `serial_speed` does not matter in this mode.

If you did everything correctly, the information from the ESC will start updating (a little slower than when connecting the UART).

## Configuration description
The entire configuration is stored in the `vesc-display/configs/config.json` file that appears after the first launch of `vesc-display`.
All options, except for text `serial_vesc_api` and `serial_port`, can be changed using the built-in GUI editor. Config also contain other service options, changing which is not recommended without studying the source code


Options and their descriptions:

| Option | Description | 
| ------ | ------ |
| delay_update_ms | Pause in milliseconds between updates in the main thread of receiving information. It may be necessary if you want to reduce the load on the device or ESC by reducing the number of requests |
| delay_chart_update_ms | Pause in milliseconds between graph updates in the GUI |
| chart_power_points | The number of power graph points. The more points - the more time interval is displayed on the graph |
| chart_speed_points | The number of speed graph points. |
| wh_km_nsec_calc_interval | The update interval of the wh_km_nsec indicator, in seconds |
| hw_controller_current_limit | The current parameter that is declared by your ESC manufacturer as the maximum. It only affects the calculation of the Load (L) point in the ESC statistics |
| switch_a_b_esc | This option will reverse the display of ESC in the main window |
| esc_b_id | When using Dual-ESC mode, it will require setting the ID of the second ESC |
| write_logs | When this option is enabled, the application will record statistics when they are updated. They will be stored in the `logs` folder named `session_$date_$time.log`. They can be reproduced in the future via `vesc-display/main.py <path to log file>`. Please note that the log size is quite large, make sure that you don't run out of disk memory |
| motor_magnets | The number of magnets in your motor. Required to convert ERPM to RPM. Affects the calculation of the current speed and mileage distances |
| wheel_diameter | The outer diameter of the wheel in mm. Affects the calculation of the current speed and mileage distances |
| battery_cells | The number of consecutive blocks of your battery. Affects the determination of the current charge level and operation of the `battery_tracking` function |
| battery_mah | The declared or tested battery capacity in milliamps per hour. Affects and operation of the `battery_tracking` function |
| serial_vesc_api | The API address of the `vesc-uart` service. By default, it is assumed that it is located on the same device |
| serial_port | The address of the serial interface to connect to the ESC. It can have either the name of the physical port (eg. /dev/ttyUSB0) or the network address (see Running in test mode via TCP) |
| serial_speed | The speed of the serial port, when using the physical interface to connect to the ESC |
| service_enable_debug | Option to enable debugging of the `vesc-uart` service. By default, when enabled, the log will be output to syslog |
| service_rcv_timeout_ms | The option of the maximum waiting time for the response of the `vesc-uart` service from ESC. It can be increased when using TCP mode and a bad network |

Changes `serial_vesc_api`, `serial_port`, `serial_speed`, `service_enable_debug` and `service_rcv_timeout_ms` options require restarting the connection with ESC in the `service vesc-uart status` panel (UART button at the top).
The remaining options are applied immediately after clicking OK in the editor.

The standard values for the options can be found in the file `vesc-display/config.py`
