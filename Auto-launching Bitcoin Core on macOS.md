## Setting up macOS Ventura-native service for Bitcoin core 

*a.k.a., a* `systemctl * bitcoind` *equivalent*

#### Assumptions

1.    `bitcoind`, `bitcoin-cli`, etc. are installed in `/usr/local/bin` and functional.
2.    `bitcoind` is installed under `bitcoinuser`.
3.    `bitcoind` is using the macOS standard working directory for Bitcoin core: `/Users/bitcoinuser/Library/Application Support/Bitcoin`

Important: ensure that your `bitcoin.conf` does <u>not</u> contain the line `daemon=1` because  macOS' `launchctl` will manage its 'daemonizing.'  Otherwise,  `launchctl` will think `bitcoind` has disappeared and will throw an error. 

#### Configure

`sudo nano /Library/LaunchDaemons/org.bitcoin.bitcoind.plist`

Paste this into an empty file:

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>EnvironmentVariables</key>
	<dict>
		<key>PATH</key>
		<string>/usr/local/bin</string>
	</dict>
	<key>GroupName</key>
	<string>staff</string>
	<key>InitGroups</key>
	<true/>
	<key>KeepAlive</key>
	<dict>
		<key>Crashed</key>
		<true/>
		<key>SuccessfulExit</key>
		<false/>
	</dict>
	<key>Label</key>
	<string>org.bitcoin.bitcoind</string>
	<key>ProcessType</key>
	<string>Interactive</string>
	<key>Program</key>
	<string>/usr/local/bin/bitcoind</string>
	<key>StandardErrorPath</key>
	<string>/tmp/bitcoind.err</string>
	<key>StandardOutPath</key>
	<string>/tmp/bitcoind.out</string>
	<key>UserName</key>
	<string>bitcoinuser</string>
	<key>WorkingDirectory</key>
	<string>/Users/bitcoinuser/Library/Application Support/Bitcoin</string>
</dict>
</plist>
```

#### Launch

`sudo launchctl bootstrap system /Library/LaunchDaemons/org.bitcoin.bitcoind.plist`

#### Monitor

`tail -f /tmp/bitcoind.out`

`tail -f /tmp/bitcoind.err`

#### Check

Reboot the host, ssh into it without logging into its GUI, and ensure that

 `bitcoin-cli getblockchaininfo` returns something healthy.

To examine the process under the hood:

`sudo launchctl print system/org.bitcoin.bitcoind`

#### Know how to stop (and restart)

To stop the service:

`sudo launchctl bootout system /Library/LaunchDaemons/org.bitcoin.bitcoind.plist`

To restart, use the earlier command: 

`sudo launchctl bootstrap system /Library/LaunchDaemons/org.bitcoin.bitcoind.plist`

You can also use the regular command to stop:

`bitcoin-cli stop`

There is also another way:

`sudo launchctl unload system /Library/LaunchDaemons/org.bitcoin.bitcoind.plist`

and 

`sudo launchctl load system /Library/LaunchDaemons/org.bitcoin.bitcoind.plist`

None of the above changes the fact that the service will come up after reboot. 

To disable from launching at boot:

`sudo launchctl disable system/org.bitcoin.bitcoind`

... and reverse:

`sudo launchctl enable system/org.bitcoin.bitcoind`

Note: the above `disable` and `enable` options will take effect after the reboot, but they will not change the current running status of the service.

