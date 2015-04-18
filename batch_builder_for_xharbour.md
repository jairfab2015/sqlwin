
```
@Echo Off
Cls
Set hbdir=C:\xHarbour
Set bcdir=C:\bcc55
Set fwdir=C:\FwH
 
If Exist TestSqlw.exe Del TestSqlw.Exe
 
%hbdir%\bin\harbour TestSqlw /m/n/a/w0 /i%fwdir%\include;%hbdir%\include /w /p TestSqlw.C  > Erro.log
%hbdir%\bin\harbour SqlWin /m/n/a/w0 /i%fwdir%\include;%hbdir%\include /w /p SqlWin.C    >> Erro.log
%bcdir%\bin\bcc32 -O2 -M -c -D__HARBOUR__ -I%hbdir%\include TestSqlw.C    >> Erro.log
%bcdir%\bin\bcc32 -O2 -M -c -D__HARBOUR__ -I%hbdir%\include SqlWin.C  >> Erro.log
 
If ErrorLevel 1 Type Erro.log | More
If ErrorLevel 1 Goto Exit
 
:ENDCOMPILE
 
echo c0w32.obj                     + > xHrb.lnk
echo TestSqlw.obj                  + >> xHrb.lnk
echo SqlWin.obj,                   + >> xHrb.lnk
echo TestSqlw.exe,                 + >> xHrb.lnk
echo TestSqlw.map,                 + >> xHrb.lnk
echo %fwdir%\lib\Fivehx.lib        + >> xHrb.lnk
echo %fwdir%\lib\Fivehc.lib        + >> xHrb.lnk
echo %hbdir%\lib\rtl.lib           + >> xHrb.lnk
echo %hbdir%\lib\usrrdd.lib        + >> xHrb.lnk
echo %hbdir%\lib\vm.lib            + >> xHrb.lnk
echo %hbdir%\lib\gtgui.lib         + >> xHrb.lnk
echo %hbdir%\lib\lang.lib          + >> xHrb.lnk
echo %hbdir%\lib\macro.lib         + >> xHrb.lnk
echo %hbdir%\lib\rdd.lib           + >> xHrb.lnk
echo %hbdir%\lib\dbfntx.lib        + >> xHrb.lnk
echo %hbdir%\lib\dbfcdx.lib        + >> xHrb.lnk
echo %hbdir%\lib\dbffpt.lib        + >> xHrb.lnk
echo %hbdir%\lib\hbsix.lib         + >> xHrb.lnk
echo %hbdir%\lib\debug.lib         + >> xHrb.lnk
echo %hbdir%\lib\common.lib        + >> xHrb.lnk
echo %hbdir%\lib\pp.lib            + >> xHrb.lnk
echo %hbdir%\lib\ct.lib            + >> xHrb.lnk
echo %hbdir%\lib\pcrepos.lib       + >> xHrb.lnk
echo %bcdir%\lib\cw32.lib          + >> xHrb.lnk
echo %bcdir%\lib\import32.lib      + >> xHrb.lnk
echo %bcdir%\lib\psdk\odbc32.lib   + >> xHrb.lnk
echo %bcdir%\lib\psdk\rasapi32.lib + >> xHrb.lnk
echo %bcdir%\lib\psdk\nddeapi.lib  + >> xHrb.lnk
echo %bcdir%\lib\msimg32.lib       + >> xHrb.lnk 
echo %bcdir%\lib\psdk\iphlpapi.lib   >> xHrb.lnk
%bcdir%\bin\ilink32 -Gn -aa -Tpe -s -v @xHrb.lnk
 
IF ERRORLEVEL 1 GOTO LINKERROR
TestSqlw.exe
Goto Exit
:LINKERROR
:Exit
@If Exist *.Obj      Del *.Obj
@If Exist *.Log      Del *.Log
@If Exist *.Map      Del *.Map
@If Exist *.PPO      Del *.Ppo
@If Exist *.C        Del *.C
```