# ハンズオン: AWS Elemental MediaConvert トランスコード & イベント駆動ワークフロー

## 概要

本ハンズオンでは、AWS Elemental MediaConvert を使用して動画ファイルをトランスコードします。  
前半では手動でジョブを作成して動作を確認し、後半では S3 にファイルをアップロードするだけで自動的にトランスコードが実行されるイベント駆動ワークフローを構築します。

### 前提条件

- AWS アカウントにサインイン済みであること
- 使用するリージョンとして東京（ap-northeast-1）を選択済みであること

### 到達目標

- S3 バケットの作成とファイルアップロードができる
- MediaConvert でトランスコードジョブを作成・実行できる
- 入力と出力のファイルを比較し、トランスコードの効果を確認できる
- Lambda と S3 イベント通知を使い、自動トランスコードの仕組みを構築できる

### サンプル動画ファイル

本ハンズオンでは以下のファイルを使用します。事前にダウンロードしておいてください。

- **Big Buck Bunny（10 秒・1080p・約 30MB）**
  - URL: `https://test-videos.co.uk/vids/bigbuckbunny/mp4/h264/1080/Big_Buck_Bunny_1080_10s_30MB.mp4`
  - ライセンス: Creative Commons Attribution 3.0（Blender Foundation）

---

## ステップ 1: S3 バケットの作成

入力ファイルと出力ファイルの保存先として S3 バケットを作成します。

### 手順

1. ページ上部の **検索** に `s3` と入力して、**S3** のメニューを選択します。

1. ページ左側のナビゲーションメニューで **汎用バケット** をクリックします。

1. **バケットを作成** をクリックします。

1. 入力用バケットを作成します。
   - **バケット名**: `mediaconvert-handson-input-<自分のID>`
   - その他の設定はデフォルトのまま
   - **バケットを作成** をクリックします。

1. ページ左上の三本線アイコンをクリックし、ナビゲーションメニューで **汎用バケット** をクリックします。

1. 同様に出力用バケットを作成します。
   - **バケットを作成** をクリックします。
   - **バケット名**: `mediaconvert-handson-output-<自分のID>`
   - その他の設定はデフォルトのまま
   - **バケットを作成** をクリックします。

---

## ステップ 2: サンプル動画ファイルのアップロード

### 手順

1. S3 コンソールの **汎用バケット** 一覧で、入力用バケット（`mediaconvert-handson-input-<自分のID>`）をクリックします。

1. **アップロード** をクリックします。

1. **ファイルを追加** をクリックし、ダウンロード済みのサンプル動画ファイルを選択します。

1. **アップロード** をクリックします。

1. アップロードが完了したら、**閉じる** をクリックします。

1. アップロードしたファイル名のリンクをクリックし、**S3 URI** の値をメモしておきます。
   - 例: `s3://mediaconvert-handson-input-<自分のID>/Big_Buck_Bunny_1080_10s_30MB.mp4`

---

## ステップ 3: トランスコードジョブの作成と実行

MediaConvert でジョブを作成します。設定はできるだけシンプルに、最小限の変更で実行します。

### 手順

1. ページ上部の **検索** に `mediaconvert` と入力して、**MediaConvert** のメニューを選択します。

1. 左側のナビゲーションメニューで **ジョブ** をクリックします。
   - ナビゲーションメニューが表示されていない場合は、左上の三本線アイコンをクリックします。

1. **ジョブの作成** をクリックします。

### 3-1. IAM ロールの設定

1. 左ペインの **ジョブの設定** から **AWS の統合** をクリックします。

1. **サービスアクセス** セクションで以下を設定します:
   - **サービスロールの制御**: 「新しいサービスロールを作成し、アクセス許可を設定」を選択します。
   - **新しいロール名**: デフォルトの `MediaConvert_Default_Role` のままにします。
   - **S3 の 入力場所**: **ロケーションの追加** をクリックし、**参照** から入力用バケットを選択します。
   - **S3 出力場所**: **ロケーションの追加** をクリックし、**参照** から出力用バケットを選択します。

> **補足:** 既に `MediaConvert_Default_Role` が存在する場合は「既存のサービスロールを使用する」を選択してください。

### 3-2. 入力ファイルの指定

1. 左ペインの **入力** セクションで **入力 1** をクリックします。

1. **入力ファイル URL** にステップ 2 でメモした S3 URI を入力します。
   - **参照** ボタンからバケット内のファイルを選択することもできます。

### 3-3. 出力グループと出力の設定

1. 左ペインの **出力グループ** セクションで **追加** をクリックします。

1. **ファイルグループ** を選択し、**選択** をクリックします。

1. **送信先** に出力用バケットのパスを入力します。
   - 例: `s3://mediaconvert-handson-output-<自分のID>/`
   - **参照** ボタンから選択することもできます。末尾のスラッシュを忘れないようにしてください。

1. 左ペインで **出力グループ** > **File Group** の下に表示されている出力をクリックします。

1. **出力設定** セクションで **名前修飾子** に `-720p` と入力します。

1. **エンコード設定** セクションの **動画** タブで、以下の 3 項目を設定します:
   - **幅 - オプション**: `1280`
   - **高さ - オプション**: `720`
   - **最大ビットレート (ビット/秒)**: `3000000`

   その他の設定（コーデック、レート制御モードなど）はすべてデフォルトのままにします。

> **ポイント:** 解像度の 2 項目で 1080p → 720p へのダウンスケールを指定し、最大ビットレートで出力ファイルサイズの上限を制御します。レート制御モードがデフォルトの QVBR（品質ベース可変ビットレート）の場合、最大ビットレートの指定が必須です。

### 3-4. ジョブの実行

1. 画面右下の **作成** ボタンをクリックします。

1. ジョブが作成されます。

> The service does not have permission to assume the IAM role: というエラーが表示された場合は、もう一度 **作成** ボタンをクリックしてみて下さい。

---

## ステップ 4: 結果の確認

### 4-1. ジョブステータスの確認

1. 左側のナビゲーションメニューで **ジョブ** をクリックします。

1. ジョブ一覧画面で、作成したジョブのステータスを確認します。
   - **PROGRESSING** → **COMPLETE** になるまで待ちます。
   - 10 秒の動画であれば 1 分以内に完了します。

1. ステータスが **COMPLETE** にならない場合は、ジョブ ID をクリックしてエラーメッセージを確認してください。

### 4-2. 出力ファイルの確認

1. ページ上部の **検索** に `s3` と入力して、S3 コンソールに移動します。

1. 出力用バケット（`mediaconvert-handson-output-<自分のID>`）を開きます。

1. 出力ファイル（例: `Big_Buck_Bunny_1080_10s_30MB-720p.mp4`）が生成されていることを確認します。

1. ファイル名をクリックし、**ダウンロード** ボタンからダウンロードします。

### 4-3. トランスコードの効果を確認

入力ファイルと出力ファイルを比較して、トランスコードの効果を確認します。

| 確認項目 | 入力ファイル | 出力ファイル |
|---------|------------|------------|
| 解像度 | 1920 x 1080 | 1280 x 720 |
| ファイルサイズ | 約 30MB | これより小さくなっているはず |

ローカルに `ffprobe` または `MediaInfo` がインストールされている場合は、以下のコマンドで詳細を確認できます:

```bash
# 入力ファイルの情報
ffprobe -v quiet -print_format json -show_streams Big_Buck_Bunny_1080_10s_30MB.mp4

# 出力ファイルの情報
ffprobe -v quiet -print_format json -show_streams Big_Buck_Bunny_1080_10s_30MB-720p.mp4
```

確認ポイント:
- 映像の解像度が 1280x720 に変わっていること
- コーデックが H.264 であること
- ファイルサイズが入力より小さくなっていること

---

## ステップ 5: バケット内ファイルの削除

後半のイベント駆動ワークフローに備えて、入力用バケットと出力用バケットの中身を空にします。

### 手順

1. ページ上部の **検索** に `s3` と入力して、**S3** のメニューを選択します。

1. 入力用バケット（`mediaconvert-handson-input-<自分のID>`）のラジオボタンを選択します。

1. **空にする** をクリックします。
   - 確認画面で「完全に削除」と入力し、**空にする** をクリックします。

1. バケット一覧に戻り、出力用バケット（`mediaconvert-handson-output-<自分のID>`）のラジオボタンを選択します。

1. **空にする** をクリックします。
   - 確認画面で「完全に削除」と入力し、**空にする** をクリックします。

---

## ステップ 6: Lambda 関数の作成（イベント駆動ワークフロー）

ここからは、S3 に動画ファイルをアップロードするだけで自動的にトランスコードが実行される仕組みを構築します。  
S3 のイベント通知 → Lambda 関数 → MediaConvert ジョブ という流れです。

### 手順

1. ページ上部の **検索** に `lambda` と入力して、**Lambda** のメニューを選択します。

1. **関数の作成** をクリックします。

1. 以下の設定で関数を作成します:
   - **関数名**: `mediaconvert-auto-transcode`
   - **ランタイム**: Python 3.14
   - その他はデフォルトのまま

1. ページ右下の **関数を作成** をクリックします。

1. **Getting started** が表示された場合は、**Dismiss** をクリックします。

### 6-1. Lambda 関数のコードを入力

関数が作成されたら、コードエディタが表示されます。

1. `lambda_function.py` の内容をすべて削除し、以下のコードを貼り付けます:

```python
import json
import os
import boto3

def lambda_handler(event, context):
    """S3 イベントを受け取り、MediaConvert ジョブを作成する"""

    # S3 イベントからファイル情報を取得
    record = event['Records'][0]
    source_bucket = record['s3']['bucket']['name']
    source_key = record['s3']['object']['key']

    # 環境変数から設定を取得
    destination_bucket = os.environ['DESTINATION_BUCKET']
    media_convert_role = os.environ['MEDIA_CONVERT_ROLE']

    # MediaConvert クライアントを作成
    mediaconvert = boto3.client('mediaconvert')

    # エンドポイントを取得
    endpoints = mediaconvert.describe_endpoints()
    endpoint_url = endpoints['Endpoints'][0]['Url']

    # エンドポイント指定でクライアントを再作成
    mediaconvert = boto3.client('mediaconvert', endpoint_url=endpoint_url)

    # 入力ファイルのパス
    input_file = f's3://{source_bucket}/{source_key}'

    # 出力先のパス
    output_destination = f's3://{destination_bucket}/'

    # ジョブ設定
    job_settings = {
        "Inputs": [
            {
                "FileInput": input_file,
                "AudioSelectors": {
                    "Audio Selector 1": {
                        "DefaultSelection": "DEFAULT"
                    }
                }
            }
        ],
        "OutputGroups": [
            {
                "Name": "File Group",
                "OutputGroupSettings": {
                    "Type": "FILE_GROUP_SETTINGS",
                    "FileGroupSettings": {
                        "Destination": output_destination
                    }
                },
                "Outputs": [
                    {
                        "NameModifier": "-720p",
                        "VideoDescription": {
                            "Width": 1280,
                            "Height": 720,
                            "CodecSettings": {
                                "Codec": "H_264",
                                "H264Settings": {
                                    "RateControlMode": "QVBR",
                                    "MaxBitrate": 3000000
                                }
                            }
                        },
                        "AudioDescriptions": [
                            {
                                "CodecSettings": {
                                    "Codec": "AAC",
                                    "AacSettings": {
                                        "Bitrate": 96000,
                                        "CodingMode": "CODING_MODE_2_0",
                                        "SampleRate": 48000
                                    }
                                }
                            }
                        ],
                        "ContainerSettings": {
                            "Container": "MP4"
                        }
                    }
                ]
            }
        ]
    }

    # ジョブを作成
    response = mediaconvert.create_job(
        Role=media_convert_role,
        Settings=job_settings
    )

    job_id = response['Job']['Id']
    print(f'MediaConvert ジョブを作成しました: {job_id}')

    return {
        'statusCode': 200,
        'body': json.dumps({'jobId': job_id})
    }
```

1. **Deploy** をクリックしてコードを保存します。

### 6-2. 環境変数の設定

1. **設定** タブをクリックします。

1. 左側メニューの **環境変数** をクリックします。

1. **編集** をクリックします。

1. **環境変数の追加** をクリックして、以下の 2 つを追加します:

   | キー | 値 |
   |------|---|
   | `DESTINATION_BUCKET` | `mediaconvert-handson-output-<自分のID>` |
   | `MEDIA_CONVERT_ROLE` | `arn:aws:iam::<AWSアカウントID>:role/service-role/MediaConvert_Default_Role` |
   > **補足:** AWS アカウント ID は、ページ右上のアカウント名をクリックすると確認できます。

1. **保存** をクリックします。

### 6-3. Lambda 実行ロールに権限を追加

Lambda 関数が MediaConvert と S3 にアクセスできるよう、実行ロールに権限を追加します。

1. **設定** タブ > 左側メニューの **アクセス権限** をクリックします。

1. **ロール名** のリンクをクリックします。IAM コンソールが新しいタブで開きます。

1. **許可を追加** > **ポリシーをアタッチ** をクリックします。

1. 検索ボックスに `AmazonS3ReadOnlyAccess` と入力し、チェックボックスを選択します。

1. 検索ボックスをクリアし、`AWSElementalMediaConvertFullAccess` と入力し、チェックボックスを選択します。

1. **許可を追加** をクリックします。

1. IAM コンソールのタブを閉じ、Lambda のタブに戻ります。

### 6-4. タイムアウトの変更

1. **設定** タブ > 左側メニューの **一般設定** をクリックします。

1. **編集** をクリックします。

1. **タイムアウト** を `0` 分 `30` 秒 に変更します。

1. **保存** をクリックします。

---

## ステップ 7: S3 イベント通知の設定

入力用バケットにファイルがアップロードされたとき、Lambda 関数が呼び出されるように設定します。

### 手順

1. ページ上部の **検索** に `s3` と入力して、**S3** のメニューを選択します。

1. 入力用バケット（`mediaconvert-handson-input-<自分のID>`）をクリックします。

1. **プロパティ** タブをクリックします。

1. ページ下部の **イベント通知** セクションまでスクロールし、**イベント通知を作成** をクリックします。

1. 以下を設定します:
   - **イベント名**: `auto-transcode-trigger`
   - **サフィックス**: `.mp4`（MP4 ファイルのみ対象にします）
   - **イベントタイプ**: **すべてのオブジェクト作成イベント** にチェックを入れます
   - **送信先**: **Lambda 関数** を選択します
   - **Lambda 関数**: `mediaconvert-auto-transcode` を選択します

1. **変更の保存** をクリックします。

---

## ステップ 8: 自動トランスコードの動作確認

設定が完了しました。サンプル動画ファイルをアップロードして自動変換を確認します。

### 手順

1. S3 コンソールで入力用バケット（`mediaconvert-handson-input-<自分のID>`）の **オブジェクト** タブを開きます。

1. **アップロード** をクリックします。

1. サンプル動画ファイル（`Big_Buck_Bunny_1080_10s_30MB.mp4`）を選択し、**アップロード** をクリックします。

1. アップロードが完了したら、**閉じる** をクリックします。

### 8-1. MediaConvert ジョブの確認

1. ページ上部の **検索** に `mediaconvert` と入力して、**MediaConvert** のメニューを選択します。

1. 左側のナビゲーションメニューで **ジョブ** をクリックします。

1. 新しいジョブが自動的に作成されていることを確認します。
   - ステータスが **PROGRESSING** → **COMPLETE** になるまで待ちます。

> **ポイント:** 手動でジョブを作成していないのに、ファイルをアップロードしただけでジョブが実行されています。これがイベント駆動ワークフローの効果です。

### 8-2. 出力ファイルの確認

1. S3 コンソールで出力用バケット（`mediaconvert-handson-output-<自分のID>`）を開きます。

1. 出力ファイル（例: `Big_Buck_Bunny_1080_10s_30MB-720p.mp4`）が生成されていることを確認します。

1. ファイルサイズが入力（約 30MB）より小さくなっていることを確認します。

---

## トラブルシューティング

| 症状 | 原因と対処 |
|------|-----------|
| ジョブが ERROR になる | IAM ロールの権限不足の可能性があります。ステップ 3-1 で入力・出力バケットを正しく指定したか確認してください |
| 入力ファイルが見つからない | S3 URI のパスが正しいか確認してください。リージョンが一致しているかも確認してください |
| 出力ファイルが生成されない | 送信先の S3 パスが正しいか確認してください。末尾のスラッシュが必要です |
| ファイルをアップロードしてもジョブが作成されない | Lambda 関数の実行ロールに権限が不足している可能性があります。ステップ 6-3 を確認してください |
| Lambda がエラーになる | Lambda の **モニタリング** タブ > **CloudWatch Logs を表示** からログを確認してください |
| `MEDIA_CONVERT_ROLE` のエラー | 環境変数の ARN が正しいか確認してください。アカウント ID が 12 桁の数字であることを確認してください |

---

## 参考リンク

- [AWS Elemental MediaConvert ユーザーガイド - Getting Started](https://docs.aws.amazon.com/mediaconvert/latest/ug/getting-started.html)
- [MediaConvert ジョブ設定チュートリアル](https://docs.aws.amazon.com/mediaconvert/latest/ug/setting-up-a-job.html)
- [IAM ロールの設定](https://docs.aws.amazon.com/mediaconvert/latest/ug/iam-role.html)
- [AWS Lambda デベロッパーガイド](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Amazon S3 イベント通知](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html)

---

## クリーンアップ（インストラクターが実施）

> **注意:** この手順はインストラクターが実施します。受講者は実行しないでください。

ハンズオン終了後、AWS CloudShell からクリーンアップスクリプトを実行し、作成したリソースをすべて削除します。

### 手順

1. AWS マネジメントコンソールのページ下部にある **CloudShell** アイコンをクリックします。

1. CloudShell が起動したら、以下のコマンドを実行します:

```bash
curl -s https://tnobep-work-public.s3.ap-northeast-1.amazonaws.com/mediaconvert-work/cleanup.sh | bash
```

1. すべてのリソース（S3 バケット、Lambda 関数、CloudWatch Logs、IAM ロール）が削除されます。

