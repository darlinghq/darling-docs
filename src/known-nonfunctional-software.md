# Known non-functional software

The following software has been tested and is known not to work with Darling:

- XQuartz 2.8.1:
  - Hangs, and causes client applications to hang when they try to use it.
- Macports (though Homebrew does work).

GUI applications will not work in Darling at this point in time, with very few exceptions. More specifically:

- Most GUI toolkits, including:
  - Anything using the Python Tk/Tkinter toolkits.
  - Anything using the WxPython and WxWidgets toolkits.
  - Anything using the Xamarin/MAUI toolkits.
  - Mac Catalyst applications (there is no UIKit implementation for Darling yet).
- Most GUI applications, including:
  - The Xcode GUI.
  - Logic.
  - Final Cut Pro.
  - Any Adobe Suite applications.
  - Any complex GUI application in general will not work at this point in time.
