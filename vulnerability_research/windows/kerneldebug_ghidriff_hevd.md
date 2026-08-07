## Latest Notes ##

:exclamation: ESXi adapters (vmxnet) don't play well with KDNET!

Setup target (debugee) for debugging:
```
bcdedit /debug on
bcdedit /dbgsettings net hostip:<YOUR_DEBUGGER_IP> port:50000 nodhcp
bcdedit /set "{dbgsettings}" busparams <output from kdnet for adapter>
bcdedit /set testsigning on
```

Check if supported adapter is present:
```
kdnet.exe
```

Debugging lsass.exe:
```
!process 0 0 lsass.exe
.process /i /p <EPROCESS address>
g

.reload /user
lm
```




Ghidriff:

ghidra_diff_engine.py patch:
```
@@ -1618,7 +1618,13 @@
             self.normalize_ghidra_decomp(new_code)
 
             old_code_no_sig = self.remove_code_sig(ematch_1['code'])
+            if (old_code_no_sig == ""):
+                old_code_no_sig = []
+
             new_code_no_sig = self.remove_code_sig(ematch_2['code'])
+            if (new_code_no_sig == ""):
+                new_code_no_sig = []

             self.normalize_ghidra_decomp(old_code_no_sig)
             self.normalize_ghidra_decomp(new_code_no_sig)
```


```
ghidriff.exe .\apr25_vuln_clfs.sys .\may25_patched_clfs.sys
```


```
```


HEVD:

Load driver:
```
sc create HEVD type= kernel binPath= "<path_to_driver>"
sc start HEVD
```

IOCTL decode in windbg:
```
!ioctldecode 0x222003
```

DbgPrintEx mask in windbg:
Given the DbgPrintEx() call, determine the component name (arg1) and log level (arg2). arg1 maps to
```
https://github.com/tpn/winsdk-10/blob/master/Include/10.0.16299.0/shared/dpfilter.h
```

For example, Ghidra shows the call as DbgPrintEx(0x4d,3,"message"). 0x4d converted to decimal is 77 so the mask we need to set is the following: 
```
ed nt!Kd_IHVDRIVER_Mask 0xFFFFFFFF
dd nt!Kd_IHVDRIVER_Mask
```

With the mask set, we can now see the proper HEVD DbgPrintEx messages in Windbg.


windbg analyze bugcheck:
```
!analyze -v
```

windbg reboot:
```
.reboot
```

OR

snapshot/revert with:
```
.reload
```



