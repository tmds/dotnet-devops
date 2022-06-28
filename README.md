# dotnet-devops

This repo demonstrates continuous deployment of a .NET application by integrating the following services:

- GitHub source control
- Azure DevOps CI/CD
- quay.io image registry
- OpenShift deployment

## Running the demo

To run this demo you need:
- GitHub account (https://github.com/)
- Azure subscription with DevOps (https://dev.azure.com)
- quay.io account (https://quay.io/)
- OpenShift instance (https://developers.redhat.com/developer-sandbox/get-started)

We need to configure Azure DevOps with:
- A Docker Registry Service, this is used to push the .NET application image to `quay.io`.
- A Pipeline Environment, this is used to update the OpenShift resources

In OpenShift, we'll create a service account.
This is the account that is used by the pipeline environment.
We'll also add a pull secret to the account so it can pull the images from `quay.io`.


### Create a quay.io account

Create an account at https://quay.io/.

### Creating and configure the OpenShift Developer Sandbox

Create a developer sandbox at https://developers.redhat.com/developer-sandbox/get-started.

Copy the login command for the OpenShift Developer Sandbox web UI.

```
oc login --token=... --server=https://...
```

The sandbox is provisioned with two namespaces, we'll use the `<redhat-account-name>-stage` project.

```
oc project <redhat-account-name>-stage
```

Add a service account:
```
cat << EOF | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-sa
EOF
```

Add the edit role to the service account:
```
oc policy add-role-to-user edit system:serviceaccount:<redhat-account-name>:azure-sa
```

Create the pull secret with your `quay.io` credentials:
```
kubectl create secret docker-registry quay --docker-server=quay.io --docker-username=xxx --docker-password=yyy
```

Give the service account access to the secret:
```
kubectl patch serviceaccount azure-sa -p '{"imagePullSecrets": [{"name": "quay"}]}'
```

### DevOps: Create an Organization and a project

At https://dev.azure.com create a new organization, and in the organization create a project. For the organization name, you can accept what Azure suggests. For the project name, you can use something like `devops-demo`.

When you open the project, you'll see the _Pipelines_ tab on the left, and _Project settings_ at the bottom.

### DevOps: Add the Docker Registry Service

The Docker Registry Service is added to Azure DevOps by going to _Project Settings_ (bottom left), and creating a _New service connection_ in the _Service connections_ tab.

Select _Docker Registry_. At the top, pick: _Others_. For _Docker Registry_ add `https://quay.io`. For _Docker ID_, and _Docker Password_ use your `quay.io` credentials. As a _Service Connection Name_ use _DefaultDockerRegistryService_. At the bottom you can check _Grant access permission to all pipelines_.

### DevOps: Create the target environment

Under the _Pipelines_ > _Environments_ tab, create a new environment named `StagingEnvironment`, and choose the _Kubernetes_ resource type.

For the _Kubernetes resource_, pick: _Generic provider_.
As the _Cluster name_ enter _Developer Sandbox_ and as namespace enter the name of the stage project: _<redhat-account-name>-stage_.

For the _Server URL_, copy-paste and run the command from the UI.

```
kubectl config view --minify -o jsonpath={.clusters[0].cluster.server}
```

For the _Secret_, copy-paste and run the two commands from the UI.

You need to fill in the namespace (_<redhat-account-name>-stage_) and service-account-name (`azure-sa`) in the first command.

```
kubectl get serviceAccounts azure-sa -n <redhat-account-name>-stage -o=jsonpath={.secrets[*].name}
```

In the second command you need to fill in the secret name that was output from the first command.

```
kubectl get secret azure-sa-token-<xxxxxx> -n <redhat-account-name>-stage -o json
```

Copy the complete JSON output into the _Secret_ field.

### DevOps: Create the Pipeline

Create a fork of this repository under your own GitHub account.

When you create a new pipeline, select _GitHub_ and then select your fork of the project.
You'll be asked to grant DevOps access to your GitHub project.

DevOps will show you the `azure-pipelines.yml`. You need to update a few values.

`ImageName`. This variable is the name of the image that is pushed to `quay.io`. It has the format `<quay user>/<image>`. You need to update it with your username.

`TargetEnvironment`. This variable refers to the Kubernetes resource in the target environment we've created earlier. You need to update it with your _<redhat-account-name>_ so it matches `StagingEnvironment.<redhat-account-name>-stage`.

`Namespace`. This is the OpenShift namespaces that gets deployed to. You need to update it to match `<redhat-account-name>-stage`.

That's it.

Now click the _Run_, and watch the pipeline in action.