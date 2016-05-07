.. title: Compile and install Kodi on iPad without jailbreak
.. slug: compile-and-install-kodi-on-ipad-without-jailbreak
.. date: 2016-01-10 22:10:42 UTC+01:00
.. tags: kodi,iOS,iPad,iPhone,Xcode,Mac,OSX
.. category:iOS 
.. link: 
.. description: 
.. type: text

With iOS 9 and Xcode 7 it's finally possible to compile and deploy apps on
your iPhone/iPad with a free Apple developer account (no paid membership
required).

I compiled XBMC/Kodi many times on my mac but had never signed an app with
Xcode before and it took me some time to get it right. 
So here are my notes:

First thanks to memphiz for the `iOS9 support <http://forum.kodi.tv/showthread.php?tid=239610>`_!

I compiled from his ios9_workaround branch, but it has been `merged
<https://github.com/xbmc/xbmc/pull/8250>`_ to
master since::

$ git clone https://github.com/xbmc/xbmc.git Kodi
$ cd Kodi
$ git remote add memphiz https://github.com/Memphiz/xbmc.git
$ git fetch memphiz
$ git checkout -b ios9_workaround memphiz/ios9_workaround


Follow the instructions from the README.ios file::

$ git submodule update --init addons/skin.re-touched
$ cd tools/depends
$ ./bootstrap
$ ./configure --host=arm-apple-darwin
$ make -j4
$ make -j4 -C target/binary-addons
$ cd ../..
$ make -j4 -C tools/depends/target/xbmc
$ make clean
$ make -j4 xcode_depends

Start Xcode and open the Kodi project.
Open the Preferences, and add your Apple ID if not already
done:

.. image:: /images/add_account.png

Select the Kodi-iOS target:

.. image:: /images/kodi_ios_target.png

Change the bundle identifier to something unique and click on *Fix Issue*
to create a provisioning profile.

.. image:: /images/bundle_identifier.png

Connect your device to your mac and select it:

.. image:: /images/device.png

Click on *Run* to compile and install Kodi on your device!
