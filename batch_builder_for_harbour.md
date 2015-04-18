
```
@Echo Off
Cls
Set hbdir=C:\Harbour
Set bcdir=C:\bcc55
Set fwdir=C:\FwH
 
If Exist TestSqlw.exe Del TestSqlw.Exe
 
%hbdir%\bin\harbour TestSqlw /n /i%fwdir%\include;%hbdir%\include /w /p %2 %3 > erro.log
%hbdir%\bin\harbour SqlWin /n /i%fwdir%\include;%hbdir%\include /w /p %2 %3 > erro.log
%bcdir%\bin\bcc32 -O2 -M -c -D__HARBOUR__ -I%hbdir%\include TestSqlw.C    >> Erro.log
%bcdir%\bin\bcc32 -O2 -M -c -D__HARBOUR__ -I%hbdir%\include SqlWin.C  >> Erro.log
 
If ErrorLevel 1 Type Erro.log | More
If ErrorLevel 1 Goto Exit
 
:ENDCOMPILE
 
echo c0w32.obj                     + >  Harb.lnk
echo TestSqlw.obj                  + >> Harb.lnk
echo SqlWin.obj,                   + >> Harb.lnk
echo TestSqlw.exe,                 + >> Harb.lnk
echo TestSqlw.map,                 + >> Harb.lnk
echo %fwdir%\lib\FiveH.lib         + >> Harb.lnk
echo %fwdir%\lib\FiveHC.lib        + >> Harb.lnk
echo %hbdir%\lib\usrrdd.lib        + >> Harb.lnk
echo %hbdir%\lib\rtl.lib           + >> Harb.lnk
echo %hbdir%\lib\vm.lib            + >> Harb.lnk
echo %hbdir%\lib\gtgui.lib         + >> Harb.lnk
echo %hbdir%\lib\lang.lib          + >> Harb.lnk
echo %hbdir%\lib\macro.lib         + >> Harb.lnk
echo %hbdir%\lib\rdd.lib           + >> Harb.lnk
echo %hbdir%\lib\dbfntx.lib        + >> Harb.lnk
echo %hbdir%\lib\dbfcdx.lib        + >> Harb.lnk
echo %hbdir%\lib\dbffpt.lib        + >> Harb.lnk
echo %hbdir%\lib\hbsix.lib         + >> Harb.lnk
echo %hbdir%\lib\debug.lib         + >> Harb.lnk
echo %hbdir%\lib\common.lib        + >> Harb.lnk
echo %hbdir%\lib\pp.lib            + >> Harb.lnk
echo %hbdir%\lib\codepage.lib      + >> Harb.lnk
echo %hbdir%\lib\hbwin32.lib       + >> Harb.lnk
echo %bcdir%\lib\cw32.lib          + >> Harb.lnk
echo %bcdir%\lib\import32.lib      + >> Harb.lnk
echo %bcdir%\lib\psdk\odbc32.lib   + >> Harb.lnk
echo %bcdir%\lib\psdk\nddeapi.lib  + >> Harb.lnk
echo %bcdir%\lib\psdk\iphlpapi.lib + >> Harb.lnk
echo %bcdir%\lib\psdk\rasapi32.lib   >> Harb.lnk
%bcdir%\bin\ilink32 -Gn -aa -Tpe -s -v @Harb.lnk
 
IF ERRORLEVEL 1 GOTO LINKERROR
TestSqlw.exe
Goto Exit
:LINKERROR
:Exit
@If Exist *.Obj      Del *.Obj
@If Exist *.Log      Del *.Log
@If Exist *.Map      Del *.Map
@If Exist *.Map      Del *.Map
@If Exist *.PPO      Del *.Ppo
@If Exist *.Tds      Del *.Tds
@If Exist *.C        Del *.C
```