# Deploying JupyterHub to OpenShift

This is entirely based on the work of Graham Dumpleton <gdumplet@redhat.com> who has been very helpful in getting this
set up. Mostly it is based on instructions found in his
[jupyter-notebooks](https://github.com/jupyter-on-openshift/jupyter-notebooks) and
[jupyterhub-quickstart](https://github.com/jupyter-on-openshift/jupyterhub-quickstart)
GitHub repos.

An earlier version of the instructions can be found [here](README-v1.md).

Unlike the earlier instructions all artifacts used are checked into this repo avoid problems with changes in remotely
located files.

What is deployed is:

* A new project named `jupyter` where all the action takes place.
* s2i builders for basic, scipy and tensorflow notebook images.
* A new `jupyterhub` database in the PostgreSQL database that resides in the `openrisknet-infra` project, along with a
secret in this `jupyter` project that contains the database credentials.
* JupyterHub.
* Notebooks and hub support JupyterLab interface
* SSO using Keycloak from the `openrisknet-infra` project.

## Prerequistes

OpenShift cluster with:

* Dynamic volume provisioning (e.g. using GlusterFS)
* A PostgreSQL database running in the `openrisknet-infra` project.
* Red Hat SSO (Keycloak) running in the `openrisknet-infra` project with a suitable realm (e.g. `openrisknet`)

## Deploy

The main deployment can be done as a user without `cluster-admin` privs. However, provisioning of the PostgreSQL database
in the `openrisknet-infra` project does need admin privs.

### new project
```
oc new-project jupyter
```

### Build Jupyter images
```
oc create -f templates/build-configs/s2i-minimal-notebook.yaml
oc create -f templates/build-configs/s2i-scipy-notebook.yaml
oc create -f templates/build-configs/s2i-tensorflow-notebook.yaml
```
This takes about 15 mins. Whilst that is running you can get some other things ready.

### Create the image stream for the JupyterHub image

```
oc create -f templates/image-streams/jupyterhub.yaml
```

### Load the template to deploy JupyterHub

```
oc create -f templates/jupyterhub/jupyterhub-deployer.yaml
```

### Set up SSO

In Keycloak go to the appropriate realm (e.g. `openrisknet`) and add `jupyterhub` as a new client.
Specify `confidential` as the `Access Type`. 
The Redirect URL will need to be something like `https://jupyterhub-jupyter.prod.openrisknet.org/*` (or whatever you specify
as the `ROUTE_NAME` parameter when you deploy JupyterHub).

You will need to know the client secret that is generated.

### Set up the PostgreSQL database

Unlike Graham Dumpleton's templates which provision a PostgreSQL database in
the `jupyter` project just for the `jupyterhub` application we instead use the
central database that is in the `openrisknet-infra` project.
To do this we run a database provisioner playbook that creates a new
database named `jupyterhub`, a database user named `jupyterhub` and a
randomly generated password for that user. These are stored in a secret in
the `jupyter` project and used by the `jupyterhub` pod.

__Note__: this playbook is currently located in the
[Squonk repo](https://github.com/InformaticsMatters/squonk). It will soon 
be added to this repo. 

As an admin user you need to source the appropriate `setenv.sh` file that
describes your OpenShift environment and where the `IM_PARAMETER_FILE`
variable in it points to a suitable YAML-based parameter file that defines
values for the following variables, an example with some typical values
is shown below: -

```yaml
oc_admin: admin
oc_admin_password: -SetMe-
oc_infra_project: orn-infra
oc_infra_sa: -SetMe-
oc_master_url: -SetMe-
oc_postgresql_service: db-postgresql
```

Then run:

```
ansible-playbook playbooks/infra/create-user-db.yaml \
    -e oc_db=jupyterhub \
    -e oc_db_user=jupyterhub \
    -e db_namespace=jupyter
```

Once added you can check for a secret named `database-credentials-jupyterhub`
in the `jupyter` project that contains the database connection details.

If you need to delete these you can run:

```
ansible-playbook playbooks/infra/delete-user-db.yaml \
    -e db=jupyterhub \
    -e oc_db_user=jupyterhub
```

After running those playbooks you need to switch back to the jupyter project:

```
oc project jupyter
```

### JupyterHub Configuration

Create the jupyterhub_config.py configuration file. A template named `jupyterhub_config_template.py` is provided in this 
dir but you may want different options. You can store these configurations in the `jupyterhub_configs` dir which is excluded 
from git.
You must replace the correct value for the `c.OAuthenticator.client_secret` property, the various URLs, and maybe some
other values. 

TODO: work out how to specify the need for specific role(s) for authorisation.

### Custom JupyterHub image

Sometimes you might need to build a custom jupyterhub image rather than using the prebuilt ones.
One way to do this is

1. Fork the https://github.com/jupyter-on-openshift/jupyterhub-quickstart repo
2. Make appropriate changes e.g. to the `requirements.txt` and `build-configs/jupyterhub.json` files
3. Delete the jupyterhub imagestream that was created above
4. Create a new build config for the jupyterhub image using soemthing like this: 
`oc create -f https://raw.githubusercontent.com/tdudgeon/jupyterhub-quickstart/develop/build-configs/jupyterhub.json`
(change the GitHub location to your fork). You need to make sure the image stream tag being built 
corresponds to the one in the `jupyterhub-deployer` template.

### Persistent Volumes

JupyterHub uses a separate persistent volume claim for each user. These PVCs are retained so if the notebook pod is depeted
(which happens automatically after a period of inactivity) then the PVC with your notebooks will still be present and will be
re-claimed next time the user's pod starts.

The easiest way to handle provisioning of PVCs is using dynamic provisioning (e.g. using GlusterFS or Cinder) but if this is not possible
then you can use a pool of pre-generated PVs. For example, this can be done using NFS volumes.

First create a number of NFS exports e.g. /nfs-jupyter/jupyter-1, /nfs-jupyter/jupyter-2 ...
This can be done with a PV definition like this:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jupyter-1
  labels:
    purpose: jupyter
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: prod-infra
    path: "/nfs-jupyter/vol1"
```

Set the ownership of these directories to root.root and the permissions to 770

In your `jupyterhub_config.py` file you will need to add these additional properties:

```
c.KubeSpawner.supplemental_gids = [ 65534 ]
c.KubeSpawner.storage_class = ''
c.KubeSpawner.storage_selector = { 'matchLabels': {'purpose': 'jupyter'}}
```

The `storage_selector` label ensures the Jupyter PVC only bind the expected PVs.

**Note**: the `storage_class` and `storage_selector` properties are not yet in the main JupyterHub codebase.
You might need to build a custom image as described above. An example repo can be found at https://github.com/tdudgeon/jupyterhub-quickstart/tree/develop 

### Deploy

Deploy it using:
```
oc new-app --template jupyterhub-deployer --param JUPYTERHUB_CONFIG="`cat jupyterhub_configs/jupyterhub_config.py`"
```
If you want a different hostname for the route (the default will be something like `jupyterhub-jupyter.your.domain.org`)
you can specify this as the `ROUTE_NAME` parameter. e.g. add `--param ROUTE_NAME=jupyter.prod.openrisknet.org` to that 
command.

The value of the `JUPYTERHUB_CONFIG` is used to create a ConfigMap named `jupyterhub-cfg`. If you need to change the settings
you can edit that ConfigMap and re-deploy JupyterHub.

## Delete
Delete the deployment (buildconfigs, imagestreams, secrets and pvcs (user's notebooks) will remain):
```
oc delete all,configmap,serviceaccount,rolebinding --selector app=jupyterhub
```

Or delete everything:
```
oc delete project jupyter
```

### TLS

By default trusted TLS certificates are not deployed. Once you are happy with the setup you can change this by changing 
the value of the `kubernetes.io/tls-acme` annotation for the route to `true`. You should also update the `jupyterhub_config.py`
file to set the appropriate TLS settings (in two places) and then redeploy.

## Database backups

__Note__: This section is no longer relevant as the database is now located in the `openrisknet-infra` project so backups 
need to be handled there.

These example templates provide backups of your JupyterHub database
using the Informatics Matters backup container image.

>   For the following recipe to work the JupyterHub database must be configured
    to permit remote access to the admin account, i.e. a password must be
    provided in the database Pod's `POSTGRESQL_ADMIN_PASSWORD` environment
    variable. If a password is not set you can run `psql -U postgres` from
    within the Pod and change the password with the `\password` meta command
    of psql.

-   [PVC](backup-pvc.yaml)
-   [CronJob](backup.yaml)

The backup image provides rich control of actions
through a number of container environment variables.

The backup essentially creates a compressed SQL file from the
result of running a `pg_dumpall` command.

The example template has a backup schedule that results in the
backup running at **00:07** each day whilst maintaining a history of
**4** of the most recent backups.

>   If you want to change the total number of backups that are held set the
    `BACKUP_COUNT` parameter, with `-p BACKUP_COUNT=<n>`.

To deploy the backup, using an OpenShift login on the Jupyter project,
first create the storage and then start the CronJob. You will need
the admin password of the database and (optionally) the host and
admin user (defaulted to `jupyterhub-db` and `jupyterhub` respectively).

So, if the admin password is `xyz123` and you're using the default Jupyter
database host service name and admin username you can deploy with the
following sequence of commands, after logging onto OpenShift as an admin user:

```
oc project jupyter
oc process -f backup-pvc.yaml | oc create -f -
oc process -f backup.yaml -p PGADMINPASS=xyz123 | oc create -f -
```

Or, with a different host, the backup can be deployed with: -

```
oc process -f backup.yaml -p PGADMINPASS=xyz123 \
    -p PGHOST=jupyterhub-db | oc create -f -
```

### Database recovery
A recovery image can be launched using the example Job template.

-   [Job](recovery.yaml)

This template, which has the same database user parameters as the backup
template, and attempts to recover the database from the latest backup can
be run with the following command: -

```
oc process -f recovery.yaml -p PGADMINPASS=xyz123 \
    -p PGHOST=jupyterhub-db | oc create -f -
```

>   It is important to ensure that the backup CronJob is not running
    or will not run when you recover the database.
 
