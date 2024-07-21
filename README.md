# Running an ARM Emulator in Azure

## Docker Install
Follow the Docker docs to install on Ubuntu: https://docs.docker.com/engine/install/ubuntu/

To use Docker without having to type "sudo" with each command:
```
sudo groupadd docker
sudo usermod -aG docker $USER
```

Then reboot to take effect.

Install the required kernel modules for running the container:
```
sudo apt install linux-modules-extra-`uname -r`
sudo modprobe binder_linux devices="binder,hwbinder,vndbinder"
sudo modprobe ashmem_linux
```

## Automatically load the kernel modules after next reboot
- Create a service to automatically load the modules after restarting.
- Create the script that will be loaded by the `modprobe.service`.
  (Make sure `modprobe` path `/usr/sbin/modprobe` is correct for your VM by running `whereis modprobe`)
- Set the script as executable, reload the systemd daemon, enable the service to run at boot and finally start it:
```
sudo bash -c 'echo -e "[Unit]\nDescription=Load Kernel Modules at Boot\nDefaultDependencies=no\nBefore=local-fs.target\n\n[Service]\nType=oneshot\nExecStart=/usr/local/bin/load-modules.sh\nRemainAfterExit=true\n\n[Install]\nWantedBy=multi-user.target" > /etc/systemd/system/modprobe.service'
sudo bash -c 'echo -e "#!/bin/bash\n/usr/sbin/modprobe binder_linux devices=\"binder,hwbinder,vndbinder\"\n/usr/sbin/modprobe ashmem_linux" > /usr/local/bin/load-modules.sh'
sudo chmod +x /usr/local/bin/load-modules.sh
sudo systemctl daemon-reload
sudo systemctl enable modprobe.service
sudo systemctl start modprobe.service
```

After the next reboot you can check if the modules were properly loaded by running:
```
lsmod | grep "_linux"
```
And the response should be something like
```
ashmem_linux           24576  59
binder_linux          225280  88
```

## ADB
Install ADB to be able to connect to the device:
```
sudo apt-get update -y && sudo apt-get install android-tools-adb -y
```
### Example Helpful ADB Commands
If you need to run as root on the device. You may need to reconnect to the device after this:
```
adb root
```
List the emulators that are running:
```
adb devices
```
If the device isn’t appearing (assuming the device is running on localhost port 5555):
```
adb connect 127.0.0.1:5555
```
## Running the Emulator
Redroid Docker for ARM Android: https://hub.docker.com/r/redroid/redroid
```
docker run -itd --privileged \
 --name androidemu \
 -v ~/data:/data \
 -p 5555:5555 \
 redroid/redroid:11.0.0-latest
```
If you did not make the modprobe commands persistent and the device is failing to start properly, you may need to run those previous commands one more time.

To start the emulator after restarting the VM:
```
docker start androidemu
```

To kill and remove the emulator:
```
docker container rm -f androidemu
```
It's also a good idea to auto start the docker container after the next reboot:
```
echo -e "[Unit]\nDescription=Start Android Emulator Docker Container\nAfter=docker.service\nRequires=docker.service modprobe.service\n\n[Service]\nExecStart=/usr/bin/docker start androidemu\nRestart=always\n\n[Install]\nWantedBy=multi-user.target" | sudo tee /etc/systemd/system/androidemu.service > /dev/null && sudo systemctl daemon-reload && sudo systemctl enable androidemu.service && sudo systemctl start androidemu.service
```

## Local Screen Connect
To connect to the device screen, use SSH tunnelling to forward the ADB port on the Azure machine:
```
ssh -L 5555:127.0.0.1:5555 azureuser@AZURE_MACHINE_IP_ADDRESS
```
Now you can run ADB connect on your host machine and communicate with the remote device:
```
adb connect 127.0.0.1:5555
adb devices
```
Install scrcpy on your host: https://github.com/Genymobile/scrcpy. Then run scrcpy to connect the device connected to ADB:
```
scrcpy.exe -s emulator-5554 # replace this with the name of the emulator
```
