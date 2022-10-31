@echo off
SETLOCAL ENABLEDELAYEDEXPANSION
rem ---------------------------------------------------------------------
rem #環境変数定義 [ユーザ定義可]
rem ---------------------------------------------------------------------

rem バックアップファイル保存場所
set WORKDIR=C:\regbackup

rem WSUSサーバ
set WU_SRV=http://10.13.161.

rem WSUSステータスサーバ
set WU_STATSRV=http://10.13.161.

rem ターゲットのグループ名
set TARGET_GRP_NAME=DMZ1001_1318

rem 自動更新オプション
set AUOPT=2

rem 自動更新日(曜日or毎日) ※自動更新オプションが4の場合のみ有効
set SCHEDULEDDAY=0

rem 自動更新時(0～23) ※自動更新オプションが4の場合のみ有効
set SCHEDULEDTIME=3

rem 初回起動と判断された時にSUSIDをリセットするか(1：する/1以外:しない)
set SUSIDRESET=1

rem **ADD 20170526 Start
rem Windows10用の特殊処理を実施するか
set FORWIN10=0
ver | find "Version 10" > nul
if %errorlevel% EQU 0 (set FORWIN10=1)
rem **ADD 20170526 end


rem ---------------------------------------------------------------------
rem #環境変数定義 [ユーザ定義不可] ※変更すると正常に動作しません
rem ---------------------------------------------------------------------
SET RKEY=HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
SET RKEY2=HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate
SET TODAY=%DATE:~-10,4%%date:~-5,2%%date:~-2,2%
SET TIME2=%TIME: =0%
SET NOW=%TIME2:~0,2%%TIME2:~3,2%%TIME2:~6,2%
SET TIMESTAMP=%TODAY%%NOW%
SET BKFILE=%TIMESTAMP%_winupdate.reg
SET BKFILE2=%TIMESTAMP%_susid.reg
SET FLAGFILE=.wsus
rem バックアップを取得できたか(取得できなかったら処理を中断)
SET BKFLAG=0
SET BKFLAG2=0
rem ---------------------------------------------------------------------
rem #事前準備
rem ---------------------------------------------------------------------
mkdir %WORKDIR% 2>nul 1>nul
echo ■■■■■■■■■■■■■■■■■■■■■■
echo   WSUS クライアント 設定bat
echo ■■■■■■■■■■■■■■■■■■■■■■
cd %WORKDIR% 2>nul 1>nul >C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Dir)

rem add start 2010/6/30
rem ----------------------------------------------------------------------
rem #レジストリ参照確認 ※参照できない場合は新規にキーのみ作成する
rem ----------------------------------------------------------------------
reg query %RKEY% 2>nul 1>nul
if %errorlevel% EQU 1 (
reg add %RKEY% /ve >>C:\wsus.log
echo [INFO]RKEYを参照できません。RKEYを新規に作成しました。　>>C:\wsus.log
)

reg query %RKEY2% 2>nul 1>nul
if %errorlevel% EQU 1 (
reg add %RKEY2% /ve >>C:\wsus.log
echo [INFO]RKEY2を参照できません。RKEY2を新規に作成しました。　>>C:\wsus.log
)
rem add end 2010/6/30

rem ---------------------------------------------------------------------
rem #レジストリ情報のバックアップ処理
rem ---------------------------------------------------------------------
echo [INFO]WSUS-Client関連情報をバックアップします。
reg export %RKEY% %BKFILE% 2>nul 1>nul >>C:\wsus.log
rem add start 2010/06/28
if %ERRORLEVEL% NEQ 0 (goto FAIL_Backup)
rem add end 2010/06/28
rem del start 2010/06/28
rem if %ERRORLEVEL% NEQ 0 (set BKDO=1)
rem del end 2010/06/28

reg export %RKEY2% %BKFILE2% 2>nul 1>nul >>C:\wsus.log
rem add start 2010/06/28
if %ERRORLEVEL% NEQ 0 (goto FAIL_Backup)
rem add end 2010/06/28
rem del start 2010/06/28
rem if %ERRORLEVEL% NEQ 0 (set BKDO2=1)
rem del end 2010/06/28

rem ---------------------------------------------------------------------
rem #初回起動時(FLAGFILEが存在しない)は、SUSIDのリセット
rem ---------------------------------------------------------------------
if not exist %WORKDIR%\%FLAGFILE% (
if %SUSIDRESET% EQU 1 (
echo [INFO]初回起動と判断。既存SUSIDが存在すればリセットします。
reg delete %RKEY2% /f /v SusClientId 2>nul 1>nul >>C:\wsus.log
)
)

rem ---------------------------------------------------------------------
rem #レジストリ情報の登録処理
rem ---------------------------------------------------------------------
echo [INFO]設定を変更します。
rem 自動更新定義
reg add %RKEY%\AU /f /v NoAutoRebootWithLoggedOnUsers /t REG_DWORD /d 1 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY%\AU /f /v NoAutoUpdate /t REG_DWORD /d 0 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY%\AU /f /v AUOptions /t REG_DWORD /d %AUOPT% 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY%\AU /f /v ScheduledInstallDay /t REG_DWORD /d %SCHEDULEDDAY% 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY%\AU /f /v ScheduledInstallTime /t REG_DWORD /d %SCHEDULEDTIME% 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

rem WSUSサーバの2種が登録されていればWSUS指定キーの登録(無い場合はGroup定義までスキップ)
set | findstr WU_SRV 2>nul 1>nul
if %ERRORLEVEL% NEQ 0 (goto GROUPSET)
set | findstr WU_STATSRV 2>nul 1>nul
if %ERRORLEVEL% NEQ 0 (goto GROUPSET)

reg add %RKEY%\AU /f /v UseWUServer /t REG_DWORD /d 1 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY% /f /v WUServer /t REG_SZ /d %WU_SRV% 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY% /f /v WUStatusServer /t REG_SZ /d %WU_STATSRV% 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

rem Goup定義
:GROUPSET
reg add %RKEY% /f /v TargetGroupEnabled /t REG_DWORD /d 0 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

rem WSUSのグループが指定されていればGroupキーの登録(無い場合はスキップ)
set | findstr TARGET_GRP_NAME 2>nul 1>nul
if %ERRORLEVEL% NEQ 0 (goto FIN)

reg add %RKEY% /f /v TargetGroupEnabled /t REG_DWORD /d 1 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

reg add %RKEY% /f /v TargetGroup /t REG_SZ /d %TARGET_GRP_NAME% 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

:FIN
rem ---------------------------------------------------------------------
rem #Windows10 専用レジストリ情報の登録処理
rem ---------------------------------------------------------------------
if %FORWIN10% EQU 0 (goto FIN_WIN10)

echo [INFO]設定を変更します。(Windows10 専用）
rem インターネット上のWindowsUpdateに接続しない
reg add %RKEY% /f /v DoNotConnectToWindowsUpdateInternetLocations /t REG_DWORD /d 1 1>nul 2>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

rem 自動更新の無効化
reg add %RKEY%\AU /f /v NoAutoUpdate /t REG_DWORD /d 1 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

rem デュアルスキャンの無効化
reg add %RKEY% /f /v DisableDualScan /t REG_DWORD /d 1 1>nul 2>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto FAIL_Setting)

:FIN_WIN10


rem ---------------------------------------------------------------------
rem #終了処理
rem ---------------------------------------------------------------------

rem del start グループポリシーは使用しないので削除 2010/5/10---------------
rem echo [INFO]ポリシーの即時反映を試みます。 >>C:\wsus.log
rem gpupdate /force 2>nul 1>nul >>C:\wsus.log
rem del end グループポリシーは使用しないので削除 2010/5/10-----------------

rem add start win8対応： 失敗時の切り分け用に追加 2013/10/7----------------
echo [INFO]自動更新サービスの状態を確認します>>C:\wsus.log
sc query wuauserv | findstr -i state 2>nul 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto WARN_AUService)
rem add end win8対応： 失敗時の切り分け用に追加 2013/10/7------------------

echo [INFO]自動更新サービスの再起動を実施します。>>C:\wsus.log
net stop wuauserv 2>nul 1>nul >>C:\wsus.log

rem del start： wuauservが起動していなければ、エラー無視して起動 2013/10/7----
rem if %ERRORLEVEL% NEQ 0 (goto WARN_AUService)
rem del end： wuauservが起動していなければ、エラー無視して起動 2013/10/7------

net start wuauserv 2>nul 1>nul >>C:\wsus.log
if %ERRORLEVEL% NEQ 0 (goto WARN_AUService)

echo [INFO]wsusサーバの即時検知を行います。 >>C:\wsus.log
wuauclt /resetauthorization /detectnow 2>nul 1>nul >>C:\wsus.log

echo [INFO]処理は正常に終了しました。>>C:\wsus.log
echo [INFO]処理は正常に終了しました。更新結果はログファイルをご確認ください。
echo wsusset > %WORKDIR%\%FLAGFILE%

echo.>>C:\wsus.log　
echo --------------------------------------------------------------------------- >>C:\wsus.log
echo 　■更新後レジストリ値 >>C:\wsus.log
echo --------------------------------------------------------------------------- >>C:\wsus.log
echo.>>C:\wsus.log　　

echo 自動更新オプション設定値：0x2 >>C:\wsus.log
reg query %RKEY%\AU /v AUOptions >>C:\wsus.log

echo ターゲットグループ設定値：%TARGET_GRP_NAME% >>C:\wsus.log
reg query %RKEY% /v TargetGroup >>C:\wsus.log

echo WSUSサーバ設定値：%WU_SRV%　>>C:\wsus.log
reg query %RKEY% /v WUServer >>C:\wsus.log

echo WSUSステータスサーバ設定値：%WU_STATSRV%>>C:\wsus.log
reg query %RKEY% /v WUStatusServer >>C:\wsus.log
echo --------------------------------------------------------------------------- >>C:\wsus.log

pause
exit /b 0

rem ------------------------------------------------------------------------------------------
rem #作業ディレクトリに移動できなかった(戻り値:1)
rem ------------------------------------------------------------------------------------------
:FAIL_Dir
echo [FAIL]作業ディレクトリに移動が出来ませんでした。 >>C:\wsus.log
pause
exit /b 1

rem ------------------------------------------------------------------------------------------
rem #値の設定が上手く行かなかった-バックアップファイルがあればロールバックして終了(戻り値:2)
rem ------------------------------------------------------------------------------------------
:FAIL_Setting
echo [FAIL]処理を中断します(レジストリ情報の書き込みに失敗) >>C:\wsus.log
echo [FAIL]ロールバックして終了します。 >>C:\wsus.log

if %BKFLAG% EQU 0 (
reg delete %RKEY% /f /v SusClientId 2>nul 1>nul
reg delete %RKEY%\AU /f /v NoAutoRebootWithLoggedOnUsers 2>nul 1>nul
reg delete %RKEY%\AU /f /v NoAutoUpdate 2>nul 1>nul
reg delete %RKEY%\AU /f /v AUOptions 2>nul 1>nul
reg delete %RKEY%\AU /f /v ScheduledInstallDay 2>nul 1>nul
reg delete %RKEY%\AU /f /v ScheduledInstallTime 2>nul 1>nul
reg delete %RKEY%\AU /f /v UseWUServer 2>nul 1>nul
reg delete %RKEY% /f /v WUServer 2>nul 1>nul
reg delete %RKEY% /f /v WUStatusServer 2>nul 1>nul
reg delete %RKEY% /f /v TargetGroupEnabled 2>nul 1>nul
reg delete %RKEY% /f /v TargetGroupEnabled 2>nul 1>nul
reg delete %RKEY% /f /v TargetGroup 2>nul 1>nul

rem WINDOWS10の場合のみもう1つ削除
if %FORWIN10% EQU 1 (
reg delete %RKEY% /f /v DoNotConnectToWindowsUpdateInternetLocations 2>nul 1>nul
reg delete %RKEY% /f /v DisableDualScan 2>nul 1>nul
)


reg import %BKFILE%
if %ERRORLEVEL% NEQ 0 (goto FAIL_Crit)
)

if %BKFLAG2% EQU 0 (
reg delete %RKEY2% /f /v SusClientId 2>nul 1>nul
reg import %BKFILE2%
if %ERRORLEVEL% NEQ 0 (goto FAIL_Crit)
)

echo [INFO]ロールバックが終了しました。 >>C:\wsus.log
pause
exit /b 2

rem ---------------------------------------------------------------------
rem #ロールバックが失敗-終了(戻り値:3)
rem ---------------------------------------------------------------------
:FAIL_Crit
echo [FAIL]ロールバックが失敗しました。手動でロールバックを行ってください。 >>C:\wsus.log
pause
exit /b 3

rem ---------------------------------------------------------------------
rem #WindowsUpdateサービスの再起動が失敗-終了(戻り値:5)
rem ---------------------------------------------------------------------
:WARN_AUService
echo [WARN]自動更新サービスの再起動処理に失敗しました。手動でサービスの再起動を行ってください。 >>C:\wsus.log
pause
exit /b 5

rem 2010/06/28 add start
rem ---------------------------------------------------------------------
rem #レジストリのバックアップ失敗-終了(戻り値:6)
rem ---------------------------------------------------------------------
:FAIL_Backup
echo [FAIL]レジストリのバックアップに失敗しました。処理を中断します。 >>C:\wsus.log
pause
exit /b 6
rem 2010/06/28 add end
