---
title: "Python Version and Dependency Management"
date: 2020-04-12T12:05:22+03:00
draft: true
---

# Python Version and Dependency Management

**Apr 12, 2020**
<!-- <sup>Last modified: **Apr 12, 2020**</sup> -->

This is yet another post regarding how to handle your Python working environment, or about the way I have been handling it for the past 5 years. There are probably other all-in-one solutions out there but I think on the long run, I find this setup simpler to scale and maintain.

# Installing Python the right way

If you are running linux or macos, you probably already have a system Python. You should forget using that as soon as you format your pc. The right way to manage your Python installation is to use `pyenv`: https://github.com/pyenv/pyenv

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

This will clone pyenv repository in your home folder. We will then add some environment variables to be able to work with pyenv cli. The following commands are from pyenv readme for Ubuntu.

```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
source ~/.bashrc
```

Now you should be able to use `pyenv` command on terminal, if you can't, try opening a new terminal window. List your installed Python versions:

```bash
pyenv versions
```

And listing installable Python versions are:

```bash
pyenv install --list
```

Before installing Python, check out common build problems: https://github.com/pyenv/pyenv/wiki/Common-build-problems For Ubuntu, these are Python dependencies recommended to install:

```bash
sudo apt-get install -y \
make \
build-essential \
libssl-dev \
zlib1g-dev \
libbz2-dev \
libreadline-dev \
libsqlite3-dev \
wget \
curl \
llvm \
libncurses5-dev \
libncursesw5-dev \
xz-utils \
tk-dev \
libffi-dev \
liblzma-dev \
python-openssl \
git
```

Let's install Python 3.8.2 by:

```bash
pyenv install 3.8.2
```

And set it as global Python with

```bash
pyenv global 3.8.2
```

Now we are able to use 3.8.2 as:

```bash
python --version
```

Of course you can install as many Python versions as you want, and enable them globally or locally in particular folders. You can find more details in pyenv docs.

# Managing Python Packages

The sane way to manage your project dependencies is to use `pipenv`. This is a tool that combines `pip` and `virtualenv` benefits.

```bash
sudo apt-get install -y python3-pip
pip3 install pipenv
```

Now in a project folder, run pipenv:

```bash
PIPENV_VENV_IN_PROJECT=1 pipenv run python
```

Pipenv is integrated with pyenv and by default will use your global python 3.8.2 to create a virtual environment (a folder named `.venv` is created) for your project. It will also create a toml formatted file named `Pipfile` which will contain details about your Python environment. Now we will add a new package dependency by:

```bash
pipenv install requests
```

This will install `requests` package in your project environment, and add it to your Pipfile. It will also create a `Pipfile.lock` json formatted file, which will fix the version of all the dependencies of your project as well as file hashes for security reasons. This lock file can be reused when deploying this environment elsewhere to ensure both environments end up installing exactly same packages with correct versions and hashes.

```bash
cat Pipfile
```

Now let's change Pipfile manually.

To manually trigger updating the lock file, we can run:

```bash
pipenv lock
```

And we can update installed packages by:

```bash
pipenv install
```

Or we could run `pipenv update` that does update the lock file and then installs missing packages to your environment. To use this environment, always run python with

```bash
pipenv run python project.py
```

Check installed packages in your environment:

```bash
pipenv graph
```

You can also spawn a new shell with this environment activated by `pyenv shell`. More details can be found by running just `pipenv`.

Now you can work on bunch of independent projects with their unique working environment and project specific dependencies.