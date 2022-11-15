try_podman_devcontainers
========================

You must add the following to your settings.json

```json
{
    "dev.containers.dockerPath": "podman"
}
```

> **Warning** Rebuild Container
> If you are changing settings like `remoteUser` and `containerUser` you must use the command
> **Dev Containers: Rebuild and Reopen in Container** for the changes to apply

Workarounds are commented out in `devcontainer.json`. The ideal scenario would be for podman and docker to work out-of-the-box with no changes to `devcontainer.json` required.

## Rootful

Set rootful mode:
```console
podman machine stop; podman machine set --rootful; podman machine start
```

Add the following to `~/.config/containers/containers.conf`:
```conf
[engine]
  env = ["BUILDAH_FORMAT=docker"]
```

> **Note** Issue 1
> `Command in container failed: mkdir -p '/root/.vscode-server/bin' && ln -snf '/vscode/vscode-server/bin/linux-arm64/6261075646f055b99068d3688932416f2346dd3b' '/root/.vscode-server/bin/6261075646f055b99068d3688932416f2346dd3b'`
> `mkdir: cannot create directory '/root': Permission denied`
>
> In Docker, this works out-of-the-box.
> Setting `{"containerUser" : "vscode" }` in the devcontainer allows the container to start
> but you do not have permissions to edit/save files:
> ```sh
> vscode ➜ /workspaces/try-podman-devcontainers $ ls -lah
> total 8.0K
> drwxr-xr-x 6  501 dialout 192 Nov 15 14:54 .
> drwxr-xr-t 3 root root     38 Nov 15 15:32 ..
> drwxr-xr-x 3  501 dialout  96 Nov 15 14:57 .devcontainer
> drwxr-xr-x 9  501 dialout 288 Nov 15 14:51 .git
> -rw-r--r-- 1  501 dialout  74 Nov 15 14:54 main.go
> -rw-r--r-- 1  501 dialout 790 Nov 15 15:01 README.md
> ```
>
> The only solution is to set `{"remoteUser" : "root" }`
> While created files are root:root inside the conatainer they are correctly 501:20 outside the container

Once you've overcome Issue 1, you can use the dev container for simple tasks.
You cannot however use anything that requires the 'Docker-from-Docker' feature since mounting
/var/run/docker.sock inside containers fails with Permission Denied since the socket permissions aren't changed when mounted into the container.

## Rootless

Set rootless mode:
```console
podman machine stop; podman machine set --rootful=false; podman machine start
```

> **Note** Issue 2
> Container won't start with a similar output to Issue 1. The workaround this time though is to set `remoteUser` and `containerUser` to `vscode`.
> However, files in the container are owned by 0:65534 so you can't edit/save antyhing


Add the following to `~/.config/containers/containers.conf`:
```conf
[engine]
  env = ["BUILDAH_FORMAT=docker", "PODMAN_USERNS=keep-id"]
```

> **Note** Issue 3
> Permissions in the workspace are totally wrong
> ```sh
> vscode ➜ /workspaces/try-podman-devcontainers $ ls -la
> total 8
> drwxr-xr-x 6 core nogroup  192 Nov 15 15:47 .
> drwxr-xr-t 3 root root      38 Nov 15 15:49 ..
> drwxr-xr-x 3 core nogroup   96 Nov 15 14:57 .devcontainer
> drwxr-xr-x 9 core nogroup  288 Nov 15 14:51 .git
> -rw-r--r-- 1 core nogroup   74 Nov 15 14:54 main.go
> -rw-r--r-- 1 core nogroup 2249 Nov 15 15:49 README.md
> ```
> That's 501:65534 inside the container.

If we can't fix the permissions here we don't have a working solution.

## What's still missing?

- A podman-in-podman feature
- A podman-from-podman feature

Testing one or the other with the feature that adds kubectl, helm and minikube to allow for working with k8s from inside a devcontainer.
