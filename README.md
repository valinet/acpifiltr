# acpifiltr
WDF-based kernel-mode filter driver based on the [sample from Microsoft](https://github.com/microsoft/Windows-driver-samples/tree/main/general/toaster/toastDrv/kmdf/filter/generic) that signals a user space named event when the ACPI power button is pressed. Basically, this allows mapping the power button to any custom action, besides the limited options that Windows is offering.

Theoretically, this can be used as a filter driver over any device that supports the `IOCTL_GET_SYS_BUTTON_EVENT` ioctls (for example, a PS/2 or USB keyboard that contains a power button); it only happens that I have tested it over the `ACPI Fixed Feature Button` on my setup, which coresponds to the power button on my laptop's keyboard and on the docking station. Adapt the steps below to your own setup. In the light of this, indeed `acpifiltr` is not necessarily the best name, but I kept it this way because, for some reason, I can remember it easily.

## How to?

First of all, obviously, compile the driver on your machine (I have tested it solely on `x64` architecture). You need to have installed the [WDK](https://learn.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk) for this.

To run in a secure manner, I recommend [this](https://github.com/valinet/ssde), which is implied in the installation procedure below. So, to start, ensure ssde runs with Secure Boot on (`Confirm-SecureBootUEFI` in PowerShell); no need to enable test signing or Test Mode.

For best results, adjust Windows so that pressing the power button does nothing. Open Control Panel (`control.exe`), go to `System and Security\Power Options\Change what the power buttons do` and set `When I press the power button:` to `Do nothing` for both `On battery`, and `Plugged in`.

Remove the test signature from the driver:

```
signtool remove /s acpifiltr.sys
```

Sign the driver with the certificate created for ssde. Replace `password` with your certificate's password, and `17cf4521f162442bf61d3a09ec8c4455456eaf54` with the SHA1 of your `localhost-km.der` file.

```
signtool sign /fd sha256 /ac .\localhost-root-ca.der /f .\localhost-km.pfx /p password /tr http://sha256timestamp.ws.symantec.com/sha256/timestamp /td SHA256 /sha1 17cf4521f162442bf61d3a09ec8c4455456eaf54 acpifiltr.sys
```

Copy driver to `C:\Windows\System32\drivers`.

```
del C:\Windows\System32\drivers\acpifiltr.sysO
ren C:\Windows\System32\drivers\acpifiltr.sys acpifiltr.sysO
xcopy /Y acpifiltr.sys C:\Windows\System32\drivers
```

Install driver in the system.

```
sc create acpifiltr binpath=%windir%\system32\drivers\acpifiltr.sys type=kernel start=demand error=normal
```

Register driver as filter for the `ACPI Fixed Feature Button` device. In the Windows Registry, go to `HKLM\SYSTEM\CurrentControlSet\Enum\ACPI\FixedButton`. In there, you probably have got a subfolder - mine is called `2&daba3ff&0`. Open that; it should contain various 'things', like entries for `HardwareID`, `Driver` etc. Create a new `REG_MULTI_SZ` and call it `UpperFilters`, if not already there. Edit `UpperFilters` and add `acpifiltr` at the top of the list.

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\ACPI\FixedButton\2&daba3ff&0]
"UpperFilters"=hex(7):61,00,63,00,70,00,69,00,66,00,69,00,6c,00,74,00,72,00,00,\
  00,00,00
```

Now, restart the system; and after reboot the driver should be up and running. You can confirm using `sc query` which should report the status as `RUNNING`:

```
sc query acpifiltr
```

When the driver is functioning and the user presses the power button, it will signal the `Global\{29DFB10F-5CFF-4A9D-B9D1-31B053A1AE95}` named event. This event should be created by a user land application which should wait on it, and then perform whatever action the user desires when the event gets signaled.

For example, [here](https://github.com/valinet/wh-mods/blob/master/mods/valinet-power-button-action.wh.cpp) is a Windhawk module that opens `calc.exe` when the driver is installed and the user presses the power button.

Enjoy.

## References

* https://github.com/microsoft/Windows-driver-samples/tree/main/general/toaster/toastDrv/kmdf/filter/generic
* https://web.archive.org/web/20130313055404/http://support.microsoft.com/kb/302092
* https://learn.microsoft.com/en-us/archive/blogs/doronh/how-are-power-buttons-reported-in-windows
* https://learn.microsoft.com/en-us/archive/blogs/doronh/how-ps2-and-hid-keyboads-report-power-button-events
* https://www.techtalkz.com/threads/acpi-intercepting-power-button-and-lid-switch-questions-xp.259038/
* https://community.osr.com/t/how-to-track-the-power-button-press-event-in-kernel/44660
* https://community.osr.com/t/capturing-physical-power-button-event-in-windows-xp-embedded/48075/18
* https://stackoverflow.com/questions/5769786/installing-filter-drivers-to-an-existing-device
