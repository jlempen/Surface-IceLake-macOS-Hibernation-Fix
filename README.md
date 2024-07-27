# Surface-IceLake-macOS-Hibernation-Fix
After many hours of tinkering, **I finally found a way to fix ACPI S4 Hibernation** (`hibernatemode 25`) on the **i5 Surface Pro 7** on macOS 13.6.7 Ventura, **15-Inch i7 Surface Laptop 3** on macOS 14.5 Sonoma, **i5 Surface Laptop Go 1** on macOS 13.6.7 Ventura and very likely on the **Surface Book 3** (untested) as well. ACPI S3 Sleep (`hibernatemode 0`) is still broken, though, but perhaps the Hibernation fix will lead the way to a full S3 Sleep fix at some point in the future.

The key to the fix is to **enable ACPI S3 Sleep** in the DSDT. This is actually very easy and I'm stunned nobody tried this before. Searching for `_S3` in the `DSDT.aml` file leads us to this:

```
    If (SS3)
    {
        Name (_S3, Package (0x04)  // _S3_: S3 System State
        {
            0x05, 
            Zero, 
            Zero, 
            Zero
        })
    }
```
So to enable `_S3`, `SS3` needs to return `One`. Now searching for `SS3` leads us to this:
```
    Name (SS1, Zero)
    Name (SS2, Zero)
    Name (SS3, Zero)
    Name (SS4, One)
```


## Enable ACPI S3 Sleep
Change `Name (SS3, Zero)` to `Name (SS3, One)` to enable **ACPI S3 Sleep** with the following ACPI patch in `ACPI -> Patch`:
```
Comment: Rename Name (SS3, Zero) to One - Enable S3 Sleep
Find: 5353335F00
Replace: 5353335F01
```
Before the rename, the system log shows:
> (AppleACPIPlatform) ACPI: sleep states S4 S5

After the rename, the system log shows:
> (AppleACPIPlatform) ACPI: sleep states S3 S4 S5

and commands such as `pmset -g log` or `pmset -g log | grep -e "Sleep.*due to" -e "Wake.*due to"` now clearly show that the device is entering sleep and (eventually) waking up from sleep.

## Add Reserved Memory region
This fix was found in [Tyler Nguyen's x1c6-hackintosh repo](https://github.com/tylernguyen/x1c6-hackintosh/issues/44#issuecomment-705971765). Thank you @benbender @1Revenger1 @savvamitrofanov and @vit9696!

Get rid of the black screen on Wake with the following Reserved Memory region in `NVRAM -> ReservedMemory`:
```
Comment: Fix hibernate mode 25 black screen on wake
Address: 569344
Size: 4096
Type: RuntimeCode
Enabled: true
```

## Enable DiscardHibernateMap
This doesn't seem to be required on every Surface device, but at least on the Surface Laptop 3, it fixes a kernel panic on Wake.

Enable the following Booter Quirk in `Booter -> Quirks`:
```
DiscardHibernateMap=true
```

## Set HibernateMode to NVRAM
In `Misc -> Boot`, set `HibernateMode` to `NVRAM`:
```
HibernateMode=NVRAM
```
I would also recommend enabling `HibernateSkipsPicker` in `Misc -> Boot`:
```
HibernateSkipsPicker=true
```

## Add the GPRW instant wake patch
At least on the **Surface Pro 7** and the **Surface Laptop Go 1**, the GPRW instant wake patch is required as well. 

Add [SSDT-GPRW.aml](https://github.com/jlempen/Surface-IceLake-Hibernation-Fix/blob/main/SSDT-GPRW.aml) to the ACPI folder of your EFI. Then add `SSDT-GPRW.aml` to `ACPI -> Add` and the following ACPI patch to `ACPI -> Patch` in the `config.plist` file:
```
Comment: Change Method(GPRW,2,N) to XPRW, pair with SSDT-GPRW.aml
Find: 4750525702
Replace: 5850525702
```

## Enable hibernatemode 25 in macOS
Open the `Terminal` and enter the following commands, then reboot for the changes to take effect:
```
sudo pmset restoredefaults
sudo pmset -a hibernatemode 25
```
If for whatever reason Hibernate is still not working on your system, you should reset the `Power Management` settings and rebuild the `sleepimage` file. To do so, open the `Terminal` and enter the following commands, then reboot for the changes to take effect:
```
sudo rm /Library/Preferences/com.apple.PowerManagement*
sudo rm /var/vm/sleepimage
sudo pmset hibernatefile /var/vm/sleepimage
```
Once you are back in macOS, disable Sleep and enable Hibernate again, then reboot:
```
sudo pmset restoredefaults
sudo pmset -a hibernatemode 25
```

## Fix broken Bluetooth on Wake from Hibernation
After the device wakes up from Hibernation, Bluetooth may be broken / unable to connect.

A very simple fix for this issue is to [download and install Bluesnooze](https://github.com/odlp/bluesnooze). Launch the app, enable `Launch at login` and you're done!

## A few comments / Help needed!
Hibernation seems to be working perfectly on my i5 / 8GB / 256GB **Surface Pro 7** and i5 / 8GB / 256GB **Surface Laptop Go 1** both running on macOS Ventura 13.6.7 with the SMBIOS `MacBookAir9,1` and using [Xiashangning's BigSurface.kext v6.5](https://github.com/Xiashangning/BigSurface/releases/tag/v6.5). Everything comes back online after waking up from hibernation.

However, my i7 / 16GB / 2TB WD SN770M **15-Inch Surface Laptop 3** running macOS Sonoma 14.5 with the SMBIOS `MacBookPro16,2` and with [Xiashangning's BigSurface.kext v6.2](https://github.com/Xiashangning/BigSurface/releases/tag/v6.2) wakes up with a dead trackpad. I have yet to find a way to fix the dead trackpad.

The main difference between the **Surface Pro 7** or **Surface Laptop Go 1** and the **Surface Laptop 3** or **Surface Book 3** is that on the former, the keyboard and trackpad are attached through USB, whereas on the latter, they are attached through a proprietary interface.

Both the **Surface Laptop 3** and the **Surface Book 3** are plagued by a nasty issue with the trackpad on macOS. Fixing the issue is possible, but [requires downgrading the firmware to 13.101.140.0 and BigSurface.kext to 6.2](https://github.com/Xiashangning/BigSurface/issues/79#issuecomment-2208484390), that's why it still runs with `BigSurface.kext` v6.2.
The trackpad works with the latest `BigSurface.kext` v6.5, though, but the trackpad lags and skips every few seconds. Furthermore, tests show that the `BigSurface.kext` v6.5 doesn't fix the dead trackpad after wake from hibernation.

Playing around trying to unload the `BigSurface.kext` or `VoodooSerial.kext` plug-in before hibernate and reloading it on wake from hibernate has been a dead end until now, as I'm unable to do so due to various errors with the `kextstat`, `kextunload` and `kextload` commands in macOS.

So, the first steps to troubleshoot the trackpad issue would be to downgrade macOS to 13.6.7 Ventura and change the SMBIOS back to `MacBookAir9,1`, but I must admit I'm not all too confident that would fix the dead trackpad after wake from hibernation. We would probably need to find someone who is able to fix the `BigSurface.kext` and I'm not sure Xiashangning is still maintaining his `BigSurface` repo, as there hasn't been any activity for over a year now, [since July 14th 2023](https://github.com/Xiashangning/BigSurface/issues/109#issuecomment-1636433545).
