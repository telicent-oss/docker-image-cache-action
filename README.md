# Docker Image Cache Action

This repository provides a GitHub Action that handles the pulling and caching of Docker images so that we are not
constantly pulling images from public Docker Hub.  This helps us to avoid hitting Docker Hub [rate limits][1] and speeds
up builds as we don't have to pull the images from Docker Hub each time but can merely restore them from GitHub Actions
cache instead.

# Usage

At its most basic the action is used as follows:

```yaml
name: Docker Image Cache Example
on: 
  push:
  workflow_dispatch:

jobs:
  example:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      # Normal Job setup steps happen...
     
      # Pull/Restore Docker Images from Cache
      - name: Trivy Filesystem Scan
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: ${{ github.ref_name != 'main' }}
          images: |
            confluentinc/cp-kafka:7.7.1
            postgres:15-alpine

      # Some more build steps that generate a Docker image...
```

In the above example we invoke the action twice, once to do a `fs` scan and another to do an `image` scan.

# Inputs

| Input | Required? | Default | Purpose |
|-------|-----------|---------|---------|
| `images` | Yes | N/A | Specifies one/more image references that you want to pull and cache from Docker Hub |
| `cache-key` | No | Hash of the image references | Specifies a cache key to use for this set of images. |
| `restore-only` | No | `false` | When set to `true` will only restore images from the cache and not attempt to update the cache, this can mean that images are pulled from Docker Hub if they aren't yet cached. |

# License

This Action is licensed under the Apache License 2.0, see [LICENSE](LICENSE) and [NOTICE](NOTICE) for more information.

[1]: https://docs.docker.com/docker-hub/usage/
