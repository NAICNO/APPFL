# Install conda in Lightweight HPC

## Install via EasyBuild

Guide:

1. Load modules `EESSI/2023.06` and `EESSI-extend/2023.06-easybuild`
2. Check config with `eb --show-config`
3. Search for miniconda
4. Install `Miniconda3-25.7.0-2.eb`

```bash
module load EESSI/2023.06
module load EESSI-extend/2023.06-easybuild
eb --search miniconda
eb Miniconda3-25.7.0-2.eb
```

*Verify conda instalation:*

```none
== Build succeeded for 1 out of 1 (total: 2 mins 49 secs)
== Summary:
   * [SUCCESS] Miniconda3/25.7.0-2
== Temporary log file(s) /tmp/eb-5utf2vvf/easybuild-0ke6oof8.log* have been removed.
== Temporary directory /tmp/eb-5utf2vvf has been removed.


$ module avail miniconda

-------------------------------------------- /home/pubuduss/eessi/versions/2023.06/software/linux/x86_64/generic/modules/all ---------------------------------------------
   Miniconda3/25.7.0-2
   
```

**Add module directory and update `$MODULEPATH`:**

```bash
# update ~/.bashrc
module use /home/pubuduss/eessi/versions/2023.06/software/linux/x86_64/generic/modules/all
```

### Notes

- Install `screen` with `sudo dnf install screen`
- Verify screen session

```none
$ screen --version
Screen version 4.08.00 (GNU) 05-Feb-20
```
