# omp+skills vagrant

A Vagrant setup for running [OMP](https://omp.sh/) with [mattpocock/skills](https://github.com/mattpocock/skills)
inside an isolated libvirt VM.

The host project directory is mounted into the VM, allowing agents to work on the
project in a controlled environment while keeping dependencies and execution
isolated from the host system.

## Requirements

Install the following on the host machine before starting the VM:

1. **Vagrant**
2. **libvirt**
3. **NFS server**

The setup uses libvirt as the VM provider and NFS to mount the project directory
into the guest.

## Usage

### Start the VM

Set `PROJECTS_DIR` to the project directory you want to mount into the VM,
then start Vagrant:

```sh
PROJECTS_DIR=<HOST_PATH_TO_PROJECT> vagrant up --provider=libvirt
```

For example:

```sh
PROJECTS_DIR=$HOME/src/my-project vagrant up --provider=libvirt
```

### Connect to the VM

SSH into the VM:

```sh
vagrant ssh
```

Use X11 forwarding when browser-based login is required, for example when
authenticating with an LLM provider:

```sh
vagrant ssh -Y
```

### Locate your project inside the VM

The project directory is mounted at:

```sh
/home/vagrant/<BASENAME_OF_PROJECTS_DIR>
```

For example, if the host project path is:

```sh
/home/alice/src/my-project
```

then the project will be available inside the VM at:

```sh
/home/vagrant/my-project
```

## Making packages and tools available to omp

omp can use any tools available inside the VM. The recommended approach is to
define project-specific tooling with `direnv` and a Nix flake.

### Option 1: direnv and Nix flake recommended

Using `direnv` with a Nix flake makes project dependencies deterministic and
reproducible, even in a newly provisioned VM.

For example, in a Go project, add a `flake.nix` file:

```nix
{
  description = "Minimal Go project";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux";

      pkgs = import nixpkgs {
        inherit system;
      };
    in
    {
      devShells.${system}.default = pkgs.mkShell {
        packages = with pkgs; [
          go
          gopls
          golangci-lint
        ];
      };
    };
}
```

Search for available Nix packages here:

<https://search.nixos.org/packages?channel=unstable>

Then add a `.envrc` file to the project root:

```sh
use flake
watch_file flake.nix flake.lock
```

Allow the environment:

```sh
direnv allow
```

The configured tools should now be available whenever you enter the project
directory.

Commit the following files to your project:

```text
flake.nix
flake.lock
.envrc
```

Add `.direnv` to `.gitignore`:

```gitignore
.direnv
```

### Option 2: apt-get

The `vagrant` user has passwordless `sudo` configured, so you can install
packages directly inside the VM:

```sh
sudo apt-get update
sudo apt-get install <package>
```

This is useful for quick experiments, but less reproducible than the Nix-based
approach.

## Running omp

After connecting to the VM and entering your mounted project directory, run:

```sh
omp
```

See [here](https://github.com/mattpocock/skills) on the usage of the preinstalled skills.

## Notes

- Use the Nix flake approach for project dependencies that should be reproducible.
- Use `apt-get` for ad-hoc tools or debugging inside the VM.
- Use `vagrant ssh -X` only when X11 forwarding is required.
