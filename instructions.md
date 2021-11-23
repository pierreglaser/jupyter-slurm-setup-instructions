# Working with (ana)conda

First, some semantic clarification:

## Anaconda

- Anaconda, inc. is a for-profit company creating data science software for companies.
- Among Anaconda's main product are:
  1. An Python distribution called "Anaconda". It comes with > 1500 packages directly, as well as a package manager called conda, that we will present later on.
  The big advantage is that this distribution is that the versions of the packages were carefully chosen to be compatible with each other.
  2. A channel named "anaconda" that mirrors the packages contained in the Anaconda distribution.


## Conda

- [https://github.com/conda/conda](conda) is an open source, (binary) package manager. The killer features are as follows:
  - Conda offers the possibility to create and manage environments. An environment is a set of packages aware of each other existence (and thus, that can act as dependencies of each other). For instance,
  numpy depends on the python interpreter - scikit-learn depends on numpy. When using a scikit-learn version installed in an environment "env", this scikit-learn will rely on the numpy installed in the same environment "env". This feature is very useful when working on different coding projects: each project may require specific packages versions to work: in that case, you can create one environment per package.
  - Conda comes with a "base" environment, which I recommend you to activate everytime you open a bash shell.
  - As explained, conda is a binary package manager, and not a simple python package manager. You can install R or even gcc (a C compiler) by typing "conda install gcc". Provided that anaconda was installed in a user folder, all packages will be installed in user folders, thus installing packages conda environment do not require you to be sudo, as opposed to other binary package managers such as apt-get on ubuntu.
  - Conda exposes executables specific to a single environment when activating such environment (using conda activate). It does so by mutating the "$PATH" environment variable. Thus, working with environment-specific executables "feels" like working with system-wide executables (such as the ones in "/usr/bin", or "/bin").
  - Conda installs packages through channels. The "anaconda" channel was mentioned previously.
  - The anaconda channel can contain closed source software, such as icc (an intel proprietary C/C++ compiler target for Intel CPU architectures) or MKL (an implementation of BLAS targeted for Intel CPU architectures). Another important channel is conda-forge, which contains only open source packages


## Miniconda

- Miniconda is a lightweight python distribution that comes with an installer, conda, but far fewer packages than the anaconda distribution.

## Miniforge

- Miniforge is yet another python distribution/installer with small differences w.r.t miniconda, including support for more computer architecture, and the use of conda forge as a default channel.

## Setup instructions

In practice, I recommend using the most portable version of these tools, which is `miniforge`. Miniforge can be installed on UNIX compliant OS (MacOS, Linux)
as follows:

```sh
#!/usr/bin/env bash
if [[ $(uname) == "Darwin" ]]; then
    MINIFORGE_SCRIPT_NAME="Miniforge3-MacOSX-arm64.sh"
elif [[ $(uname) == "Linux" ]]; then
    MINIFORGE_SCRIPT_NAME="Miniforge3-Linux-x86_64.sh"
else
    echo "Unrecognized OS"
fi
if [[ ${MINIFORGE_SCRIPT_NAME} != "" ]]; then
    curl -LO "https://github.com/conda-forge/miniforge/releases/latest/download/${MINIFORGE_SCRIPT_NAME}"
    bash ${MINIFORGE_SCRIPT_NAME} -b -p ${HOME}/.local/miniforge
    rm ${MINIFORGE_SCRIPT_NAME}
fi
```

Optionally, I recommend that login shells always activate your conda "base"
environment in your base environment by appending these lines to your shell rc file (~/.bashrc in Linux, ~/.bash_profile on MacOS, ~/.profile if none of these files exists):

```sh
source $HOME/.local/miniforge/etc/profile.d/conda.sh
conda activate
```


# Interactive development using jupyter(lab)

## The `jupyter` ecosystem

- "Jupyter" is an organization that created a suite of development tools particularly suited for interactive scientific computing and data analysis.
- The original Jupyter tool is "jupyter notebook", but a most recent and feature complete program is `jupyterlab`, which is what I personally use.
- VSCode provides "native" a `jupyter` notebook experience as part of the VSCode Python extension, and I heard it is good. Setup guides should be available online.


## Setup instructions

### Installing `jupyterlab`

`jupyter` is a development tool, and as such its installation should be decoupled from your conda environment projects. Thus, I recommend you use a specific environment
to install `jupyterlab`:

```sh
conda create --name jupyterlab jupyterlab
```

To run `jupyterlab` from a fresh bash shell, either activate the `jupyterlab` environment before typing the command:

```sh
conda activate jupyterlab
jupyter lab
```

Or alternatively (less recommended), you can symlink the `jupyterlab` command into an entry available in your $PATH:

```sh
# you only need to do this once:
conda activate jupyterlab
ln -s $(which jupyter) ~/.local/bin/jupyter
conda deactivate
```

### Pointing environments to a `jupyterlab` using ipykernel

To let `jupyterlab` know of a virtual environment present in your machine, you need to use `ipykernel`.
Here, we show how to create an environment for my project "my-project", and link it to `jupyterlab`:

```sh
conda create -n my-project-env ipykernel -y  # whenever you create a new environment, install ipykernel along side it to later connect it with jupyter..
conda activate my-project-env
python -m ipykernel install --prefix=/path/to/jupyterlab/conda/env --name="my-project-env"  # this assumes that jupyterlab was installed as instructed above
conda deactivate

# make sure this environment is now available
conda activate jupyterlab && jupyter lab
```

## Working on the cluster

### Configuring ssh

SWC has a set of computing resources administered by the `slurm` cluster manager. The cluster is accessed via a login node, `ssh.swc.ucl.ac.uk`
In order to connect to this login node without having to type your password every time, you can configure your ssh connections by creating a ssh private/public key pair

```sh
cd ~/.ssh
ssh-keygen -t rsa -f id_rsa_gatsby -q -N ""
ssh-copy-id <your-cluster-username>@ssh.swc.ucl.ac.uk
```

You can make your life even simpler by adding the following lines in your ~/.ssh/config file:

```
Host sgw2
User <your-cluster-username>
Hostname sgw2
ProxyCommand ssh sgw exec nc %h %p
IdentityFile  ~/.ssh/id_rsa_gatsby
LogLevel QUIET

Host sgw
User <your-cluster-username>
Hostname ssh.swc.ucl.ac.uk
IdentityFile  ~/.ssh/id_rsa_gatsby
LogLevel QUIET
```

Now you can simply run `ssh sgw2` - It should ask you for a password at most once a day.



### Running jupyter from a node of the compute cluster

Running jupyter remotely will be useful when doing interactive work using GPUs, or larger scale data analysis.
1. The first thing you need to do is installing conda and jupyterlab in the cluster.
   Your $HOME directory of the login node of the cluster is part of a network file system (nfs), that is mounted on all nodes of the cluster. Thus, installing programs in your $HOME available to all nodes can be done from any node, including the login node (but don't do any compute work on the login node though!), and the installation steps of conda/jupyterlab done locally previously port transparently to the cluster setting.
2.  As said before, you should not run any compute-heavy programs on the login node. Instead, you should request a node to slurm using the following command:

```sh
srun --job-name=jupyter --pty /bin/bash -l
```

Which should launch a bash interactive shell inside a compute node.

Now, you can activate the jupyterlab environment and launch jupyter lab on the compute node:
```
conda activate jupyterlab
jupyter lab
```

At this point however, the jupyter server will serve its web application into a port of the compute node, and thus cannot accessed from your machine.
To use access the jupyterlab web interface from your machine, you need to set up port forwarding between your local machine and the compute node.
This can be done using the following script:

```sh
#!/usr/bin/env bash
set -o errexit

echo "querying current jupyter sessions..."
# || true is needed to make grep not exit 1 (triggering the errexit option) if there is no currently
# running jupyter sessions
jupyter_host_name=$(ssh sgw2 squeue --name=jupyter --user=<your-cluster-username> -o "%R" | grep -v NODELIST || true)

if [[ ${jupyter_host_name} = "" ]]; then
    echo "no jupyter session found, exiting."
else
    num_lines=$(echo ${jupyter_host_name} | wc -l)
    if [[ $num_lines -ne 1 ]]; then
        echo "there is more than one jupyter session ($num_lines)"
        echo "${jupyter_host_name}"
        echo "exiting."
    else
        # XXX: ucl is a custom hostname linking to the swc cluster. You should use your own hostname.
        echo "jupyter session found at ${jupyter_host_name}, setting up port forwarding..."
        ssh -q -t -t ucl -L 8787:localhost:8787 -L 8789:localhost:8789 ssh -q -N ${jupyter_host_name} -L 8787:localhost:8787 -L 8789:localhost:8789
    fi
fi
```

Once you run this script, try to access: `http://127.0.0.1:8789/lab` from your favorite browser.
