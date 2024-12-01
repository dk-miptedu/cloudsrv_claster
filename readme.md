# YandexCloud BI

## Create VPS with standart settings

```bash
yc compute instance create \
--folder-id b1gn61susoertj6sijgb \
--name dkazanskii-inst05 \
--zone ru-central1-a \
--platform standard-v3 \
--cores 4 \
--memory 8 \
--create-boot-disk size=40,image-folder-id=standard-images,image-family=ubuntu-2204-lts \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--ssh-key ~/.ssh/id_finsrv.pub\
```

## Create VPS with users settings

```bash
yc compute instance create \
--folder-id b1gn61susoertj6sijgb \
--name dkazanskii-docker \
--zone ru-central1-a \
--platform standard-v3 \
--cores 4 \
--memory 8 \
--core-fraction 20 \
--create-boot-disk size=20,image-folder-id=standard-images,image-family=ubuntu-2204-lts \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--metadata-from-file user-data="calccloud_homework_02.yaml"
```
