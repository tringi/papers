:arrow_backward: [Win32 Proposals and ideas](README.md)

*Jan Ringoš*
# Start Menu: Uninstall command for Win32 apps

Currently Store/Modern/Metro/packaged apps can be immediately Uninstalled from Start Menu
by right-clicking the entry and choosing Uninstall command.

*Legacy* Win32 apps can not. The same menu command opens Apps Settings (or Control Panel) whe the user is required
to search for the application again.

This is inconsistent and cumbersome experience.

## Reasons

Win32 apps in Start Menu are .lnk files. Classic old Shortcuts.
Those small files contain path of the EXE, arguments, hotkey, path to icon (customizable),
and a few other things. But they don't contain anything regarding uninstallation.

## Solution

The LNK file format is easily extensible, and Windows offer COM object, IShellLink, to manipulate it.
The solution is to extend the LNK format and add one of the following:

* BASIC: Add the Uninstall command path  
  *Effectively duplicate the content of `Software\Microsoft\Windows\CurrentVersion\Uninstall\XXXX\UninstallString` value.*

* UNIVERSAL: Add the `XXXX` part of the registry key  
  *The Start Menu can then combine appropriate key, and find the `UninstallString` value by itself.*  
  *This approach also allows the Start Menu to retrieve and display additional data set by the installer.*

Then add IShellLink method to write it, e.g.: `SetUninstallKey`.

*This means idiomatically extend IShellLink to **IShellLink2** containing the new method, akin to existing
ITaskbarList, ITaskbarList2, ITaskbarList3 and ITaskbarList4, not changing IShellLink itself, of course.*

## Behavior

The Start Menu, on selecting Uninstall, will then retrieved this value (optionally loading it from Uninstall key),
and starts the installer directly, without opening Settings or Control Panel.

Now, it is true that that would also require developers of the installer/uninstaller software to add a support in their code,
a single line, but seeing how eagerly everyone still adds new Windows features, or at least those that make user's life easier,
I'm sure most would do that immediately.
