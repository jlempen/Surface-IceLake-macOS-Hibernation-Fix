# Surface-IceLake-Hibernation-Fix
To fix Hibernation (`hibernatemode 25`) in macOS on the **Surface Pro 7**, **Surface Laptop 3**, **Surface Laptop Go 1** and very likely also on the **Surface Book 3**, make the following changes to your `config.plist` file:

## Enable ACPI S3 Sleep
Change `Name (SS3, Zero)` to `Name (SS3, One)` to enable **ACPI S3 Sleep** with the following ACPI patch in `ACPI -> Patch`:
```
Comment: Rename Name (SS3, Zero) to One - Enable S3 Sleep
Find: 5353335F00
Replace: 5353335F01
```

## Add Reserved Memory region
Get rid of the black screen on Wake with the following Reserved Memory region in `NVRAM -> ReservedMemory`:
```
Comment: Fix hibernate mode 25 black screen on wake
Address: 569344
Size: 4096
Type: RuntimeCode
Enabled: true
```

## Enable DiscardHibernateMap
Enable the following Booter Quirk in `Booter -> Quirks`:
```
DiscardHibernateMap=true
```

## Set HibernateMode to NVRAM
In `Misc -> Boot`, set `HibernateMode` to `NVRAM`:
```
HibernateMode=NVRAM
```

## Add the GPRW instant wake patch
At least on the **Surface Pro 7** and the **Surface Laptop Go 1**, the GPRW instant wake patch is required as well. Add `SSDT-GPRW.aml` to the ACPI folder of your EFI. Then add `SSDT-GPRW.aml` and the following ACPI patch to the `config.plist` file in `ACPI -> Patch`:
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
