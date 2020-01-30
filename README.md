# linuxcnc
Personal LinuxCNC configurations

Notes on the SVD-PS VFD (LAPOND, Shenzhen Soyanpower New Energy Co.,Ltd)

Docs: http://www.soyan-tech.com/uploads/20189675/PDF/SVD-p-User-Manual.pdf

Only the following settings for Modbus seem to work (for instance changing the default baud rate will break communication)

{code}
PD-00 6005
PD-01 0
PD-02 1
PD-03 2
PD-04 0
PD-05 1 <---- Note the change from the default settings
PD-06 0
PD-08 0
{code}

Modbus registers:

{code}
0x2000 { 1: Forward run
         2: Reverse run
         3: Forward JOG
         4: Reverse JOG
         5: Stop
       }
{code}
