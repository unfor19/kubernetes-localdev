# kubernetes-localdev

**Work In Progress (WIP)**

For those who wish to deepen their knowledge, I'll be adding some Hands-on challenges.

## 1 - Earn Makefile

### Scenario

Willy Wonka, a junior DevOps enginner, protected his [Makefile](https://en.wikipedia.org/wiki/Make_(software)), that is located at [ch1-earn-makefile/earn-me](./ch1-earn-makefile/earn-me).

### Challenge

Find a way to read Willy's Makefile, so it will make sense.

The prize is quite nice, Willy wrote a Makefile which is related to this repository.

### Hint

<details>

<summary>Expand/Collapse</summary>

How would you decode a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/)?

</details>

### Solution

<details>

<summary>Expand/Collapse</summary>

The content of [ch1-earn-makefile/earn-me](./ch1-earn-makefile/earn-me) seems like it's encoded with base64. How can I assume that? A "long" string that ends with `==` is usually (not always) the base64 represenation of a normal string. When I see those kind of strings, I attempt to decode them with the built-in command [base64](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html).

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

Use `make help` to check for available commands

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

**Encoding** is nothing more than a different representation of the same thing. Just as `11` in binary is `3` in decimal, you just need to know the base and that's it, there's no protection at all.

Willy should've used an **Encryption** mechanism, such as password, private+public keys, etc., to protect his Makefile.

</details>