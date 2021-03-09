# Elyra notebook for KF

build with this command:

```shell
docker build --format docker --squash-all -t quay.io/raffaelespazzoli/kf-elyra:2.0.1 -t quay.io/raffaelespazzoli/kf-elyra:latest .
docker push quay.io/raffaelespazzoli/kf-elyra:2.0.1
```