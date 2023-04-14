:arrow_backward: [Win32 Proposals and ideas](README.md)

How to fix:
# DWM: ClearType on composited/translucent surfaces

[ClearType](https://en.wikipedia.org/wiki/ClearType) font smoothing is slowly but surely disappearing from Windows.
This is making *modern* apps look increasingly more ugly and difficult to read at normal scaling.
As the adoption of high-DPI displays is lagging, I took the liberty of designing a simple [solution](#the-solution).

## History and technical reasons

Windows Vista introduced [DWM](https://learn.microsoft.com/en-us/windows/win32/dwm/dwm-overview) composition.
Applications now present surfaces to DWM instead of painting directly into a desktop's Device Context.
These surfaces can be partially transparent and are later composited using GPU acceleration.
This allows them to be smoothly and efficiently animated.
Which makes it a technology of choice of modern UI frameworks to implement individual controls.

ClearType text cannot be rendered onto transparent surface,
because it needs to know the pixels underneath it (*to properly blend sub-pixel colors*).
And those aren't known until composition time.

Windows Vista and 7 mitigated the most obvious case of text on composited surface, window caption, 
by blacking the transparency by a shadow first, then drawing ClearType text onto it. Windows 8 removed
the shadow and forced gray-scale antialiasing. It was all downhill from there...

## The solution

...is to defer text rendering to composition phase. Render the text in DWM. Or rather, add the option to do so.

Provide API that will allow the application to tell the DWM to render one or more strings when compositing the window.
At that time DWM knows underlying pixels and should be able to efficiently render ClearType, even taking advantage of HW acceleration.

## API

Following APIs are drafted for GDI, roughly reflecting existing
[DrawShadowText](https://learn.microsoft.com/en-us/windows/win32/api/commctrl/nf-commctrl-drawshadowtext) and
[DrawThemeTextEx](https://learn.microsoft.com/en-us/windows/win32/api/uxtheme/nf-uxtheme-drawthemetextex) APIs
for consistency, but other design are possible.

```cpp
DwmSetDeferredText (HWND hWnd, LONG_PTR id, HFONT, DWORD flags, RECT *, DTTOPTS *, LPCWSTR, int);
DwmSetDeferredThemeText (HWND hWnd, LONG_PTR id, HTHEME, int part, int state, DWORD flags, RECT *, DTTOPTS *, LPCWSTR, int);
```

* `id` uniquely identifies the string on the `hWnd` and using same value replaces existing string
* `NULL` for `LPCWSTR` parameter to remove string, or call `DwmRemoveDeferredText (HWND, LONG_PTR);`?

Arguably, the number of parameters is excessive, and passing them in a structure might improve readability, for example:

```cpp
struct DWM_DEFERRED_TEXT_PARAMS {
    RECT r;
    DWORD flags;
    DTTOPTS opts;
};

DwmSetDeferredText (HWND hWnd, LONG_PTR id, HFONT, const DWM_DEFERRED_TEXT_PARAMS *, LPCWSTR, int);
```





