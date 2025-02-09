# Install Percona Distribution for PostgreSQL on Minikube

Installing the Percona Operator for PostgreSQL on [minikube](https://github.com/kubernetes/minikube)
is the easiest way to try it locally without a cloud provider. Minikube runs
Kubernetes on GNU/Linux, Windows, or macOS system using a system-wide
hypervisor, such as VirtualBox, KVM/QEMU, VMware Fusion or Hyper-V. Using it is
a popular way to test the Kubernetes application locally prior to deploying it
on a cloud.

The following steps are needed to run Percona Operator for PostgreSQL on
minikube:

1. [Install minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/),
using a way recommended for your system. This includes the installation of
the following three components:

    1. kubectl tool,
    2. a hypervisor, if it is not already installed,
    3. actual minikube package

    After the installation, run `minikube start` command. Being executed,
    this command will download needed virtualized images, then initialize and run
    the cluster. After minikube is successfully started, you can optionally run
    the Kubernetes dashboard, which visually represents the state of your cluster.
    Executing `minikube dashboard` will start the dashboard and open it in your
    default web browser.

2. The first thing to do is to add the `pgo` namespace to Kubernetes,
    not forgetting to set the correspondent context for further steps:

    ``` {.bash data-prompt="$" }
    $ kubectl create namespace pgo
    $ kubectl config set-context $(kubectl config current-context) --namespace=pgo
    ```

    !!! note

        To use different namespace, you should edit *all occurrences* of
        the `namespace: pgo` line in both `deploy/cr.yaml` and
        `deploy/operator.yaml` configuration files.

    If you use Kubernetes dashboard, choose your newly created namespace to be
    shown instead of the default one:

    ![image](assets/images/minikube-ns.svg)

3. Deploy the operator with the following command:

    ``` {.bash data-prompt="$" }
    $ kubectl apply -f https://raw.githubusercontent.com/percona/percona-postgresql-operator/v{{ release }}/deploy/operator.yaml
    ```

4. Deploy Percona Distribution for PostgreSQL:

    ``` {.bash data-prompt="$" }
    $ kubectl apply -f https://raw.githubusercontent.com/percona/percona-postgresql-operator/v{{ release }}/deploy/cr-minimal.yaml
    ```

    This deploys PostgreSQL on one node, because `deploy/cr-minimal.yaml` is
    for minimal non-production deployment. For more configuration options please
    see `deploy/cr.yaml` and [Custom Resource Options](operator.md#operator-custom-resource-options).

    Creation process will take some time. The process is over when both
    operator and replica set pod have reached their Running status:

    ``` {.bash data-prompt="$" }
    $ kubectl get pods
    ```
    ??? example "Expected output"

        ``` {.text .no-copy}
        
        NAME                                                    READY   STATUS      RESTARTS   AGE
        backrest-backup-minimal-cluster-dcvkw                   0/1     Completed   0          68s
        minimal-cluster-6dfd645d94-42xsr                        1/1     Running     0          2m5s
        minimal-cluster-backrest-shared-repo-77bd498dfd-9msvp   1/1     Running     0          2m23s
        minimal-cluster-pgbouncer-594bf56d-kjwrp                1/1     Running     0          84s
        pgo-deploy-lnbv7                                        0/1     Completed   0          4m14s
        postgres-operator-6c4c558c5-dkk8v                       4/4     Running     0          3m37s
        ```

    You can also track the progress via the Kubernetes dashboard:

    ![image](assets/images/minikube-pods.svg)

5. During previous steps, the Operator has generated several [secrets](https://kubernetes.io/docs/concepts/configuration/secret/),
    including the password for the `pguser` user, which you will need to access
    the cluster.

    Use `kubectl get secrets` command to see the list of Secrets objects (by default Secrets object you are interested in has `minimal-cluster-pguser-secret` name). Then you can use `kubectl get secret minimal-cluster-pguser-secret -o yaml` to look through the YAML file with generated secrets (the actual password will be base64-encoded), or just get the needed password with the following command:

    ``` {.bash data-prompt="$"}
    $ kubectl get secrets minimal-cluster-users -o yaml -o jsonpath='{.data.pguser}' | base64 --decode | tr '\n' ' ' && echo " "
    ```

6. Check connectivity to a newly created cluster.

    Run new Pod to use it as a client and connect its console output to your
    terminal (running it may require some time to deploy). When you see the
    command line prompt of the newly created Pod, run `psql` tool using the
    password obtained from the secret. The following command will do this,
    naming the new Pod `pg-client`:

    ``` {.bash data-prompt="$" data-prompt-second="[postgres@pg-client /]$"}
    $ kubectl run -i --rm --tty pg-client --image=perconalab/percona-distribution-postgresql:{{ postgresrecommended }} --restart=Never -- bash -il
    [postgres@pg-client /]$ PGPASSWORD='pguser_password' psql -h cluster1-pgbouncer -p 5432 -U pguser pgdb
    ```

    This command will connect you to the  PostgreSQL interactive terminal.

    ``` {.bash data-prompt="$" data-prompt-second="pgdb=>"}
    $ psql ({{ postgresrecommended }})
    Type "help" for help.
    pgdb=>
    ```
