
#### Docker useful commands
```bash
# To remove all images which are not used by existing containers, use the `-a` flag:
docker image prune -a
# To remove the exited containers
docker rm $(docker ps -a -f status=exited -f status=created -q)
```

#### Docker & Crane

```shell
# Remote image size
crane manifest registry.env/platform/tests:v20231109-120332 | jq '.config.size + ([.layers[].size] | add)' | numfmt --to=iec
# Local image size
docker inspect artifactory.env/docker/golang:1.21-alpine | jq ".[] | .Size" | numfmt --to=iec
```

#### Containers
| Tool | Action |
|---|---|
| Podman | runs containers |
| Buildah | builds containers |
| Skopeo | transports containers |
| cri-o | oci container runtime interface |
| runc | runs containers |
| containerd | runs containers |

