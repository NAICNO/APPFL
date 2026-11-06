# POSIX service account

## Why APPFL needs a service account?

- To serve as the dedicated, isolated local user
- To execute incoming federated tasks
  - When the authorized Globus identity submits a job to the endpoint, Globus Compute maps the submitter identity directly to run locally as service account (e.g. `flamby_svc`)

### Following endpoint-setup guide

[Step 3 — Create the service account](https://github.com/NAICNO/APPFL/blob/main/Endpoint_Setup_and_Testbed_Verification/docs/source/1-endpoint-setup.md#step-3--create-the-service-account) **created the account but didn't create the home directory**

- [Step 3 — Create the service account](https://github.com/NAICNO/APPFL/blob/main/Endpoint_Setup_and_Testbed_Verification/docs/source/1-endpoint-setup.md#step-3--create-the-service-account) command and error

```bash
$ sudo -iu flamby_svc bash -lc 'source "$(conda info --base)/etc/profile.d/conda.sh" && conda activate appfl_env && python -V'

> Creating home directory for flamby_svc.
sudo: unable to change directory to /home/flamby_svc: No such file or directory
```

- Try creating a service account with `useradd -m flamby_svc`

```none
$ useradd -m flamby_svc
> useradd: user 'flamby_svc' already exists

$ ls /home/flamby_svc
> ls: cannot access '/home/flamby_svc': No such file or directory
```

### Verifying the service account

```
$ getent passwd flamby_svc
> flamby_svc:x:1000:1003::/home/flamby_svc:/bin/bash

$ id flamby_svc
> uid=1000(flamby_svc) gid=1003(flamby_svc) groups=1003(flamby_svc)
```

## Creating ‘/home/flamby_svc’

- Login to `hrothmund.hpc` -> `bioint04`
- Verifying $USER

```bash
sudo -i
# echo $USER
root
```

- Create `/home/flamby_svc`

```bash
mkdir -v /home/flamby_svc
# > mkdir: cannot create directory ‘/home/flamby_svc’: Permission denied
```
