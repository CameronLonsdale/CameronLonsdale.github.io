---
layout: post
title:  "How does FTK Imager snapshot memory?"
date:   2017-08-06
---

I was recently introduced to the [FTK Imager](http://accessdata.com/products-services/forensic-toolkit-ftk) program in my digital forensics class. As I was using I began wondering, how can this program see all of RAM? According to my Operating Systems knowledge, programs are limited to their own virtual memory, therefore it must have to contact the kernel in order to read other parts of memory. If so, that's a fairly worrying privilege, so how does it work?


I did a very basic analysis, unfortunately I can't spend more time on it because my Advanced Operating Systems subject is my life now, but here's what I found.

Upon commencing dumping memory, FTK makes a sequence of Windows kernel [DeviceIoControl](https://msdn.microsoft.com/en-us/library/windows/desktop/aa363216(v=vs.85).aspx) calls; Which makes sense, because that allows programs to read and write to files & devices.

At the start of the capture, a file is created on our drive as an output file, remember the handle <code class="inline-highlight">0x00000448</code>, it will come back later.

{% highlight text %}
FTK Imager.exe CreateFileW ( "C:\Users\victim\Desktop\memdump.mem", GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL ) 0x00000448
Once created, there is a clear, repeatable pattern of calls that happens until completion of the capture.

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b77e70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b77e70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b78e70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b78e70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b79e70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b79e70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b7ae70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b7ae70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b7be70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b7be70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b7ce70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b7ce70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b7de70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b7de70, 4096 )

FTK Imager.exe DeviceIoControl ( 0x00000450, 2147541020, 0x06b77b20, 12, 0x06b7ee70, 4096, 0x06b76ab8, NULL )
KERNELBASE.dll NtDeviceIoControlFile ( 0x00000450, NULL, NULL, NULL, 0x06b76940, 2147541020, 0x06b77b20, 12, 0x06b7ee70, 4096 )

FTK Imager.exe WriteFile ( 0x00000448, 0x06b77e70, 32768, 0x06b77b0c, NULL )
KERNELBASE.dll NtWriteFile ( 0x00000448, NULL, NULL, NULL, 0x06b77970, 0x06b77e70, 32768, NULL, NULL )
{% endhighlight %}

8 Calls to DeviceIoControl then a Write to our output <code class="inline-highlight">0x00000448</code>.

But what are the DeviceIoControl calls doing? That's where it gets confusing.

{% highlight text %}
BOOL WINAPI DeviceIoControl(
 _In_ HANDLE hDevice,
 _In_ DWORD dwIoControlCode,
 _In_opt_ LPVOID lpInBuffer,
 _In_ DWORD nInBufferSize,
 _Out_opt_ LPVOID lpOutBuffer,
 _In_ DWORD nOutBufferSize,
 _Out_opt_ LPDWORD lpBytesReturned,
 _Inout_opt_ LPOVERLAPPED lpOverlapped
);
{% endhighlight %}

We're interacting with a device handle <code class="inline-highlight">0x00000450</code>, which I could not find mentioned anywhere else in the binary or on the web. The control code is <code class="inline-highlight">2147541020</code> which is also undocumented. The lpInBuffer is the same every time <code class="inline-highlight">0x06b77b20</code>, size 12. The lpOutBuffer increments by 4096 each time, and the size of the output is 4096.

Purely from the fact that the out buffer is much larger, these IoControl calls are most likely performing a read of 4096 bytes from a device. 4096 * 8 is 32768, which is the number of bytes we write to our output file after 8 reads.

So our pattern is, read 32768 bytes from *somewhere* and write that to our output file.

The size 4096 makes me think that each read is of a page mapped from physical memory. Hence, the kernel must be mapping in pages and allow FTK to read from them.

Undocumented windows internals are fun. Maybe someone with more time can find out if we can abuse this :D

**Update**: The driver handle originates from this call: Which is of the form of a driver or physical storage but it's not well documented what that name is.

{% highlight text %}
FTK Imager.exe CreateFileW ( "\\.\addriver", GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL ) 0x000004e4
{% endhighlight %}
