# YandexCloud BI
yc compute instance create \
  --name dkazanskii \
  --zone ru-central1-a \
  --platform standard-v3 \
  --cores 4 \
  --memory 8 \
  --create-boot-disk size=40 \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_finsrv.pub \
  --create-boot-disk image-id=fd876gids9srs8ma0592
