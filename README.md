# ADサーバUpdate事前作業相談

### メール本文

**件名：ADサーバーWindows Update適用に関する事前作業の提案について**

**様

先日もお話ししましたが、現在ADサーバーに約3年分のWindows Updateが溜まっている影響で、前回の適用作業には8時間以上を要し、急遽2日間を確保するという話を伺っております。
このインストール時間が極端に長引く主な原因として、蓄積された「superseded updates（古い更新の残骸）」の処理にリソースが割かれている可能性が高いのではないかと推測致しました。
次回の更新作業を確実に営業時間内（8時間以内）に収めるため、本番作業の前に、私の方であらかじめ以下の「事前クリーンアップ」を実施しておきたいと考えております。
なので、私の方でスクリプトを作成致しました。
つきましては、インフラ運用上の観点から、本手順の実行による妥当性についてご確認をいただけますでしょうか。
※スクリプトの動作自体はクライアントPC上で試験済みです。

ただし、すでにインストールが完了して再起動を待っている状態では、
OSは「次の起動時にこのファイルを入れ替える」という予約リスト（ペンディング操作）を作成しています。
その最中にキャッシュを削除したり、システムコンポーネントを整理（DISM）したりすると、不整合が起きてOSが起動しなくなったり、長時間のロールバックが発生したりするリスク

なので、今回のセキュリティパッチ適用については再起動待ちの状態ではOSの整合性を優先し、一度OS再起動という形で適用を完了させる。
その直後にクリーンアップを実施することで、累積している残骸を排除し、後続のパッチ適用を加速させたいと考えておりますが、妥当性についてご教示いただきますようお願い致します。

また、AD1号機と2号機について別システムへのファイルサーバーも担っているため、別システムのアプリ確認が必要となり、B社の対応が必要ですが
例えば1号機だけは完全に弊社だけでOS再起動対応を内製化し、2号機の対応のみをA社とB社を巻き込んだ形で依頼できれば稼働時間を削減できるため、予算も削減につながるのではないかと推察しております。

**【実施内容】**
1.  Windows Update関連サービスの停止
2.  `SoftwareDistribution` および `Catroot2` のフォルダクリア（更新キャッシュのリセット）
3.  `DISM /StartComponentCleanup` の実行（コンポーネントストアの最適化）

PS C:\WINDOWS\system32> cd $HOME\OneDrive\デスクトップ
PS C:\Users\austr\OneDrive\デスクトップ>
powershell -ExecutionPolicy Bypass -File .\PreCleanup.ps1

<本番用>
**【スクリプト内容】**
```powershell
# ================================================
# ADサーバー 事前クリーンアップスクリプト
# Windows Update適用前の準備（SoftwareDistribution削除 + DISMクリーンアップ）
# ================================================

$LogPath = "C:\Temp\PreCleanup_Log_$(Get-Date -Format 'yyyyMMdd_HHmmss').txt"
$TranscriptPath = "C:\Temp\PreCleanup_Transcript_$(Get-Date -Format 'yyyyMMdd_HHmmss').txt"

# ログ開始
Start-Transcript -Path $TranscriptPath -Append
Write-Host "=== Windows Update 事前クリーンアップ開始 ===" -ForegroundColor Green

# 1. サービス停止
Write-Host "1. Windows Update関連サービスを停止しています..." -ForegroundColor Yellow
Stop-Service -Name wuauserv, bits, cryptSvc, msiserver -Force -ErrorAction SilentlyContinue

# 2. 更新キャッシュ削除
Write-Host "2. SoftwareDistribution と Catroot2 をクリアしています..." -ForegroundColor Yellow
Remove-Item -Path "C:\Windows\SoftwareDistribution\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Windows\System32\catroot2\*" -Recurse -Force -ErrorAction SilentlyContinue

# 3. サービス再起動
Write-Host "3. サービスを再起動しています..." -ForegroundColor Yellow
Start-Service -Name wuauserv, bits, cryptSvc, msiserver -ErrorAction SilentlyContinue

# 4. DISMクリーンアップ（superseded updatesの整理）
Write-Host "4. DISMクリーンアップを実行しています..." -ForegroundColor Yellow
Dism.exe /Online /Cleanup-Image /StartComponentCleanup

# 完了確認
Write-Host "5. クリーンアップ結果を確認しています..." -ForegroundColor Yellow
Dism.exe /Online /Cleanup-Image /AnalyzeComponentStore

Write-Host "`n=== クリーンアップ完了 ===" -ForegroundColor Green
Stop-Transcript
```
以上、よろしくお願いいたします。


