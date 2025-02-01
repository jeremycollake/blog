# A Hazard of ExtractAssociatedIcon and ExtractAssociatedIconEx

A couple years ago I encountered a strange crash on a customer system. After diving deep into it, I found memory corruption had occurred. But how?

Well, I eventually traced it to a call to [ExtractAssociatedIcon](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-extractassociatediconw) / [ExtractAssociatedIconEx](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-extractassociatediconexw). At the time, I neglected to properly document this hazard, so this Gist exists to warn others about the unsafe design of these APIs and their incomplete documentation.

Do see anything wrong with this code?

`ExtractAssociatedIcon(GetModuleHandle(nullptr), “c:\\program files\\app\\anexe.exe”, &wIcon);`

Looks pretty typical, eh? It’d be the intuitive way to call ExtractAssociatedIcon without further thought, and you’ll find countless examples on the web calling it just that way.

The problem? Let’s look at the function prototype:

```
HICON ExtractAssociatedIconA(
  [in]      HINSTANCE hInst,
  [in, out] LPSTR     pszIconPath,
  [in, out] WORD      *piIcon
);
```

See the issue now? pszIconPath is non-const and annotated as in/out. Uh oh! That means this API can modify the buffer we passed in. That’s surprising since there is no buffer length parameter. Had such as parameter existed, then it’d have been obvious that the API may modify that buffer.

So, when does it choose to modify the pszIconPath buffer and what is the minimum size the buffer should be? 

Turns out it modifies the buffer only when the associated icon for that file was found in some other module, for instance a DLL. In that case, it’ll fill the buffer with the path of the file that actually contains the associated icon. For our purposes, that was a very rare situation, so this code had been in production for some time before a crash was observed.

From the [docs](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-extractassociatediconw):

> The ExtractAssociatedIcon function first looks for the indexed icon in the file specified by lpIconPath. If the function cannot obtain the icon handle from that file, and the file has an associated executable file, it looks in that executable file for an icon. Associations with executable files are based on file name extensions and are stored in the per-user part of the registry.

As for the minimum size, the documentation makes no reference to that. We can assume it’s MAX_PATH, but on Windows that can be expanded and it’s unclear how the API would behave in those situations.

So, as you can now deduce, the problem was that the buffer passed to ExtractAssociatedIcon was, in rare cases, being overwritten, sometimes resulting in a buffer overflow.

Although these APIs are not deprecated, there are newer Shell32 APIs like [SHGetFileInfo](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shgetfileinfow) to perform this task, and my recommendation is to use them instead.
