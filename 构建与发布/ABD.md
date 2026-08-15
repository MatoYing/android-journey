#### 如何连接车机
连接车机第一种方式是使用 USB，插上就可以，下面那里就会显示出你的车机，点击运行，车机就会显示你的程序。并且你可以利用 Scrcpy 将车机的画面投射到你的电脑上，只需在 Terminal 中输入 `scrcpy`。![[../assets/Pasted image 20260815180907.png]]还可以通过网络的方式进行连接，连接上车机的 WIfi（在“连接”里的“车辆热点”），拿到这个 Router，然后 `adb connect 10.16.169.82` 即可。
![[../assets/Pasted image 20260815180934.png]]注意，这里可能会出现一些问题，如果让车机和我的电脑，共同连上我的手机热点，按上面同样操作，连接 Router `172.20.10.1` 发现无法连接。
上面呢个 CADILLAC_9836 应该是车机自身的 WiFi，所以呢个 Router IP 本身就是车机自己，直接 connect 就可以成功。而连上手机热点的呢个 Router IP，是手机的，connect 肯定失败，必须连车机的，也就是车机在这个网络里分配到的 IP。如何获得呢？通过下面的命令，找到 wlan0（Wireless LAN 无线局域网），这个代表了这台设备用来连 Wi-Fi 的网卡接口，`adb connect 172.20.10.9` 可以成功。
```shell
adb shell
# 在shell里切换到root用户；adb root是让adb以root启动
patacvcupro:/ $ su
# 查看网络相关配置
patacvcupro:/ $ ifconfig
# .....
wlan0     Link encap:Ethernet  HWaddr 66:b4:1b:28:6f:22  Driver cnss_pci
          inet addr:172.20.10.9  Bcast:172.20.10.15  Mask:255.255.255.240 
          inet6 addr: fe80::64b4:1bff:fe28:6f22/64 Scope: Link
          inet6 addr: 240e:46d:8500:c407:64b4:1bff:fe28:6f22/64 Scope: Global
          inet6 addr: 240e:46d:8500:c407:7083:5c9a:7c4b:9e6b/64 Scope: Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:22656 errors:0 dropped:0 overruns:0 frame:0 
          TX packets:26319 errors:0 dropped:0 overruns:0 carrier:0 
          collisions:0 txqueuelen:3000 
          RX bytes:12462013 TX bytes:2784198 
# .....
```
另外，想执行上面的东西，在车机和你的电脑连上电脑后，你首先得借一条 USB，连上电脑，进入`adb shell`，然后用上面的命令去查看 wlan0，否则进入不了 `adb shell`。