# Notes on GitHub Actions

## Install libvirt

```sh
sudo apt install qemu-kvm libvirt-daemon-system
```

From <https://ubuntu.com/server/docs/virtualization-libvirt>

Test with `virt-host-validate`

You should see `QEMU... : PASS`

I believe that `WARN` is OK for IOMMU and secure.

Make sure the needed user in in the `libvirt` group.
You may also need to be in the `libvirt-qemu` group (not sure).

## Install minikube

Following directions here: <https://minikube.sigs.k8s.io/docs/start/>

```sh
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

Test that this will work with kvm:

```sh
minikube start --driver=kvm2
```

See <https://minikube.sigs.k8s.io/docs/drivers/kvm2/>

**Problem:** Need to make sure that it's using space that's not in /home (not on NFS).

Solution: `export MINIKUBE_HOME=/scr/minikube`

Assuming things are working so far, set the default diver to kvm.

```sh
minikube config set driver kvm2
```

Added the following to my `.zshrc`

```sh
alias kubectl="minikube kubectl --"
```

## Start installing/configuring

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

Create a token [on github](https://github.com/settings/tokens).
I'm not sure what permissions are needed.
For testing, I chose repo and workflow.
Make sure you save this as it's impossible to see it again.

The directions say to run the following, but I got an error (see below).

```sh
kubectl apply -f \
    https://github.com/actions/actions-runner-controller/\
    releases/download/v0.22.0/actions-runner-controller.yaml
```

**Problem:** I got this error: `invalid: metadata.annotations: Too long: must have at most 262144 bytes`.

Solution: Use `kubectl replace` or `kubectl create`.

Set up the token you created above.

```sh
kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_token=`
```

Create a `runnerdeployment.yaml` file with the following content.

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runnerdeploy
spec:
  replicas: 1
  template:
    spec:
      repository: powerjg/actions-test
```

## Turning it off

```sh
minikube stop
```

No idea if that's right, but it seems reasonable.

## Other notes

We probably want to use special labels for builder nodes vs runner nodes so that we can use more cores to build faster.
We can add custom labels to the runners and then specify those labels in our scripts.

See <https://github.com/actions/actions-runner-controller/blob/master/docs/using-arc-runners-in-a-workflow.md>.

Can we use ccache? <https://github.com/hendrikmuhs/ccache-action>

## Things we need to do

- Document everything we need to migrate
  - Deploy docker images (can do this only when the dockerfiles are changed)
  - Compiler tests
  - nightlies
  - CI tests
  - doxygen builder
  - Things we no longer need:
    - gem5 bot
    - mirror bot
- Get this working on amarillo and azacca (also loupe!)
- Come up with some possible architectures
- Can we create the gem5 binary and then pass it off to the tests?
- Run some simple things on github servers
- Could potentially pass the binary around between the build and test parts. See <https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts#passing-data-between-jobs-in-a-workflow>
