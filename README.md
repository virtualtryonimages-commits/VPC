# VPC ラボ短縮手順

## 方針

前半の default ネットワーク確認・削除は講義中に口頭で説明し、受講者は default ネットワークを削除せず、VM 作成から開始する。

## 1\. VM 作成まではラボ手順どおり実施

タスク 2 の VM 作成部分から実施する。

* VPC ネットワーク: 既存の `default` を使用  
* VM 1: `mynet-us-vm`  
* VM 2: `mynet-r2-vm`

VM 作成後、Cloud Shell を開く。

## 2\. ゾーンと IP を自動取得する

Cloud Shell で以下を実行する。

```shell
ZONE1=$(gcloud compute instances list \
  --filter="name=mynet-us-vm" \
  --format="value(zone)")

ZONE2=$(gcloud compute instances list \
  --filter="name=mynet-r2-vm" \
  --format="value(zone)")

R2_INTERNAL_IP=$(gcloud compute instances describe mynet-r2-vm \
  --zone="$ZONE2" \
  --format="get(networkInterfaces[0].networkIP)")

R2_EXTERNAL_IP=$(gcloud compute instances describe mynet-r2-vm \
  --zone="$ZONE2" \
  --format="get(networkInterfaces[0].accessConfigs[0].natIP)")

echo "ZONE1=$ZONE1"
echo "ZONE2=$ZONE2"
echo "R2_INTERNAL_IP=$R2_INTERNAL_IP"
echo "R2_EXTERNAL_IP=$R2_EXTERNAL_IP"
```

## 3\. 初期状態の接続確認

```shell
gcloud compute ssh mynet-us-vm \
  --zone="$ZONE1" \
  --command="ping -c 3 $R2_INTERNAL_IP"
```

```shell
gcloud compute ssh mynet-us-vm \
  --zone="$ZONE1" \
  --command="ping -c 3 $R2_EXTERNAL_IP"
```

確認ポイント:

* 内部 IP への ping は通る  
* 外部 IP への ping も通る

## 4\. allow-icmp を削除して確認

```shell
gcloud compute firewall-rules delete default-allow-icmp --quiet
```

```shell
gcloud compute ssh mynet-us-vm \
  --zone="$ZONE1" \
  --command="ping -c 3 $R2_INTERNAL_IP"
```

```shell
gcloud compute ssh mynet-us-vm \
  --zone="$ZONE1" \
  --command="ping -c 3 $R2_EXTERNAL_IP"
```

確認ポイント:

* 内部 IP への ping はまだ通る  
* 外部 IP への ping は通らなくなる  
* 外部 IP 宛ての ICMP は `default-allow-icmp` に依存していたことがわかる

## 5\. allow-internal を削除して確認

```shell
gcloud compute firewall-rules delete default-allow-internal --quiet
```

```shell
gcloud compute ssh mynet-us-vm \
  --zone="$ZONE1" \
  --command="ping -c 3 $R2_INTERNAL_IP"
```

確認ポイント:

* 内部 IP への ping も通らなくなる  
* VPC 内部通信は `default-allow-internal` に依存していたことがわかる

## 6\. allow-ssh を削除して確認

```shell
gcloud compute firewall-rules delete default-allow-ssh --quiet
```

その後、Cloud Console から `mynet-us-vm` の SSH をクリックする。

確認ポイント:

* SSH 接続できなくなる  
* コンソールの SSH も、裏側では TCP 22 番への到達性に依存していることがわかる

## ポイント

* `default-allow-icmp`: 外部 IP 宛ての ping を許可  
* `default-allow-internal`: VPC 内部の通信を許可  
* `default-allow-ssh`: SSH 接続を許可
