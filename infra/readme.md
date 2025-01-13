## SOPS operator

SOPS operator is used to decrypt the sops secrets stored in the git repository and transform them into kubernetes secrets

Below is an example of how to create a secret with sops to safely store it on git.

Create a yaml SopsSecret:

```yaml
apiVersion: isindir.github.com/v1alpha3
kind: SopsSecret
metadata:
  name: sopssecret-sample
spec:
  secretTemplates:
    - name: sopssecret-sample
      labels:
        label0: value0
        labelK: valueK
      annotations:
        key0: value0
        keyN: valueN
      stringData:
        data-name0: data-value0
        data-nameL: data-valueL
```

Encrypt the secret:
```sh
sops encrypt test.yaml > test.enc.yaml
```

Decrypt the secret:
```sh
SOPS_AGE_KEY_FILE=./ignore/key.txt sops test.enc.yaml
```
