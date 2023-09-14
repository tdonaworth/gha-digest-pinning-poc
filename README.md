# GHA-Digest-Pinning-PoC
This repo demonstrates how digest pinning isn't an end-all fix, and only one tool. 

## What is Digest Pinning?
Digest pinning is a way to lock the version to a specific commit. This is done by using the `@sha256` syntax, instead of the `@version` syntax. This applies in GitHub Actions (GHA), Docker, and other places.

```yaml
  # GHA Versions
  uses: actions/checkout@v4.0.0
  # versus
  uses: actions/checkout@3df4ab11eba7bda6032a0b82a6bb43b11571feac # v4.0.0

  # Docker Versions
  uses: docker://alpine:3.13.5
  # versus
  uses: docker://alpine@sha256:3df4ab11eba7bda6032a0b82a6bb43b11571feac # 3.13.5

  # Or within a Dockerfile
  FROM alpine:3.13.5
  # versus
  FROM alpine@sha256:def822f9851ca422481ec6fee59a9966f12b351c62ccb9aca841526ffaa9f748 # 3.13.5
```

If you work with dependencies in the JavaScript/npm ecosystem, you may be used to exact versions being immutable. For example, if you set a version like 2.0.1, you and your colleagues always get the exact same "code".

GHAs', Docker's, and even Python/pip version tags are not immutable versions. Effectively you can reassign the tag to a new commit, and the next time you run your workflow, you'll get the new code, without needing to change the version tag. This is considered bad practice by the maintainers, but there's nothing really enforcing or restricting this practice.

## So Digest Pinning is Good... Right?
Digest pinning is a good practice, but it's not a silver bullet. It's a tool in your toolbelt, but it's not the only tool.

Let's demonstrate how digest pinning can be misleading.

The act of pinning to a commit is still valid, but the core issue lies with additional dependencies down the chain. For example, I can pin to a specific GHA Digest, but if that GHA uses other non-digest resources, (e.g. Docker) I'm still vulnerable to those resources being changed.

```yaml
- name: Docker Action (remote)
  id: docker-action
  uses: tdonaworth/gha-digest-pinning-poc-docker-action@0552a8d3c45459bf5b7332525a853aaaeab9ee92 # v1.1.0
  # ^^^^^^^ This is a pinned digest action
```

The output of this Action is:

```bash
> Run tdonaworth/gha-digest-pinning-poc-docker-action@0552a8d3c45459bf5b7332525a853aaaeab9ee92

"Hello I'm a super-safe Docker Image v2.0 "
```

This is because the `tdonaworth/gha-digest-pinning-poc-docker-action` calls a Docker image `tdonaworth/digest-pinning-poc:2.0`. But what if I change the Docker image, build and push to the same version tag? 

```dockerfile
FROM alpine
#CMD ["echo", "Hello I'm a super-safe Docker Image v2.0 "]
CMD ["echo", "Hello I'm a super-evil Docker Image v2.0 mwah hahahahaha! "]
```

```bash
docker build -t tdonaworth/digest-pinning-poc:2.0 .
docker push tdonaworth/digest-pinning-poc:2.0
```

Now when I run the same GHA, I get the following output:

```bash
> Run tdonaworth/gha-digest-pinning-poc-docker-action@0552a8d3c45459bf5b7332525a853aaaeab9ee92

"Hello I'm a super-evil Docker Image v2.0 mwah hahahahaha! "
```

:exclamation: As yu can see, the GHA in our pipeline didn't change, but the output from the execution did! :exclamation:

## tl;dr - Audit Your GHAs; 
![audit-all-the-things](doc/image-1.png)



## Changes (easy changes to trigger a new build)
1. This is a change...
2. Another...
3. And another...
4. And another...
5. v2.0...
6. pinned

### References
This repo and poc was inspired by the following resource(s):
- [Unpinnable Actions: How Malicious Code Can Sneak into Your GitHub Actions Workflows](https://www.paloaltonetworks.com/blog/prisma-cloud/unpinnable-actions-github-security/) - Yaron Avital