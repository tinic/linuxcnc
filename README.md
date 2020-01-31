# linuxcnc
Personal LinuxCNC configurations

Notes on the SVD-PS VFD (LAPOND, Shenzhen Soyanpower New Energy Co.,Ltd)

Docs: http://www.soyan-tech.com/uploads/20189675/PDF/SVD-p-User-Manual.pdf

Only the following settings for Modbus seem to work (for instance changing the default baud rate will break communication)

```
PD-00 6005
PD-01 0
PD-02 1
PD-03 2
PD-04 0
PD-05 1 <---- Note the change from the default settings
PD-06 0
PD-08 0
```

- Serial settings: 9600 baud, 8N2
- Slave address is 1
- All modbus registers are 16-bit.

Modbus read/write registers:

```
0x1000 { 0-10000: 0-100% speed in %/100}
```


Modbus read registers:

```
0x1001 { 0-12000: running frequency in Hz/100 }
0x1002 { 0-800: DC BUS voltage V/10 }
0x1003 { 0-800: Output voltage V }
0x1004 { 0-800: Output current A/100 }
0x1005 { 0-10000: 0-100% running speed in %/10 }
0x1009 { 0-65535: input terminal }
0x100a { 0-1000: AI1 }
0x100b { 0-1000: AI2? }
```

Modbus write registers:

```
0x2000 { 1: Forward run
         2: Reverse run
         3: Forward JOG
         4: Reverse JOG
         5: Stop
       }
```
