# fzf-docker

## Table of Contents

- [fzf-docker](#fzf-docker)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Installation](#installation)
    - [Install fzf using Homebrew](#install-fzf-using-homebrew)
    - [Download `fzf-docker` to your home directory](#download-fzf-docker-to-your-home-directory)
    - [How to set up using key bindings](#how-to-set-up-using-key-bindings)
      - [Zsh](#zsh)
      - [Bash](#bash)
    - [How to setup without key bindings](#how-to-setup-without-key-bindings)
  - [Usage](#usage)
    - [Overview of available commands](#overview-of-available-commands)
    - [Default command for `docker exec in interactive mode`](#default-command-for-docker-exec-in-interactive-mode)
  - [License](#license)

## Overview

This is a plugin that allows you to execute Docker commands using keyboard shortcuts, based on [`MartinRamm/fzf-docker`](https://github.com/MartinRamm/fzf-docker) and utilizing [`fzf`](https://github.com/junegunn/fzf) and [`docker`](https://www.docker.com/).

## Installation

### Install [fzf](https://github.com/junegunn/fzf) using Homebrew

```shell
brew install fzf
```

Please refer to the [fzf official documentation](https://github.com/junegunn/fzf#installation) for installation instructions on other operating systems.

### Download `fzf-docker` to your home directory

```shell
wget -O ~/.fzfdocker https://raw.githubusercontent.com/gumob/fzf-docker/main/fzf-docker.sh
```

### How to set up using key bindings

Source `fzf` and `fzfdocker` in your run command shell.<br/>
By default, no key bindings are set. If you want to set the key binding to `Ctrl+K`, please configure it as follows:

#### Zsh

Set the key binding for fzf-docker and load the script.

```shell
cat <<EOL >> ~/.zshrc
export FZF_DOCKER_KEY_BINDING="^D"
source ~/.fzfdocker
EOL
```

`~/.zshrc` should be like this.

```shell
source <(fzf --zsh)
export FZF_DOCKER_KEY_BINDING='^D'
source ~/.fzfdocker
```

Source run command

```shell
source ~/.zshrc
```

#### Bash

Set the key binding for fzf-docker and load the script.

```shell
cat <<EOL >> ~/.bashrc
export FZF_DOCKER_KEY_BINDING='\C-d'
source ~/.fzfdocker
EOL
```

`~/.bashrc` should be like this.

```shell
eval "$(fzf --bash)"
export FZF_DOCKER_KEY_BINDING='\C-d'
source ~/.fzfdocker
```

Source run command

```shell
source ~/.bashrc
```

### How to setup without key bindings

To run `fzf-docker` without using a keyboard shortcut, remove the line `export FZF_DOCKER_KEY_BINDING='key-combination'` from the run command and enter the following command in the shell.

```shell
fzf-docker
```

## Usage

### Overview of available commands

| description                                                                        | fzf mode | command arguments (optional)                                                                                 |
| ---------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------ |
| docker restart && open logs (in follow mode)                                       | multiple |                                                                                                              |
| docker logs (in follow mode)                                                       | multiple | time interval - e.g.: `1m` for 1 minute - (defaults to all available logs)                                   |
| docker logs (in follow mode) all containers                                        |          | time interval - e.g.: `1m` for 1 minute - (defaults to all available logs)                                   |
| docker exec in interactive mode                                                    | single   | command to exec (default - see below)                                                                        |
| docker remove container (with force)                                               | multiple |                                                                                                              |
| docker remove all containers (with force)                                          |          |                                                                                                              |
| docker stop                                                                        | multiple |                                                                                                              |
| docker stop all running containers                                                 |          |                                                                                                              |
| docker stop and remove container                                                   | multiple |                                                                                                              |
| docker stop and remove all container                                               |          |
| docker kill                                                                        | multiple |                                                                                                              |
| docker kill all containers                                                         |          |                                                                                                              |
| docker kill and remove container                                                   | multiple |                                                                                                              |
| docker kill and remove all container                                               |          |                                                                                                              |
| docker remove image (with force). This includes options to remove dangling images. | multiple |                                                                                                              |
| docker remove all images (with force). This includes dangling images.              |          |                                                                                                              |
| docker stop and remove all & docker remove all images (with force). This includes dangling images.                                                                |          |                                                                                                              |
| docker-compose up (in detached mode)                                               | multiple | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose up all services (in detached mode)                                  |          | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose build (with --no-cache and --pull)                                  | multiple | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose build (with --no-cache and --pull) all                              |          | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose pull                                                                | multiple | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose pull all services                                                   |          | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose update image (rebuild or pull)                                      | multiple | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |
| docker-compose build (with --no-cache and --pull) all && docker-compose pull all services                                                                  |          | path to docker-compose file (defaults to recursive search for `docker-compose.yml` or `docker-compose.yaml`) |

### Default command for `docker exec in interactive mode`

The command used to `exec` into a container is dependent on the base image.
The fallback command used to `exec` into a container is similar to `zsh || bash || ash || sh`.
Useful standards are already implemented for images like `mysql` or `mongo` (PRs to add more default commands are appreciated).

You may however add custom commands that `docker exec in interactive mode` will then use to `exec` into a container. To do this

1. Download the `.fzf-docker-exec.template` to your home directory, omitting the `.template` extension:

```shell
wget -O ~/.fzfdocker-exec https://raw.githubusercontent.com/gumob/fzf-docker/main/fzf-docker-exec.template
```

2. Customize the script as described in the file.

For more in-depth information, please check out the original project's README in the [`Learning by doing`](https://github.com/MartinRamm/fzf-docker?tab=readme-ov-file#learning-by-doing) section.

## License

This project is licensed under the MIT License. The MIT License is a permissive free software license that allows for the reuse, modification, distribution, and sale of the software. It requires that the original copyright notice and license text be included in all copies or substantial portions of the software. The software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement.
