# kubernetes-localdev

**Work In Progress (WIP)**

For those who wish to deepen their knowledge, I'll be adding some Hands-on challenges.

## 1 - Earn Makefile

### Scenario

Willy Wonka, a junior DevOps enginner, protected his [Makefile](https://en.wikipedia.org/wiki/Make_(software)), that is located at [ch1-earn-makefile/earn-me](./ch1-earn-makefile/earn-me).

### Challenge

Find a way to read Willy's Makefile so that it will make sense.

The prize is quite lovely, and Willy wrote a Makefile which is related to this repository.

### Hint

<details>

<summary>Expand/Collapse</summary>

How would you decode a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/)?

</details>

### Solution

<details>

<summary>Expand/Collapse</summary>

The content of [ch1-earn-makefile/earn-me](./ch1-earn-makefile/earn-me) seems like it's encoded with base64. How can I assume that? A "long" string that ends with `==` is usually (not always) the base64 representation of a normal string. When I see those kinds of strings, I attempt to decode them with the built-in command [base64](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html).

Here's how you can decode the `Makefile` using a one-liner Bash script.

```bash
base64 --decode challenges/ch1-earn-makefile/earn-me > Makefile
# A new file was created in $PWD
cat Makefile
```

`Makefile` (Short output)
```
.EXPORT_ALL_VARIABLES:
CATS_APP_NAME ?= baby
...
clean-minikube:
    minikube delete --purge --all
```

Use `make help` to check for available commands.

```
help                 Available make commands
start-minikube       Start minikube with docker driver and k8s version v1.20.2
check-versions       Check versions of relevant CLIs
helm-add-nginx-repo  Add and update the Helm repo ingress-nginx
run-cats             Run docker-cats locally
```

</details>

### Conclusion

<details>

<summary>Upon completing the challenge - Expand/Collapse</summary>

**Encoding** is nothing more than a different representation of the same thing. Just as `11` in binary is `3` in decimal, you need to know the base, and that's it; there's no protection at all.

Willy should've used an **Encryption** mechanism, such as password, private+public keys, etc., to protect his Makefile.

</details>

## 2 - Change The Image

### Scenario

As part of testing [docker-cats](https://github.com/unfor19/docker-cats), Willy wants to change the image of `baby.jpg` (even though it's adorable). The change should take place immediately, with minimum effort.

### Challenge

Deploy the application [1-baby.yaml](../1-baby.yaml), and then change the image `baby.jpg` to a different one, but keep the same file name `baby.jpg`.

You can use this image - [baby-new.jpg](https://d33vo9sj4p3nyc.cloudfront.net/kubernetes-localdev/baby-ch2.jpg), taken from here - https://pixabay.com/photos/cat-baby-baby-cat-kitten-cat-cute-4201051/

The main goal is to apply a change of a running application and get immediate feedback.

### Hints

<details>

<summary>Finding The Pod/Container - Expand/Collapse</summary>

The easiest option would be to use LENS (for this whole challenge). Let's try something else, check the official [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) to figure out how to get the names of the running Pods.

It's also possible to [access minikube's Docker Daemon](https://github.com/unfor19/kubernetes-localdev#build-the-application-ci), which in turn will assist with debugging running containers. This should be used when no other option is available, check this challenge's *Solution* to see what I mean.

</details>

<details>

<summary>Get In The Container - Expand/Collapse</summary>

First, you need to find the container's name (see the previous hint). Following that, you can execute

```bash
docker exec "$CONTAINER_NAME" -it bash
```

If you're going to use the above command, you'll get the error `"bash": executable file not found`. Figure out which command can be used to exec into the running container.

</details>


### Solution

<details>

<summary>Expand/Collapse</summary>

1. Find the `baby` container - instead of [grep](https://man7.org/linux/man-pages/man1/grep.1.html) you can use [kubectl get pods --field-selector ...](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/), though I think that using grep is simpler.

    ```bash
    kubectl get po | grep baby
    # baby-79d4f4c875-rb97c                   1/1     Running   0          41m
    ```
2. Exec into `baby`
    ```bash
    POD_NAME="baby-79d4f4c875-rb97c" # <-- Change this
    kubectl exec -it "$POD_NAME" -- sh
    ```
    I used `sh` because `bash` is not available in the running container. That is a common practice for [Alpine-based images](https://github.com/unfor19/docker-cats/blob/master/Dockerfile#L1).
3. Get in the `/usr/src/app/images` directory and make a backup of `baby.jpg`
    ```bash
    # In `baby` container
    /usr/src/app $ ls
    # Dockerfile         favicon.ico        index.html         package-lock.json  server.js
    # README.md          images             node_modules       package.json

    # Make a backup of current `baby.jpg`
    cd images/
    /usr/src/app/images $ cp baby.jpg baby.jpg.bak
    ```
4. Download the image [baby-ch2.jpg](https://d33vo9sj4p3nyc.cloudfront.net/kubernetes-localdev/baby-ch2.jpg) and rename it to `baby.jpg`. The curl application is not available in the container, at least not out of the box. 
   
   It's possible to install it with `apt update && apt install -y curl` because you're logged in as `root`. A better alternative is to use `wget`, which is available out of the box in the running container. Also common for Alpine-based images.
   ```bash
   # -O = output filename 
   IMAGE_URL="https://d33vo9sj4p3nyc.cloudfront.net/kubernetes-localdev/baby-ch2.jpg"
   wget -O baby.jpg "$IMAGE_URL"
   ```
5. Reload the page `baby.kubemaster.me`; you should see the newly downloaded baby cat image.
   
   ![results-ch2.png](https://d33vo9sj4p3nyc.cloudfront.net/kubernetes-localdev/results-ch2.png)
   

</details>

### Conclusion

<details>

<summary>Upon completing the challenge - Expand/Collapse</summary>

To apply an instant change of a running container, use the command [kubectl exec](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-running-pods). Instant changes should be done only in development environments since the changes are not permanent. Otherwise, once the Pod (container) restarts, the changes will be gone. Also, applying an immediate change without committing to a `git` repository makes it impossible to track changes.

The container `baby` is running as `root`, this means you can do ANYTHING. That's very bad in production environments, where the blast radius should be minimal. Think about it, if someone hacks into your application, they can download malicious code and access sensitive data with elevated permissions. To read more about "how to run a container as non-root user", check my blog post [Docker Tips And Best Practices](https://meirg.co.il/2021/02/11/docker-tips-and-best-practices/#:~:text=use%20the%20USER%20command).

Last but not least, assuming that `baby` runs with a non-root user, for example, `nobody`. Assuming you have full access to the Kubernetes Cluster (and Docker Daemon), how would you exec into a container with a `root` user?

<details>

<summary>Exec As Root - Expand/Collapse</summary>

Unfortunately, there's no out-of-the-box solution for `kubectl exec` to use a different user. So I'm going to use [docker exec --user](https://docs.docker.com/engine/reference/commandline/exec/#options).

```bash
# Use minikube's Docker Daemon
eval `minikube docker-env`

# Get container name
docker ps --filter "name=cats_baby"

# CONTAINER ID   IMAGE          COMMAND                  CREATED             STATUS             PORTS     NAMES
# f72482ace496   da4ce6f08470   "docker-entrypoint.s…"   About an hour ago   Up About an hour             k8s_cats_baby-79d4f4c875-rb97c_default_f8113de0-33be-4ef7-94f2-c3cf6ba98e6b_0

# Mine is: f72482ace496
CONTAINER_NAME="f72482ace496" # <-- Change this

# Check for available users in the running container
docker exec -it "$CONTAINER_NAME" cat /etc/passwd

# root:x:0:0:root:/root:/bin/ash
# ...
# nobody:x:65534:65534:nobody:/:/sbin/nologin

# Exec into the container with `nobody` or `root`
docker exec -it --user root "$CONTAINER_NAME"
# docker exec -it --user nobody "$CONTAINER_NAME"
```

</details>

</details>