# Docker Image Cache Action

This repository provides a GitHub Action that handles the pulling and caching of Docker images so that you are not
constantly pulling images from public repositories like Docker Hub/GHCR etc and potentially hitting rate limits.

This also speeds up builds as you don't have to pull the images from public repositories each time but can merely
restore them from GitHub Actions cache instead which is generally much faster.

# Usage

Here's an example of using this action to obtain and cache two images:

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
      - name: Cache Docker Images
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: ${{ github.ref_name != 'main' }}
          images: |
            confluentinc/cp-kafka:7.7.1
            postgres:15-alpine

      # Some more build steps that use the Docker images
```

In the above example we are caching two images, and we set `restore-only` based on whether this is not the `main`
branch.  So for `main` we would pull the images if they aren't yet cached and upload them to the Actions cache, for any
other branch we will only restore the cache and load the images from there.  In the event that the cache was missing the
images won't be present so your job would still need to pull the images in some other job step.

This works because of how GitHub Actions
[matches caches][2], if the cache key is not present for a specific branch it falls back to using the cache entry from
the base branch, and ultimately falls back to your default branch (here we've assumed this is `main` as most modern
repositories use).  

If you have multiple workflows that want to share the cache see [Priming the Cache](#priming-the-cache) for approaches
to that.

# Inputs

| Input | Required? | Default | Purpose |
|-------|-----------|---------|---------|
| `images` | Yes | N/A | Specifies one/more image references that you want to pull and cache from Docker Hub |
| `restore-only` | No | `false` | When set to `true` will only restore images from the cache and not attempt to update the cache, this can mean that images are pulled from Docker Hub if they aren't yet cached. |
| `temp-path` | No | `.images` | Path for a temporary directly which is used for the cached image tarballs. |

Note that there is no input for controlling the cache key, the cache key is based upon hashing the `images` input.  If
you have multiple jobs that all want to use the cached images it is generally best to [Prime the
Cache](#priming-the-cache) as shown later in the README.

# Outputs

| Output | Description |
|--------|-------------|
| `cache-key` | The generated cached key associated with these image references. |

# Priming the Cache

As the cache key is generated from a hash of the image references if you have multiple jobs that want to use the cached
images you are better to have a job that primes the cache first.  You can then have the rest of your jobs depend on the
existing cache by calling this action with `restore-only` set to `true`.

Here are two approaches to priming the cache, these are by no means the only way to prime the cache, but address the
most common scenarios:

1. [Job Dependencies](#priming-via-job-dependencies)
2. [Cron Jobs](#priming-via-cron-job)

## Priming via Job Dependencies

For this method we define our workflow so that we have one job that populates the cache, and then one/more dependent
jobs that simply restore the cache e.g.

```yaml
name: Dependent Job Cache Priming
on: 
  push:
  workflow_dispatch:

jobs:
  prime-image-cache:
    runs-on: ubuntu-latest
    steps:
      # Populate Docker Cache
      - name: Populate Image Cache
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: false
          images: |
            confluentinc/cp-kafka:7.7.1
            postgres:15-alpine

  example:
    needs:
      - prime-image-cache
    runs-on: ubuntu-latest
    steps:
      - name: Restore Cached Images
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: true
          images: |
            confluentinc/cp-kafka:7.7.1
            postgres:15-alpine

      # Some job steps that use the images
```

Note that the cache key is still a fixed value, so if you have many separate workflows that want to share the cached
images then this approach is not for you, see the cron job approach.

## Priming via Cron Job

In this approach we have two/more separate workflows, we have one workflow that runs on a cron schedule to ensure the
cache is always primed:

```yaml
name: Cron Job Cache Priming
on: 
  schedule:
    # Every day at 6:30AM
    - cron:  '30 6 * * *'

jobs:
  prime-image-cache:
    runs-on: ubuntu-latest
    steps:
      # Populate Docker Cache
      - name: Populate Image Cache
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: false
          images: |
            confluentinc/cp-kafka:7.7.1
            postgres:15-alpine
```

Then in other workflows that need to use the cached images you always ensure that `restore-only: true` is set wherever
you call this action e.g. as seen in the [dependent job approach](#priming-via-job-dependencies).

## FAQs

### Can I control the cache key?

No, this is generated based on hashing the provided image references.  That way if you change the versions of the images
you are caching a new cache key and entry will be generated with the updated images.

If cache key collisions are causing you a problem please refer to our [Priming the Cache](#priming-the-cache) recommendations.

If that doesn't resolve your problem then please open an issue describing your use case in more detail so we can
reconsider whether future versions of this action provide more control over the cache key.

### Can I cache a `:latest` tag for an image?

**No**, attempting to do so will cause the action to fail.

### Can I cache images independently?

Yes, to do this you will need to invoke this action multiple times supplying a single image reference to each invocation e.g.

```yaml
name: Independent Caching Example
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
      - name: Cache Confluent Kafka
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: false
          temp-path: .images/kafka
          images: |
            confluentinc/cp-kafka:7.7.1

      - name: Cache Postgres
        uses: telicent-oss/docker-image-cache-action@v1
        with:
          restore-only: false
          temp-path: .images/postgres
          images: |
            postgres:15-alpine

      # Some more build steps that use the Docker images
```

Here each invocation of the action has a different `images` input and a different `temp-path` input so will generate
different [cache keys](#can-i-control-the-cache-key) and have different independent cache entries.

There are a couple of gotchas that apply here:

1. If you don't specify a different `temp-path` for each invocation of the action both images will still end up in the
   default `.images` directory together and both cache keys will contain both images, you **MUST** supply different
   `temp-path` inputs to cache images independently.
2. You **MUST** also restore each image independently, and specify a `temp-path` that matches that used to
   originally cache the images.  This is a side-effect of GitHub Actions [Cache Versioning][6] behaviour.

### Can I tell whether the images have been made available?

Only from the log output currently, future versions of this action **MAY** be enhanced to expose this as an output.

Currently the best way to check if images are available is with another job step, as done in our
[self-tests](.github/workflows/self-tests.yml) e.g.

```yaml
    - name: Verify Confluent Kafka available
      run: |
        docker image inspect confluentinc/cp-kafka:7.7.1
```
As this step will fail if the given image isn't available locally.

Depending on how your workflow functions this check may be unnecessary anyway. For example, if you are running some
build steps that will start/build an image from the images you attempted to cache, then those toolchains will generally
pull the image if it isn't available unless you explicitly disabled that behaviour.

### How do I cache images from private repositories?

This action just uses `docker pull` to retrieve images, so as long as your workflow has authenticated itself to your
private repository, whether explicitly via `docker login`, or via a job stp using
[`docker/login-action`][3]/[`aws-actions/amazon-ecr-login`][4]/etc., then this action can be used to cache images from
it.

Please bear in mind the following from [GitHub Actions documentation][5]:

> We recommend that you don't store any sensitive information in the cache.
>
> Anyone with read access can create a pull request on a repository and access the contents of a cache. Forks of a
> repository can also create pull requests on the base branch and access caches on the base branch.

**TL;DR** If you cache a private image on a public repository you have potentially exposed that private image.

# License

This Action is licensed under the Apache License 2.0, see [LICENSE](LICENSE) and [NOTICE](NOTICE) for more information.

[1]: https://docs.docker.com/docker-hub/usage/
[2]: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache
[3]: https://github.com/docker/login-action/tree/v3.3.0/
[4]: https://github.com/aws-actions/amazon-ecr-login
[5]: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#about-caching-workflow-dependencies
[6]: https://github.com/actions/cache/blob/main/README.md#cache-version
