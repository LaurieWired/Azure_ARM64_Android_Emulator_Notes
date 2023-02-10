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

## ADB
Install ADB to be able to connect to the device:
```
sudo apt-get install android-tools-adb
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

To kill and remove the emulator:
```
docker container rm -f androidemu
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