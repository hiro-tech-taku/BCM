前提
・NVIDIA 提供の BCM head node AMI から EC2 を起動する

・ライセンスは すでに発行済みの cluster.pem（署名済みライセンスファイル） を使って設定する
ライセンス証書の配置先 /cm/local/apps/cmd/etc/：第 4章冒頭（頁 48あたり）

起動後の流れ
・マニュアルでも request-license で取得
 → install-license <signedlicense> でインストールする流れになっていますNVIDIA Docs+1
request-license／install-license の記載：同じく 第 4章 4.3節（頁 51〜57あたり）

・verify-license コマンドでライセンスが有効かどうか判定できるので、
それを使って「すでに有効ならスキップ」という idempotent な動きにします
verify-license の記載：Installation Manual v10 第 4章 4.2節（頁 49〜51あたり）

UserData（cloud-init）YAML 例
※ ここでは簡単のために「ライセンスファイル cluster.pem をあらかじめ AMI 内か、別の仕組み（S3 / EFS / 手動コピー等）で /cm/local/apps/cmd/etc/cluster.pem に置く」前提にしています。
必要ならここに aws s3 cp を足しても OK です。
#cloud-config

# 必要なら OS アップデートなど
package_update: true
package_upgrade: false

write_files:
  - path: /usr/local/sbin/bcm-license-setup.sh
    permissions: '0755'
    owner: root:root
    content: |
      #!/bin/bash
      set -euo pipefail

      LOG_FILE="/var/log/bcm-license-setup.log"
      exec >>"${LOG_FILE}" 2>&1

      echo "[INFO] $(date) BCM license setup start"

      # 1. すでにライセンス有効なら何もしないで終了
      if command -v verify-license >/dev/null 2>&1; then
        if verify-license >/dev/null 2>&1; then
          echo "[INFO] License already valid. Skipping install."
          exit 0
        else
          echo "[INFO] verify-license reports no valid license. Installing..."
        fi
      else
        echo "[WARN] verify-license command not found. Continuing assuming no license."
      fi

      # 2. ライセンスファイルの場所
      LICENSE_FILE="/cm/local/apps/cmd/etc/cluster.pem"

      if [ ! -f "${LICENSE_FILE}" ]; then
        echo "[ERROR] License file ${LICENSE_FILE} not found."
        echo "[ERROR] Place your signed license file here before rerunning this script."
        exit 1
      fi

      # 3. install-license でライセンス適用
      if ! command -v install-license >/dev/null 2>&1; then
        echo "[ERROR] install-license command not found. Is BCM properly installed?"
        exit 1
      fi

      echo "[INFO] Installing license from ${LICENSE_FILE}"
      install-license "${LICENSE_FILE}"

      # 4. 適用後に verify-license で再確認
      if verify-license >/dev/null 2>&1; then
        echo "[INFO] License verification successful."
      else
        echo "[ERROR] License verification failed after install."
        exit 1
      fi

      echo "[INFO] $(date) BCM license setup finished successfully."

runcmd:
  # 起動時に一度だけライセンス設定スクリプトを実行
  - [ bash, -c, "/usr/local/sbin/bcm-license-setup.sh" ]


処理のポイント（要件との対応）
1. ライセンス情報を設定する
- ライセンス格納場所
LICENSE_FILE="/cm/local/apps/cmd/etc/cluster.pem"

- ライセンスファイルをヘッドノード(BCM)にインストール
install-license "${LICENSE_FILE}" 

- インストール後にライセンスが適用されているか確認
verify-license

2. 再起動時にライセンス設定をスキップ
- スクリプト冒頭で verify-license を叩き、すでに有効なライセンスがあれば即 exit 0 します。

- ライセンス情報は /cm/local/apps/cmd/etc/cluster.pem と内部状態に保存されるので、
EC2 を再起動しても有効のままです（ディスクを消さない限り保持）

そのため、**UserData が再実行されたとしても「verify-license が OK ⇒ install-license を実行しない」**という動きになります。


*LICENSE_FILE を固定パスではなく
BCM_LICENSE_S3_URI のような環境変数を cloud-init の bootcmd / runcmd に追加して
aws s3 cp "$BCM_LICENSE_S3_URI "$LICENSE_FILE"
する形にもできます。

疑問
この YAML をベースに、cluster.pem をどこから持ってくるか（AMI 内、S3、EFS…）