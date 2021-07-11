# Open source display for VESC
## Screenshots
![](https://github.com/mark99i/vesc-display-install/raw/main/indicators.PNG)

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
- VESC-UART part can provide data to third-party services in Json via API

## Architecture details
The project consists of two parts: `vesc-uart` and `vesc-display`

The first part,`vesc-uart`, holds the connection to the serial interface connected to the ESC. 
Starts an HTTP server for receiving requests, receives data from ESC when requests are received and returns it as a JSON structure.
`vesc-uart` also supports tcp transport, for connecting to tcp server VescTool

The second part, `vesc-display`, receives data via the API and displays it in the PyQt5-based user interface

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
If these options are set correctly in ESC, you can get these parameters from the ESC `get battery and motor from vesc` button in settings

## Running in test mode via TCP
If your ESC has a Bluetooth module, thanks to the support of the `vesc-uart` tcp transport, you can test the functionality without connecting to the ESC serial port.

To do this, start the TCP Server in the VescTool connected to the ESC.

Edit the `vesc-display/configs/config.json` file (will appear after the first launch) and change the `serial_port` option to `IP:PORT` in the format `iii.iii.iii.iii:pppppp`, where i is part of the tcp ip address of the VescTool server, and p is the port. 
For example: `192.168.001.020:65102`. The `serial_speed` does not matter in this mode.

If you did everything correctly, the information from the ESC will start updating (a little slower than when connecting the UART).

