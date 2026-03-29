# ADサーバー Windows Update 効率化計画

## 1. 背景
直近のADサーバー更新作業において、約3年分のパッチ滞留に起因する適用時間の長期化（8時間以上）が発生した。次回の本番作業を営業時間内に完結させ、かつ保守コストを最適化するため、以下のクリーンアップ手順および体制変更を提案・検討する。

---

## 2. 外部ベンダーへの確認依頼内容

### 件名：ADサーバー更新作業の効率化に向けた「事前クリーンアップ」の実施および体制変更の検討について

**様  
いつも大変お世話になっております、**です。

先日のADサーバー更新作業において、3年分のパッチ滞留により適用時間が8時間を超えた件を受け、次回の本番作業を確実に営業時間内に収めるための効率化案を検討しております。

つきましては、私の方で以下の「事前クリーンアップ」の手順およびスクリプト案を作成いたしました。インフラ保守・運用の専門的な観点から、**本手順の実行による妥当性や潜在的なリスク**について、貴社の見解を伺いたく存じます。

#### ① 事前クリーンアップの技術的確認
更新作業の長期化は、蓄積された「superseded updates（古い更新の残骸）」の処理にリソースが割かれていることが主因と推察しております。これらを整理するため、以下の処理を先行実施したいと考えております。

* **実施内容：**
    1.  Windows Update関連サービスの停止
    2.  `SoftwareDistribution` および `Catroot2` のフォルダリセット（キャッシュクリア）
    3.  `DISM /StartComponentCleanup` によるコンポーネントストアの最適化
* **実行タイミング：**
    現在は「再起動待ち」の状態であるため、OSの整合性を優先し、**一度OSを再起動させた直後**に本スクリプトを実行。累積した残骸を排除した状態で後続のパッチ適用を加速させる計画です。

上記手順において、ADサーバー特有の挙動や、貴社保守範囲における懸念事項がございましたらご教示ください。

#### ② 作業体制および内製化によるコスト最適化の検討
現在、AD1号機・2号機ともに貴社およびB社（アプリ保守）の立ち会いを前提としておりますが、稼働工数削減のため以下の切り分けを検討しております。

* **AD1号機：** 弊社（久松）にてOS再起動および上記クリーンアップ対応を内製化する。
* **AD2号機：** 貴社およびB社を含めた従来通りの体制で実施する。

1号機の対応を弊社で完結させることで、全体の拘束時間および外注コストの適正化を図りたいと考えております。こちらの体制案についても、併せてご意見をいただけますでしょうか。

---

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

PS C:\WINDOWS\system32> cd $HOME\OneDrive\デスクトップ
PS C:\Users\austr\OneDrive\デスクトップ>
powershell -ExecutionPolicy Bypass -File .\test_PreCleanup.ps1 

<試験用>
**【スクリプト内容】**
```powershell
# ================================================
# ADサーバー 事前クリーンアップスクリプト（DryRun対応版）
# Windows Update適用前の準備（SoftwareDistribution削除 + DISMクリーンアップ）
# ================================================

param(
    [switch]$DryRun   # ← これを付けて実行するとチェックのみ（実際の変更なし）
)

$TranscriptPath = "C:\Temp\PreCleanup_Transcript_$(Get-Date -Format 'yyyyMMdd_HHmmss').txt"

Start-Transcript -Path $TranscriptPath -Append
Write-Host "=== Windows Update 事前クリーンアップ開始 ===" -ForegroundColor Green

if ($DryRun) {
    Write-Host "【DRY RUN MODE】チェックのみ実行します。実際の変更は行いません。" -ForegroundColor Cyan
}

# 0. 現在の状態を確認（常に実行）
Write-Host "0. 現在のコンポーネントストア状況を確認しています..." -ForegroundColor Yellow
Dism.exe /Online /Cleanup-Image /AnalyzeComponentStore

# 1. サービス停止
Write-Host "1. Windows Update関連サービスを停止しています..." -ForegroundColor Yellow
if (-not $DryRun) {
    Stop-Service -Name wuauserv, bits, cryptSvc, msiserver -Force -ErrorAction SilentlyContinue
} else {
    Write-Host "   → DryRunのため実際には停止しません" -ForegroundColor Gray
}

# 2. 更新キャッシュ削除
Write-Host "2. SoftwareDistribution と Catroot2 をクリアしています..." -ForegroundColor Yellow
if (-not $DryRun) {
    Remove-Item -Path "C:\Windows\SoftwareDistribution\*" -Recurse -Force -ErrorAction SilentlyContinue
    Remove-Item -Path "C:\Windows\System32\catroot2\*" -Recurse -Force -ErrorAction SilentlyContinue
} else {
    Write-Host "   → DryRunのため実際には削除しません" -ForegroundColor Gray
}

# 3. サービス再起動
Write-Host "3. サービスを再起動しています..." -ForegroundColor Yellow
if (-not $DryRun) {
    Start-Service -Name wuauserv, bits, cryptSvc, msiserver -ErrorAction SilentlyContinue
} else {
    Write-Host "   → DryRunのため実際には再起動しません" -ForegroundColor Gray
}

# 4. DISMクリーンアップ（これが一番重い）
Write-Host "4. DISMクリーンアップを実行しています..." -ForegroundColor Yellow
if (-not $DryRun) {
    Dism.exe /Online /Cleanup-Image /StartComponentCleanup
} else {
    Write-Host "   → DryRunのため実際にはクリーンアップを実行しません" -ForegroundColor Gray
}

# 完了確認
Write-Host "5. クリーンアップ結果を確認しています..." -ForegroundColor Yellow
Dism.exe /Online /Cleanup-Image /AnalyzeComponentStore

Write-Host "`n=== クリーンアップ完了 ===" -ForegroundColor Green
if ($DryRun) {
    Write-Host "【DryRun完了】実際の変更は一切行われていません。" -ForegroundColor Cyan
}
Stop-Transcript

Write-Host "詳細ログ: $TranscriptPath" -ForegroundColor Cyan
```
以上、よろしくお願いいたします。


