# PushProxy

PushProxy is a **man-in-the-middle proxy** for **iOS and OS X Push Notifications**. It decodes the push protocol and outputs messages in a readable form. It also provides APIs for handling messages and sending push notifications directly to devices without sending them via Apple's infrastructure.

For a reference on the push protocol, see [apple-push-protocol-ios5-lion.md](doc/apple-push-protocol-ios5-lion.md). iOS4 and earlier used another version of the protocol, described in [apple-push-protocol-ios4.md](doc/apple-push-protocol-ios4.md). This proxy only supports the iOS5 protocol.

I tested only using jailbroken iOS devices, but it may be possible to use a push certificate from a jailbroken device and use it to connect a non-jailbroken device to Apple's servers. At least some apps using push notifications will be confused if you do this, but I think this was a way for hacktivated iPhones to get a push certificate.

## 'Screenshot'

    2013-02-17 21:54:15+0100 [#0] New connection from 192.168.0.120:61321
    2013-02-17 21:54:15+0100 [#0] SSL handshake done: Device: B481816D-4650-49EA-977C-9FCBDEB30CB1
    2013-02-17 21:54:15+0100 [#0] Connecting to push server: 17.172.232.59:5223
    2013-02-17 21:54:15+0100 Starting factory <icl0ud.push.intercept.InterceptClientFactory instance at 0x147d5a8>
    2013-02-17 21:54:15+0100 [#0] -> APSConnect presenceFlags: 00000002 state: 01
                                       push token: 4b5543c35b48429bbe770a8af11457f39374bd4e43e94adaa86bd979c50bdb05
    2013-02-17 21:54:16+0100 [#0] <- APSConnectResponse 00 messageSize: 4096 unknown5: 0002
    2013-02-17 21:54:16+0100 [#0] -> APSTopics for token <none>
                                       enabled topics: 
                                         com.apple.itunesstored
                                         com.apple.madrid
                                         com.apple.mobileme.fmip
                                         com.apple.gamed
                                         com.apple.ess
                                         com.me.keyvalueservice
                                         4357ca2451b7b787caea8a603e51a1f45feaeda4
                                         com.apple.mobileme.fmf1
                                         com.apple.mediastream.subscription.push
                                       disabled topics:
                                         com.apple.store.Jolly
    2013-02-17 21:54:16+0100 [#0] -> APSKeepAlive 10min carrier: 31038 iPhone4,1/6.1.1/10B145
    2013-02-17 21:54:16+0100 [#0] <- APSKeepAliveResponse
    2013-02-17 22:10:04+0100 [#0] <- APSNotification 4357ca2451b7b787caea8a603e51a1f45feaeda4
                                       timestamp: 2013-02-17 22:10:16.642561 expiry: 1970-01-01 00:59:59
                                       messageId: 00000000                   storageFlags: 00
                                       unknown9:  None                       payload decoded (json)
                                       {u'aps': {u'alert': u'Chrome\u2014meeee/pushproxy \xb7 GitHub\nhttps://github.com/meeee/pushproxy',
                                                 u'badge': 3,
                                                 u'sound': u'elysium.caf'},
                                        u'urlhint': u'1234567'}
    2013-02-17 22:10:06+0100 [#0] -> APSNotificationResponse message: 00000000 status: 00

## Setup

The proxy is written in Python, I recommend setting up a virtualenv and installing the requirements:

    pip install -r src/requirements.txt

Setting up PushProxy requires redirecting push connections to your server and setting up X.509 certificates for SSL authentication.

The push protocol uses SSL for client and server authentication. You need to extract the device certificate and give it to the proxy so it can authenticate itself as device against Apple. You also need to make the device trust your proxy since you are impersonating Apple's servers.

### Server certificate

Note: This certificate creation only works for WiFi connections, see below if you want the proxy to work via 3G.

You need to create a SSL server certificate and install it on your device. It's common name should be:

    courier.push.apple.com

Then place the certificate in PEM encoding at the following path:

    certs/courier.push.apple.com/server.pem

You can install the certificate on iOS devices via Safari or iPhone Configuration Utility.

On OS X you can use keychain access to install the certificate, make sure to install it in the System keychain, not your login keychain. Mark it as trusted, Keychain Access should then display it as 'marked as trusted for all users'.

### Extract iOS Certificates

First, download the [nimble](http://xs1.iphwn.org/releases/PushFix.zip) tool, extract `PushFix.zip` and place `nimble` into `setup/ios`. The following script will copy the tool to your iOS device, run it and copy the extracted certificates back to your computer. It assumes you have **SSH** running on your device. I recommend setting up key-based authentication, otherwise you will be typing your password a few times.

Make sure you are in the pushproxy root directory, otherwise the script will fail.

    cd pushproxy
    setup/ios/extract-and-convert-certs.sh root@<device hostname>

You can find the extracted certificates in `certs/device`. Both public and private key are in one PEM-file.

### Extract OS X Certificates

Note: If you want to connect at least one device via a patched push daemon, you need to patch the push daemon on OS X first.

OS X stores the certificates in a keychain in `/Library/Keychains`, either in `applepushserviced.keychain` or in `apsd.keychain`.

This step extracts the push private key and certificate from the keychain. It stores them in `certs/device/<UUID>.pem`

    setup/osx/extract_certificate.py -f

You can remove the -f parameter to get key and certificate on stdout instead of writing them to a file.

### DNS redirect(WiFi only)

The simplest way for redirecting a jailbroken iOS device or a Mac is modifying the `/etc/hosts` file. The following command will generate a hosts file for you. It may generate a few entries too much, but that shouldn't hurt.

    python setup/generate-hosts-file.py <server ip> > hosts

You obviously need to copy the generated hosts file to your device.

Make sure your device doesn't have network access via a phone network. In this case iOS ignores the `/etc/hosts` file and uses your carrier's DNS instead. **Disabling mobile data** should do the trick.

### Redirect via push daemon patch(WiFi+3G)

This method modifies the push daemons(apsd on iOS, applepushserviced on OS X) and replaces the string `push.apple.com` with a 14-character domain name of your choice.

#### Preparation: DNS Setup

You need two DNS entries, one wildcard A-record and a TXT record.

First, you have to choose a domain name. It must be exactly **14 characters** long like `push.apple.com`, so e.g. `ps.example.com` would work. (You could probably also use a shorter name and fill the remaining space with zero-bytes, but I haven't tried that).

The first DNS entry should be a **wildcard A-record** pointing to your servers IP, like `*.ps.example.com`.

An additional TXT record is used probably for determining the number of push domains the devices choose from. I set it to the same value `50` push.apple.com uses, but another one might also work. The content of this **TXT record** should look like ``"count=50"``.

You can verify your DNS setup using `dig`, it should show a similar answer for your server like it does for Apple's:

    dig -t TXT push.apple.com

#### iOS apsd patch

This step assumes you have a codesign certificate in your keychain named `iPhone Developer`, if you prefer another name you can change `patch-apsd.sh`. You also need `ldid` on your iOS device, I'm not sure whether it comes with Cydia by default.

    cd pushproxy
    setup/ios/patch-apsd.sh <device hostname> <14-char DNS entry>

You can find instructions on how to do this manually in `doc/howto-patch-apsd.md`

#### OS X applepushserviced patch

Like the iOS patch step, this step assumes there is a codesign certificate in your keychain named `iPhone Developer`.

    cd pushproxy
    setup/osx/patch-applepushserviced <14-char DNS entry>

This modifies `/System/Library/PrivateFrameworks/ApplePushService.framework/applepushserviced` and place a backup in the same directory named `applepushserviced.orig`.

After a restart the `applepushserviced` would request a new certificate from Apple since the binary has a new signature, so Keychain doesn't allow it to access the old certificate. So just do the 'Extract OS X Certificates' step which includes a restart anyway.

## Running

    cd src
    ./runpush.sh

This should be enough for most cases, if you want it to run as daemon and write output to a logfile in `data/`, create an empty file in src:

    touch production

If you want to change some configuration, just edit `pushserver.py.

## API

### Send notifications

PushProxy offers a [Twisted Perspective Broker](http://twistedmatrix.com/documents/current/core/howto/pb-usage.html) API for sending push notification directly to connected devices. It has the following signature:

    remote_sendNotification(self, pushToken, topic, payload)

It doesn't implement the store part of the store-and-forward architecture Apple's push notifiction system implements, so notifications sent via this API for will be lost for offline devices.

See `src/icl0ud/push/notification_sender.py` for the implementation.

### Message handler

You can subclass `icl0ud.push.dispatch.BaseHandler`, look at `dispatch.py`, `pushtoken_handler.py` and `notification_sender.py` in `src/icl0ud/push`.

Handlers can be configured in `src/pushserver.py`

## Debugging

Apple provides a [document](http://developer.apple.com/library/ios/#technotes/tn2265/_index.html) on debugging push connections. Especially useful are their instructions on how to enable debug logging on iOS and OS X. You can find the download link to the configuration file for iOS in the upper right corner of the page.

The document is a bit outdated. If you want to enable debug logging on newer OS X versions like 10.8.2, you have to replace `applepushserviced` with `apsd` in the `defaults` and `killall` commands.

## Contributors

* Michael Frister
* Martin Kreichgauer

## Thanks

* [Matt Johnston](http://www.ucc.asn.au/~matt/apple/) for writing [extractkeychain](http://www.ucc.asn.au/~matt/src/extractkeychain-0.1.tar.gz) that is used for deriving the master key and decrypting keys from the OS X keychain
* Vladimir "Farcaller" Pouzanov for writing [python-bplist](https://github.com/farcaller/bplist-python) which is included and helps extracting the push private key from the OS X keychain

![pi](https://planb.frister.net/pi/piwik.php?idsite=3&rec=1)
