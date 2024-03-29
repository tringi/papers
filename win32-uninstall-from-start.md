:arrow_backward: [Win32 Proposals and ideas](README.md)

# Start Menu: Uninstall command for Win32 apps

Currently Store/Modern/Metro/packaged apps can be immediately Uninstalled from Start Menu
by right-clicking the entry and choosing Uninstall command.
*Legacy* Win32 apps can not. The same menu command opens Apps Settings (or Control Panel) where the user is required
to search for the application again.

This is inconsistent and cumbersome experience.

## Reasons

Win32 apps in Start Menu are .lnk files. Classic old [Shortcuts](https://learn.microsoft.com/en-us/windows/win32/shell/links).
Those small files contain path of the EXE, arguments, hotkey, path to icon (customizable), and a few other things.
But they don't contain anything regarding uninstallation. Thus the OS cannot reliably link them to the app to which they belong.
Any heuristics in this regard could lead to user uninstalling wrong application, which is very bad experience.

## Solution

The [LNK file format](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink/16cb4ca1-9339-4d0c-a68d-bf1d6cc0f943)
is well documented and easily extensible. Windows offers COM object,
[IShellLink](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-ishelllinkw), to manipulate it.

The solution is to extend the LNK format of either:

* **BASIC:** The Uninstall command path  
  *Effectively duplicate the content of `Software\Microsoft\Windows\CurrentVersion\Uninstall\XXXX\UninstallString` value.*

* **IDEAL:** The `XXXX` part of the [Uninstall registry key](https://learn.microsoft.com/en-us/windows/win32/msi/uninstall-registry-key)  
  *The Start Menu can then combine it with the proper key, and find the `UninstallString` value by itself.*  
  *This approach would also allow the Start Menu to retrieve and display additional data set by the installer.*

Then add IShellLink method to write it, e.g.: `SetUninstallKey (LPCTSTR)`.

*This means idiomatically extend IShellLink to **IShellLink2** containing the new method, akin to existing
ITaskbarList, ITaskbarList2, ITaskbarList3 and ITaskbarList4, not changing IShellLink itself, of course.*

## Behavior

The Start Menu, on selecting *Uninstall*, then retrieves this value (optionally loading it from Uninstall key),
and starts the uninstaller directly, without opening Settings or Control Panel.

## Caveat

This should have been done a decade ago.

It requires developers of the installer/uninstaller software to add support in their code.  
A single line, but nonetheless it will take some time to adopt.

Now, contrary to what Microsoft executives and tech journalists would love to believe, Win32 apps aren't going anywhere.
Developers still add new Windows features into their decades old products. At least those that make user's life easier.
That's why I conclude they will adopt this one.
